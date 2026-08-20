# leetcode-explorer：Rust axum + LLM 在线题解生成器架构

## 问题/背景

想构建一个 LeetCode 在线学习平台，输入题号即可查看中文题目、获取 AI 生成的中文题解（思路解析 + C++/Rust 代码 + 动画步骤），并支持在线编译运行。项目部署在 NAS 上通过 Cloudflare Tunnel 暴露外网。

## 关键操作

### 技术架构

```
用户浏览器 → CF Tunnel → leetcode-explorer (Rust axum :8094)
                                ├── GET  /              → 前端单页应用
                                ├── GET  /api/problems   → 题目列表
                                ├── GET  /api/problem/:slug → 题目详情
                                ├── POST /api/generate   → LLM 生成题解
                                ├── GET  /api/solution/:slug → 缓存题解
                                └── POST /api/compile    → 在线编译运行
```

### 核心模块

**1. leetcode.rs — LeetCode GraphQL 拉取**

通过 LeetCode CN 的 GraphQL API 拉取题目，关键字段：
- `translatedTitle`：中文标题（为空时 fallback 到 `title`）
- `codeSnippets`：多语言代码模板（`langSlug` + `code`）
- `content`：HTML 格式题目描述

```rust
// GraphQL query 关键字段
query {
  question(titleSlug: $slug) {
    questionId
    translatedTitle    // 中文标题
    difficulty
    content            // HTML 描述
    codeSnippets {
      langSlug
      lang
      code
    }
  }
}
```

**2. llm.rs — LLM 题解生成（465行）**

- 从 `~/.hermes/config.yaml` 读取 API key 和 vkey（逐行解析 YAML，不依赖 serde_yaml）
- 调用 cannbot API（OpenAI 兼容格式），model = `glm-5.2`
- 请求头：`Authorization: Bearer {api_key}` + `x-api-vkey: {vkey}`
- 生成 4 项内容：中文解析、C++ 代码、Rust 代码、动画步骤 JSON
- 题解缓存到 `problems/{slug}_solution.json`

```rust
// LLM Prompt 设计
system: "你是一个算法竞赛教练。请为 LeetCode 题目生成题解。"
user: "题目: {title_cn}\n难度: {difficulty}\n描述: {content_plain}
请输出 JSON:
{
  "explanation": "<中文思路讲解，200-500字>",
  "code_cpp": "<C++ 核心代码，只含 Solution 类>",
  "code_rust": "<Rust 核心代码，只含 impl Solution>",
  "animation_steps": "[{type,desc,array,current,compare,found,map}, ...]"
}"
```

动画步骤格式：
- `type`: `init` | `iterate` | `compare` | `found` | `store` | `done`
- `array`: 数字数组
- `current`: 当前索引
- `map`: 哈希表对象
- `desc`: 中文说明

**3. compiler.rs — 编译沙箱**

- 支持 C++（g++）和 Rust（rustc）
- 10 秒超时限制
- 编译后执行，捕获 stdout/stderr

**4. main.rs — axum 路由**

```rust
fn main() {
    let app = Router::new()
        .route("/", get(index))
        .route("/api/problems", get(list_problems))
        .route("/api/problem/:slug", get(get_problem))
        .route("/api/problem/id/:id", get(get_problem_by_id))
        .route("/api/generate", post(generate_solution_handler))
        .route("/api/solution/:slug", get(get_solution))
        .route("/api/compile", post(compile_code));
}
```

### 子代理开发流程

用 `delegate_task` 分派子代理开发 llm.rs 模块（566 秒完成）：
- 子代理自动创建 llm.rs（465 行）
- 修改 models.rs 添加 `SolutionResult` 和 `GenerateRequest` 结构体
- 修改 main.rs 添加 `/api/generate` 和 `/api/solution/:slug` 路由
- 子代理内部处理了 YAML 解析、JSON 提取、缓存写入

### 部署配置

```ini
# /etc/systemd/system/leetcode-explorer.service
[Unit]
Description=LeetCode Explorer
After=network.target

[Service]
Type=simple
Environment=HOME=/home/heron          # ← 关键！见下文坑
WorkingDirectory=/home/heron/workspace/leetcode-explorer
ExecStart=/home/heron/workspace/leetcode-explorer/target/release/leetcode-explorer
Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target
```

通过 CF Tunnel 暴露：`leetcode.heronwang.cn → localhost:8094`

### 测试结果

```
✅ LLM 题解生成成功!
  解析: 这道题要求在数组中找到两个数...哈希表优化到 O(n)...
  C++: 407 字符
  Rust: 429 字符
  动画: 785 字符, 8 个步骤
```

作品集 ID = 10，访问地址：`https://leetcode.heronwang.cn/?v=4`

## 经验/坑

### 1. systemd 缺少 HOME 环境变量

LLM 模块需要读取 `~/.hermes/config.yaml` 获取 API key，但 systemd 服务的默认 HOME 不是 `/home/heron`，导致：

```
错误: 读取配置文件 ~/.hermes/config.yaml 失败: No such file or directory
```

**修复**：在 `.service` 文件中添加 `Environment=HOME=/home/heron`

### 2. Cloudflare 缓存旧页面

CF 缓存了 293 字节的占位符页面（`cf-cache-status: HIT`, `age: 5568`），而本地实际返回 21398 字节。

**修复**：在 axum 的 index handler 中添加 no-cache 响应头：

```rust
async fn index() -> Result<Response, AppError> {
    let html = std::fs::read_to_string("web/index.html")?;
    Ok((
        StatusCode::OK,
        [
            ("Content-Type", "text/html; charset=utf-8"),
            ("Cache-Control", "no-cache, no-store, must-revalidate"),
            ("Pragma", "no-cache"),
            ("Expires", "0"),
        ],
        html,
    ).into_response())
}
```

并用 URL 参数 `?v=4` 绕过 CF 缓存（不同 URL 参数 → CF 视为不同资源 → `cf-cache-status: BYPASS`）

### 3. 子代理覆盖问题

子代理在修改 main.rs 时，不小心删除了已有的 `cache_solution` 调用。需要重新检查子代理的修改，确保 `generate_solution` 内部调用了 `write_solution_cache`。

### 4. Rust edition 配置

lint 检查报 `async fn is not permitted in Rust 2015`，但 `cargo build --release` 编译成功。这是因为 lint 工具检查单文件时不知道 Cargo.toml 中的 `edition = "2021"` 设置，属于误报。
