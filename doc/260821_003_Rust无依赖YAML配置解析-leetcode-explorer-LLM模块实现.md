# Rust 无依赖 YAML 配置解析：leetcode-explorer LLM 模块实现

> **日期**: 2026-08-21  
> **分类**: 项目经验  
> **标签**: Rust, YAML解析, LLM, leetcode-explorer, 子代理协作  
> **来源**: hermes

---

## 背景/问题

leetcode-explorer 项目需要调用 LLM（glm-5.2）生成中文题解。LLM 的 API Key 和 vkey 存储在 `~/.hermes/config.yaml` 中。需要在 Rust 中读取 YAML 配置文件，但不想引入 `serde_yaml` 等额外依赖。

## 解决方案：纯 Rust 文本解析 YAML

使用逐行扫描 + 字符串匹配方式提取 YAML 中的 key-value 对，避免了引入 `serde_yaml` 依赖。

### 核心实现

```rust
/// 从 YAML 文本中提取 `key: value` 的 value 部分（去引号、去注释）
fn extract_yaml_value(content: &str, key: &str) -> Option<String> {
    for line in content.lines() {
        let trimmed = line.trim_start();
        if let Some(rest) = trimmed
            .strip_prefix(key)
            .and_then(|s| s.strip_prefix(':').or_else(|| s.strip_prefix(" :")))
        {
            let mut val = rest.trim();
            // 去掉行尾注释（# 后面），但要注意值里可能有 #
            if let Some(pos) = find_comment_pos(val) {
                val = val[..pos].trim();
            }
            // 去引号
            val = unquote(val);
            if !val.is_empty() {
                return Some(val.to_string());
            }
        }
    }
    None
}
```

### 注释位置检测

YAML 中 `#` 前面有空格才算是注释标记，值内部的 `#` 不算：

```rust
fn find_comment_pos(s: &str) -> Option<usize> {
    let bytes = s.as_bytes();
    let mut in_squote = false;
    let mut in_dquote = false;
    for i in 0..bytes.len() {
        match bytes[i] {
            b'\'' if !in_dquote => in_squote = !in_squote,
            b'"' if !in_squote => in_dquote = !in_dquote,
            b'#' if !in_squote && !in_dquote => {
                if i == 0 || bytes[i - 1].is_ascii_whitespace() {
                    return Some(i);
                }
            }
            _ => {}
        }
    }
    None
}
```

## LLM 调用流程

```
1. get_llm_config() → 读取 ~/.hermes/config.yaml
   → 提取 api_key（288字符 JWT）和 x-api-vkey（vk-xxx 格式）

2. build_prompt(problem) → 构造 system + user prompt
   → HTML 题目描述用 html_to_text() 转纯文本
   → 截断到 3000 字符避免 token 超限

3. POST 到 cannbot API
   → Authorization: Bearer {api_key}
   → x-api-vkey: {vkey}
   → Body: {"model":"glm-5.2","messages":[...],"temperature":0.3}

4. extract_json(content) → 从 LLM 返回中提取 JSON
   → 处理 markdown 代码块标记 (```json ... ```)
   → 平衡括号匹配提取 {...}
```

## LLM Prompt 设计

```
你是一个算法竞赛教练。请为以下 LeetCode 题目生成题解:

题目: {title_cn}
难度: {difficulty}
描述: {纯文本描述}

请输出 JSON:
{
  "explanation": "<中文思路讲解，200-500字>",
  "code_cpp": "<C++ 核心代码，只含 Solution 类的关键函数>",
  "code_rust": "<Rust 核心代码，只含 impl Solution>",
  "animation_steps": "[{type,desc,array,current,compare,found,map}, ...]"
}

animation_steps type: "init" | "iterate" | "compare" | "found" | "store" | "done"
```

## 子代理协作踩坑

使用 `delegate_task` 分派子代理实现 LLM 模块时，父代理和子代理同时修改 `main.rs`，导致函数名不一致：

| 问题 | 原因 | 修复 |
|------|------|------|
| 函数名不匹配 | 子代理用了 `read_cached_solution`/`cache_solution`，实际定义是 `read_solution_cache`/`write_solution_cache` | 统一为 `llm.rs` 中的实际函数名 |
| 重复缓存写入 | `main.rs` handler 和 `generate_solution` 内部都写缓存 | 删除 handler 中的重复调用 |
| 文件写入冲突 | sibling subagent 修改了父代理已读的文件 | patch 前先 re-read 文件 |

## systemd HOME 环境变量问题

服务部署后 LLM 调用失败：`读取配置文件 ~/.hermes/config.yaml 失败: No such file or directory`。

**根因**：systemd 服务的 HOME 默认不是 `/home/heron`。

**修复**：在 `.service` 文件中添加 `Environment=HOME=/home/heron`：

```ini
[Service]
Type=simple
Environment=HOME=/home/heron
WorkingDirectory=/home/heron/workspace/leetcode-explorer
ExecStart=/home/heron/workspace/leetcode-explorer/target/release/leetcode-explorer
Restart=always
RestartSec=3
```

## 验证结果

```bash
# 测试 LLM 生成题解
curl -s -X POST http://localhost:8094/api/generate \
  -H "Content-Type: application/json" \
  -d '{"slug":"two-sum"}'

# 返回:
# ✅ 题解生成成功!
#   解析: 这道题要求在数组中找到两个数...哈希表优化到 O(n)...
#   C++: 407 字符
#   Rust: 429 字符
#   动画: 785 字符, 8 个步骤
```

## 经验总结

1. **简单 YAML 不需要 serde_yaml**：逐行字符串匹配足以处理 key-value 格式的配置文件，减少依赖
2. **子代理协作要注意文件冲突**：多个代理同时修改同一文件时，patch 前必须 re-read
3. **systemd HOME 环境变量**：所有需要读取 `~` 目录文件的服务都要设置 `Environment=HOME=...`
4. **LLM JSON 提取需健壮**：LLM 可能返回 markdown 代码块包裹的 JSON，或纯 JSON，需要两种处理方式
5. **动画步骤存储为字符串**：LLM 可能返回数组或字符串，统一序列化为字符串供前端 `JSON.parse`
