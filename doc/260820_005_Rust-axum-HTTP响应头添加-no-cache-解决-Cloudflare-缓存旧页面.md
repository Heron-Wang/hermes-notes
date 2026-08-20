# Rust axum HTTP 响应头添加 no-cache 解决 Cloudflare 缓存旧页面

## 问题/背景

leetcode-explorer 项目部署后，用户外网访问 `leetcode.heronwang.cn` 看到的是旧的 293 字节占位符页面，而非本地实际返回的 21398 字节完整页面。检查发现 Cloudflare 缓存了旧页面（`cf-cache-status: HIT`, `age: 5568` = 缓存了 1.5 小时）。

## 关键操作

### 诊断

```bash
# 检查 CF 缓存状态
curl -sI https://leetcode.heronwang.cn/ | grep -i "cf-cache\|cache-control\|age"
# 输出:
# cf-cache-status: HIT
# age: 5568
# cache-control: max-age=14400    ← CF 默认缓存 4 小时

# 对比本地和外网返回大小
curl -s https://leetcode.heronwang.cn/ | wc -c    # 293 bytes (外网, 旧缓存)
curl -s http://localhost:8094/ | wc -c            # 21398 bytes (本地, 实际)
```

### 修复

在 axum 的 index handler 中添加 no-cache 响应头：

```rust
// 修改前 — 只有 Content-Type
async fn index() -> Result<Response, AppError> {
    let html = std::fs::read_to_string("web/index.html")?;
    Ok((
        StatusCode::OK,
        [("Content-Type", "text/html; charset=utf-8")],
        html,
    ).into_response())
}

// 修改后 — 添加完整的 no-cache 头
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

### 编译重启

```bash
cd /home/heron/workspace/leetcode-explorer
cargo build --release

# sudo 重启（cron 环境用 SUDO_ASKPASS 脚本）
sudo systemctl restart leetcode-explorer
```

### 验证

```bash
# 本地验证 no-cache 头
curl -sI http://localhost:8094/ | grep -i "cache-control"
# cache-control: no-cache, no-store, must-revalidate ✅

# 外网用 URL 参数绕过现有缓存
curl -s https://leetcode.heronwang.cn/?v=4 | wc -c
# 21398 bytes ✅

# 检查 CF 缓存状态
curl -sI https://leetcode.heronwang.cn/?v=4 | grep -i "cf-cache"
# cf-cache-status: BYPASS ✅
```

## 经验/坑

### Cloudflare 缓存机制

| CF 响应头 | 含义 |
|----------|------|
| `cf-cache-status: HIT` | 从 CF 缓存返回（旧内容） |
| `cf-cache-status: MISS` | 未命中缓存，回源获取 |
| `cf-cache-status: BYPASS` | 绕过缓存（URL 参数不同或有 no-cache 头） |
| `cf-cache-status: EXPIRED` | 缓存已过期，回源获取 |
| `age: 5568` | 缓存年龄（秒），5568s ≈ 1.5 小时 |

### 三种绕过 CF 缓存的方法

| 方法 | 原理 | 适用场景 |
|------|------|----------|
| **no-cache 响应头** | CF 尊重源站的 Cache-Control 头 | 永久解决，推荐 |
| **URL 参数 `?v=N`** | 不同 URL → CF 视为不同资源 → BYPASS | 临时绕过，快速验证 |
| **CF API 清缓存** | 调用 CF API 删除缓存 | 需要 CF API Key，一次性操作 |

### axum 响应头注意事项

axum 的 `into_response()` 支持多种响应头格式：
- `[(&str, &str); N]` — 固定长度数组
- `Vec<(&str, &str)>` — 动态长度
- `HeaderMap` — 完整控制

对于静态 no-cache 头，用固定数组最简洁。

### 微信浏览器特殊缓存

微信内置浏览器（X5 内核）有更激进的缓存策略，即使加了 no-cache 头也可能缓存旧页面。需要同时配合：
- HTML `<meta http-equiv="Cache-Control" content="no-cache">`
- URL 版本号 `?v=N`
- 这在之前的笔记中有详细记录
