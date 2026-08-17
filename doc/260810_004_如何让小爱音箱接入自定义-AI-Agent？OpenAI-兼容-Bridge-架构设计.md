# 如何让小爱音箱接入自定义 AI Agent？OpenAI 兼容 Bridge 架构设计

> **日期**: 2026-08-10  
> **分类**: 经验总结  
> **标签**: xiaogpt, 小爱音箱, OpenAI兼容, Bridge, Hermes, 架构设计  
> **来源**: hermes

---

## 背景/问题

想把家里的小爱音箱接入自己搭建的 Hermes Agent（运行在 NAS 上），让小爱不仅能聊天，还能执行复杂任务（查时间、控制设备、搜索信息等）。

但面临几个问题：
1. 小爱音箱本身只支持小米自家的 AI，不能自定义后端
2. `xiaogpt` 工具可以劫持小爱的对话，但它只支持标准 OpenAI API 格式
3. Hermes Agent 不是标准 HTTP API，而是 CLI 工具
4. 即使有 OpenAI 兼容的 LLM API（如 CANNBOT），也没有 Agent 能力（工具调用、记忆、技能）

## 原因分析

小爱音箱的对话流程：

```
用户说话 → 小爱云端识别 → 小爱生成回答 → TTS 播报
```

`xiaogpt` 的原理是：监听小爱的对话记录，拦截特定关键词开头的提问，把问题转发给外部 LLM，然后用 TTS 播报 LLM 的回答。

`xiaogpt` 支持 `bot: chatgptapi` 模式，可以通过 `api_base` 指向任何 OpenAI 兼容 API。但它期望的是一个标准的 `/v1/chat/completions` 端点。

而 Hermes Agent 是一个 CLI 工具（`hermes chat -q "问题"`），不是 HTTP 服务。

## 解决方案：自建 Bridge Server

在中间加一层 Bridge HTTP 服务器，同时实现两种路由：

```
小爱音箱 → xiaogpt → Bridge Server ─┬─→ CANNBOT API 直连（快速聊天）
                                     └─→ hermes chat -q（完整 Agent）
```

### Bridge 核心设计

```python
# 路由逻辑：关键词触发决定走哪条路径
TRIGGER_KEYWORDS = ["帮", "查", "执行", "运行", "检查", "搜索", 
                     "写", "改", "创建", "删除", "部署", "看看"]

def process_query(messages, stream=False):
    user_msg = extract_last_user_message(messages)
    
    if should_use_hermes(user_msg):  # 关键词触发
        # 慢路径：调用 hermes CLI，获得完整 Agent 能力
        content = hermes_chat(user_msg)  # subprocess.run(["hermes", "chat", "-q", ...])
        return content, "hermes-agent"
    else:
        # 快路径：直接调 LLM API，2-3秒响应
        content = cannbot_chat(messages)
        return content, "cannbot-fast"
```

### Bridge 暴露的 OpenAI 兼容端点

| 端点 | 方法 | 作用 |
|------|------|------|
| `/v1/chat/completions` | POST | 接收 xiaogpt 的聊天请求 |
| `/v1/models` | GET | 返回可用模型列表 |
| `/health` | GET | 健康检查 |

### xiaogpt 配置

```yaml
# xiaogpt 的 config.yaml
bot: chatgptapi           # 使用 OpenAI 兼容模式
api_base: "http://127.0.0.1:8090/v1"  # 指向 Bridge
openai_key: "dummy"       # Bridge 不验证 key，随便填
hardware: LX06            # 小爱音箱型号
mute_xiaoai: true         # 静音小爱原生回答
stream: true              # 流式响应更快
tts: edge                 # 用 Edge TTS 获得更好音质
```

### 两种路径对比

| | 快路径（直连 LLM） | 慢路径（Hermes Agent） |
|---|---|---|
| 响应时间 | 2-3秒 | 5-30秒 |
| 能力 | 纯文本聊天 | 工具调用、技能、记忆 |
| 触发方式 | 默认 | 说"帮/查/执行..."等关键词 |
| 适用场景 | 闲聊、简单问答 | "帮我看看几点了"、"查下服务器状态" |

## 避坑提示

- Bridge 用 Python 标准库 `http.server` 实现即可，不需要 Flask/FastAPI，减少依赖
- SSE 流式响应需要手动拼接 `data: {json}\n\n` 格式，注意 `Content-Type: text/event-stream`
- `xiaogpt` 的 `api_base` 要以 `/v1` 结尾，不要带末尾斜杠
- 小爱音箱获取 DID：`pip install miservice_fork` → 设置 `MI_USER`/`MI_PASS` 环境变量 → `micli list`
- 如果 `micli list` 报错，尝试换网络（小米登录接口对 IP 有时有限制）
- Bridge 和 xiaogpt 建议用 systemd service 管理，开机自启
