# Hermes CLI -Q 安静模式为什么还会输出 reasoning？

> **日期**: 2026-08-10  
> **分类**: 踩坑记录  
> **标签**: Hermes, CLI, reasoning, subprocess  
> **来源**: hermes

---

## 背景/问题

在通过 subprocess 调用 `hermes chat -q "问题" -Q`（-Q 是 quiet 模式）获取 Hermes Agent 的回复时，发现 stdout 里混入了模型的推理过程（reasoning）文本：

```
┌─ Reasoning ────────────────────────────────┐
The user is asking what time it is. Let me check...
The user asked what time it is. Simple answer.
现在是 2026年8月10日 星期一 23:16:39 (CST)。
```

期望只拿到最后一行（实际回答），但前面的英文推理过程也混进来了。

## 原因分析

Hermes Agent 的 `-Q`（quiet）模式设计初衷是"抑制 banner、spinner、tool preview"，让输出适合程序化解析。但 reasoning（思维链）输出是模型回复的一部分，被当作正常内容输出到 stdout。

尝试了 `--reasoning none` 参数，但发现：

1. reasoning box 的开始标记 `┌─ Reasoning ─...─┐` 存在
2. 但**没有闭合标记** `└─...─┘`（box 没有正确闭合）
3. 所以用正则 `re.sub(r"┌─.*?└─", "", text, re.DOTALL)` 匹配不到

## 解决方案

在调用方（Bridge 层）逐行解析过滤：

```python
output = result.stdout.strip()
lines = output.split("\n")
cleaned = []
in_reasoning = False

for line in lines:
    stripped = line.strip()
    
    # 检测 reasoning box 开始
    if stripped.startswith("┌") and "Reasoning" in stripped:
        in_reasoning = True
        continue
    
    # 检测 reasoning box 结束（如果有）
    if stripped.startswith("└"):
        in_reasoning = False
        continue
    
    if in_reasoning:
        # reasoning 区域内的行通常是英文
        # 如果遇到中文字符，说明实际回答开始了
        if stripped and any("\u4e00" <= c <= "\u9fff" for c in stripped):
            in_reasoning = False
            cleaned.append(line)
        # 否则跳过英文 reasoning 行
        continue
    
    # 跳过独立的 box-drawing 字符行
    if stripped and all(c in "┌┐└┘│─" for c in stripped):
        continue
    
    cleaned.append(line)

output = "\n".join(cleaned).strip()
```

**核心思路**：reasoning 通常是英文，实际回答是中文。利用 Unicode 范围检测（`\u4e00`-`\u9fff` 是 CJK 统一汉字）来区分。

## 避坑提示

- 这个方案假设 reasoning 是英文、回答是中文。如果你的场景是全英文对话，需要换一种判断逻辑（比如基于关键词 "Let me"、"The user" 等模式匹配）
- `hermes chat -q` 的 stderr 会输出 `session_id: xxx`，不要把它当成错误
- 调用 hermes CLI 时要用完整路径（如 `/home/user/.local/bin/hermes`），因为 subprocess 的 PATH 可能不包含用户 bin 目录
- 如果不需要 Agent 能力（工具/技能/记忆），直接调用 LLM API 比走 hermes CLI 快 10 倍
