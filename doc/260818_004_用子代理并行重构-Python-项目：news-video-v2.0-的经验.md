# 用子代理并行重构 Python 项目：news-video v2.0 的经验

> **日期**: 2026-08-18  
> **分类**: 架构设计  
> **标签**: Hermes, 子代理, delegate_task, 并行, news-video, Python, 架构重构  
> **来源**: hermes

---

## 背景/问题

news-video 是一个每日科技新闻视频自动生成系统（采集新闻 → LLM 处理 → 生成幻灯片 → 截图渲染 → TTS 配音 → ffmpeg 合成 → B站上传）。v1 是单文件 God file（main.py 800+ 行），v2 需要拆分为模块化架构。

重构涉及 3 个独立模块：
1. `pipeline.py` — 线程安全的视频生成流水线运行器
2. `bili_uploader.py` — B站视频上传模块
3. `web_server.py` — Web UI + POST API + B站发布管理

三个模块之间有依赖关系（web_server 依赖 pipeline 和 bili_uploader），但各自可以独立编写。串行写需要 ~25 分钟，能否并行？

## 解决方案

### 用 delegate_task 并行派发 3 个子代理

```
delegate_task(tasks=[
    { goal: "创建 pipeline.py",     context: "..." },
    { goal: "创建 bili_uploader.py", context: "..." },
    { goal: "重写 web_server.py",   context: "..." },
])
```

三个子代理各自有独立的终端会话和上下文，同时开始工作。父代理在等待期间可以做其他集成工作。

### 子代理的验证规范

每个子代理完成后需要自验：
- `py_compile` 语法检查通过
- ad-hoc 验证脚本覆盖核心 API（函数签名、返回值、边界情况）
- 验证脚本和临时文件清理
- 报告中明确标注"ad-hoc 验证"而非"项目测试套件"

### 关键时间线

| 阶段 | 耗时 | 内容 |
|------|------|------|
| 派发 3 个子代理 | ~0s | 并行启动 |
| 等待期间 | ~14min | 父代理做集成工作（config.py、index.html 修改等） |
| 子代理 1 完成 | 207s | pipeline.py（532 行） |
| 子代理 2 完成 | 151s | bili_uploader.py（~300 行） |
| 子代理 3 完成 | 845s | web_server.py（最复杂，需集成前两个的接口） |
| 总耗时 | ~14min | 并行 vs 串行 ~25min |

## 经验总结

### 子代理适合的场景

| 场景 | 适合？ | 原因 |
|------|--------|------|
| **独立模块编写** | 适合 | 模块间无编译时依赖，可并行 |
| **接口明确的集成** | 适合 | 给定 API 契约后各自实现 |
| **有交叉依赖的重构** | 需规划 | 先做被依赖的模块，再做依赖方 |
| **需要多次迭代的调试** | 不适合 | 子代理不能交互式调试 |
| **需要用户确认的决策** | 不适合 | 子代理不能问用户 |

### context 传递的关键信息

子代理不知道父会话的任何内容，所有信息必须通过 context 传递：

```python
{
    "goal": "创建 pipeline.py",
    "context": """
项目路径: /home/heron/workspace/news-video/
Python 虚拟环境: .venv/
LLM API: URL 从 .env 的 LLM_API_URL 读取，header 用 Authorization: Bearer {key} + x-api-vkey: {vkey}
config.py 已存在，包含 CATEGORIES, TTS_VOICE, TTS_RATE 等常量
video_composer.compose() 接受参数: slide_images, audio_files, intro_img, outro_img
输出目录: output/video/, output/archive/
状态文件: pipeline_status.json
线程安全: 用 threading.RLock()
"""
}
```

### 子代理报告的可信度

子代理的报告是**自报告**，不是已验证的事实。子代理说"验证通过"可能：
- 验证脚本本身有 bug（用 `set -e` 导致 grep 无匹配时退出）
- 只验证了 happy path，没测边界情况
- 说"文件已写入"但实际写入路径错误

对于外部副作用（文件写入、服务启动），父代理应独立验证：
```bash
# 父代理验证子代理的产出
py_compile pipeline.py && echo "syntax ok"
grep -c 'def run_pipeline' pipeline.py  # 确认关键函数存在
wc -l pipeline.py  # 确认文件大小合理
```

## 避坑提示

- **子代理会遗留测试服务器**：子代理可能在测试时启动了 HTTP 服务器（如 `python web_server.py`），完成后没清理。父代理需要 `kill` 残留进程
- **`set -e` 在验证脚本中的陷阱**：`grep -c` 无匹配返回 1，`set -e` 会当作错误退出。用 `grep ... || true` 或 `COUNT=$(grep -c ... || echo 0)`
- **子代理的 venv 路径**：子代理可能不知道项目的 venv 路径，在 context 中明确指定 `.venv/bin/python` 或 `.venv/bin/biliup`
- **并行写同一文件会冲突**：确保子代理的文件输出路径不重叠。本例中 pipeline.py、bili_uploader.py、web_server.py 各自独立文件，无冲突
- **子代理不能调用 delegate_task**：默认 leaf 角色，不能进一步派发。如果需要多级编排，用 orchestrator 角色（需配置 max_spawn_depth）

## 相关知识

### delegate_task vs 串行执行

| 维度 | delegate_task | 串行执行 |
|------|-------------|---------|
| 速度 | 并行，最快 | 串行，慢 |
| 上下文隔离 | 完全隔离 | 共享父会话上下文 |
| 交互能力 | 不能问用户 | 可以 |
| 适合任务 | 独立、明确 | 需要迭代、交互 |
| 验证成本 | 父代理需独立验证 | 边做边验证 |
| 资源消耗 | 3 个并发 LLM 会话 | 1 个会话 |

### news-video v2.0 的最终架构

```
news-video/
├── config.py           # 配置常量（分类、TTS、路径）
├── pipeline.py         # 流水线运行器（线程安全）
├── llm_processor.py    # LLM 新闻处理
├── slide_generator.py  # 幻灯片 HTML 生成
├── screenshot.py       # 截图渲染
├── video_composer.py   # ffmpeg 视频合成
├── bili_uploader.py    # B站上传
├── web_server.py       # Web UI + API
└── web/index.html      # 前端页面
```

每个文件 ≤ 300 行（Python），遵循 route → service → store 分层。
