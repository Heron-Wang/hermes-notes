# Python 包安装卡在大依赖下载？试试 --no-deps 分步安装

> **日期**: 2026-08-10  
> **分类**: 踩坑记录  
> **标签**: Python, pip, xiaogpt, 依赖管理  
> **来源**: hermes

---

## 背景/问题

安装 `xiaogpt`（小爱音箱 LLM 对接工具，v3.23）时，`uv pip install xiaogpt` 一直卡住不动。

查看日志发现卡在一个 91MB 的包 `lingua-language-detector` 上，网络速度只有几十 KB/s，预估要下载 30 分钟以上。

## 原因分析

`xiaogpt` 声明了大量依赖（111 个包！），其中包括：

- `langchain` — 引入一大堆子依赖
- `lingua-language-detector` — **91MB**，用于语言检测
- `google-generativeai` — Google AI SDK
- `volcengine-python-sdk` — 39MB 火山引擎 SDK
- `dashscope` — 阿里通义 SDK
- `azure-cognitiveservices-speech` — Azure 语音 SDK

但实际使用中，如果你只用 `bot: chatgptapi` 模式（OpenAI 兼容 API），这些 SDK **根本不会被调用**。它们只是因为 `xiaogpt` 支持多种 AI 后端才被声明为依赖。

## 解决方案

分两步安装：先 `--no-deps` 装主包，再手动装真正需要的核心依赖：

```bash
# 1. 先装 xiaogpt 本身，跳过所有依赖
venv/bin/pip install xiaogpt --no-deps

# 2. 只装 chatgptapi 模式需要的核心依赖
venv/bin/pip install \
  miservice_fork \        # 小米设备控制
  "openai>=1" \            # OpenAI API 客户端
  aiohttp \                # 异步 HTTP
  rich \                   # 终端美化输出
  "httpx[socks]" \         # HTTP 客户端（含代理支持）
  "pyyaml>=6.0.1" \        # 配置文件解析
  "beautifulsoup4>=4.12.0" \  # HTML 解析
  "numexpr>=2.8.6" \       # 数学表达式计算
  "zhipuai>=2.0.1" \       # 智谱 AI（可选）
  "tetos>=0.2.1"            # TTS 引擎抽象层
```

这样跳过了 91MB 的 `lingua-language-detector` 和其他不用的 SDK，安装时间从 30+ 分钟降到 3 分钟。

## 避坑提示

- `--no-deps` 后如果缺少运行时依赖，程序会在 import 时报 `ModuleNotFoundError`，根据报错补装即可
- 如果用 Docker 部署 xiaogpt，官方镜像 `yihong0618/xiaogpt` 已包含全部依赖，不用折腾
- `uv pip install` 也支持 `--no-deps`，但如果网络不稳定时 pip 的重试机制更好用
- 安装前可以先 `pip download xiaogpt --no-deps -d /tmp/pkg` 看看主包多大（xiaogpt 本身只有 40KB）
