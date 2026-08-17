# 用 Rust 重构贪吃蛇游戏服务：从 Python 到纯 std HTTP Server

> **日期**: 2026-08-07  
> **分类**: 经验总结  
> **标签**: 无  
> **来源**: hermes

---

## 背景：为什么要用 Rust 重构？

之前贪吃蛇游戏用 Python `http.server` 实现，功能没问题但有几个可以改进的点：
- Python 进程占用内存约 13MB
- 每次请求都要经过 Python 解释器
- 想验证 Rust 纯标准库写 Web 服务的可行性

## 技术选型

| 对比项 | Python 旧版 | Rust 新版 |
|--------|-----------|----------|
| HTTP 服务 | http.server (stdlib) | std::net::TcpListener (手写) |
| 依赖 | 零依赖 | 零依赖（纯 std） |
| 二进制 | 无（解释执行） | 611KB 单文件 |
| 内存占用 | ~13MB | ~300KB |
| 并发 | 单线程 | 线程池 |
| 编译 | 无需编译 | cargo build --release |

## 目录结构

```
snake-rs/
├── Cargo.toml          # Rust 项目配置（零依赖）
├── Cargo.lock          # 依赖锁定（无第三方依赖）
├── .gitignore          # 忽略 target/
├── snake-rs.service    # systemd 服务文件
├── README.md           # 项目说明
└── src/
    ├── main.rs         # Rust HTTP 服务（7.9KB）
    └── index.html      # 贪吃蛇游戏页面（13KB）
```

## 架构设计

```
浏览器请求 → Cloudflare Tunnel → Rust HTTP Server (:8081)
                                    │
                               TcpListener
                                    │
                              ┌─────┴─────┐
                              │  线程池   │
                              └─────┬─────┘
                                    │
                              路由匹配
                              ├── GET /        → include_str!("index.html")
                              ├── GET /health  → JSON {"status":"ok"}
                              └── 其他         → 404
```

### 核心设计决策

1. **include_str! 宏**：HTML 文件单独放在 `src/index.html`，编译时通过 `include_str!("index.html")` 嵌入二进制。好处：修改 HTML 不用改 Rust 代码，且部署时只需要一个二进制文件。

2. **手写 HTTP 解析**：不依赖任何 HTTP 框架，直接用 `std::net::TcpListener` 接受连接，读取请求行，解析路径。对于这种只返回静态内容的场景足够了。

3. **线程池并发**：用 `std::thread::spawn` 为每个连接创建线程。虽然不是真正的线程池（没有复用），但对于这种轻量服务完全够用。

## 执行流程

### 场景1：用户访问游戏页面

```
1. 浏览器请求 https://snake.heronwang.cn/
2. Cloudflare CDN → Tunnel → 本地 :8081
3. Rust TcpListener.accept() 接受连接
4. 读取 HTTP 请求行: "GET / HTTP/1.1"
5. 路由匹配: path == "/"
6. 返回 include_str!("index.html") 内容
   - Content-Type: text/html; charset=utf-8
   - Content-Length: 13000
7. 浏览器渲染贪吃蛇游戏
```

### 场景2：健康检查

```
1. curl https://snake.heronwang.cn/health
2. Rust 解析路径: "/health"
3. 返回: {"status":"ok"}
   - Content-Type: application/json
   - 14 bytes
```

### 场景3：404 处理

```
1. 请求任意不存在的路径
2. Rust 返回: "404 Not Found"
   - Content-Type: text/plain
   - 13 bytes
```

### 场景4：编译和部署

```
# 编译
cd /home/heron/projects/snake-rs
cargo build --release
# → 生成 target/release/snake-rs (611KB)

# 部署为 systemd 服务
sudo cp snake-rs.service /etc/systemd/system/
sudo systemctl enable --now snake-rs
# → 开机自启 + 崩溃自动重启
```

## Cargo.toml 配置

```toml
[package]
name = "snake-rs"
version = "0.1.0"
edition = "2021"

[[bin]]
name = "snake-rs"
path = "src/main.rs"

[profile.release]
opt-level = 2
```

> **关键点：** 没有任何 `[dependencies]`，纯标准库实现。`opt-level = 2` 开启优化。

## main.rs 核心结构

```rust
use std::io::{Read, Write};
use std::net::{TcpListener, TcpStream};
use std::thread;

// HTML 内容编译时嵌入
const INDEX_HTML: &str = include_str!("index.html");

fn main() {
    let listener = TcpListener::bind("0.0.0.0:8081").unwrap();
    for stream in listener.incoming() {
        let stream = stream.unwrap();
        thread::spawn(|| handle_connection(stream));
    }
}

fn handle_connection(mut stream: TcpStream) {
    // 读取请求 → 解析路径 → 路由匹配 → 返回响应
}
```

> **include_str! 的优势：** HTML 文件在编译时被嵌入二进制，运行时不需要读文件系统，部署只需要一个可执行文件。

## 性能对比

| 指标 | Python 版 | Rust 版 |
|------|----------|---------|
| 二进制大小 | 无 | 611 KB |
| 进程内存 | ~13 MB | ~300 KB |
| 响应延迟 | ~0.7ms | ~0.1ms |
| 冷启动 | 即时 | 即时 |
| 部署方式 | python3 server.py | ./snake-rs |

## 避坑提示

1. **include_str! 路径相对于源文件**：`include_str!("index.html")` 中的路径是相对于 `main.rs` 所在目录的，不是项目根目录
2. **HTTP 响应头要完整**：Content-Type 和 Content-Length 缺一不可，否则浏览器可能无法正确渲染
3. **线程不是线程池**：`thread::spawn` 每次创建新线程，高并发场景应改用线程池。但贪吃蛇这种场景完全够用
4. **Cargo.toml 的 [[bin]] 段**：如果包名和二进制名不同，需要显式指定 `[[bin]] name` 和 `path`
5. **release 编译**：开发用 `cargo run`，部署用 `cargo build --release`，优化后性能差距明显
