# Cloudflare Tunnel + Python http.server 浏览器缓存问题：3 种绕过方案

> **日期**: 2026-08-18  
> **分类**: 踩坑记录  
> **标签**: Cloudflare, 缓存, Python, http.server, HTML, 微信浏览器  
> **来源**: hermes

---

## 背景/问题

news-video 项目的 Web UI 部署在 NAS 上，通过 Cloudflare Tunnel 暴露到外网（news.heronwang.cn）。后端是 Python `http.server`，更新前端 HTML 后用户反馈"看到的还是旧页面"。

问题出在三层缓存叠加：
1. Cloudflare 边缘缓存（CDN 层）
2. 浏览器缓存（Chrome/Edge/微信内置浏览器）
3. Python http.server 不发 Cache-Control 头

## 原因分析

### Python http.server 的缓存行为

Python `http.server` 是简单的文件服务器，不设置任何 Cache-Control 响应头。浏览器看到没有缓存指令时，会使用自己的启发式缓存（heuristic caching）——通常缓存文件 10% 的 Last-Modified age。

### Cloudflare Tunnel 的缓存行为

CF Tunnel 本身不缓存，但 CF 边缘节点会根据 Page Rules 和响应头决定是否缓存。Free 版最多 3 条 Page Rules，已经用了通配符规则覆盖全部子域名。

### 微信浏览器的特殊行为

微信内置浏览器（X5 内核）缓存比标准 Chrome 更激进：
- 不遵守部分 meta 标签
- 清除缓存需要手动操作（设置 → 通用 → 存储空间 → 清理缓存）
- URL 加 `?v=xxx` 参数有时无效（X5 可能忽略 query string）

## 解决方案

### 方案 1：HTML meta 标签禁用缓存

```html
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width,initial-scale=1.0,maximum-scale=5.0">
<meta http-equiv="Cache-Control" content="no-cache, no-store, must-revalidate">
<meta http-equiv="Pragma" content="no-cache">
<meta http-equiv="Expires" content="0">
<title>页面标题</title>
```

优点：零依赖，HTML 层面解决。缺点：微信 X5 内核可能不遵守。

### 方案 2：URL 版本号参数（cache-busting）

在 JS/CSS 引用和页面 URL 加时间戳版本号：

```html
<!-- 引用静态资源时加版本号 -->
<script src="app.js?v=20260818a"></script>
<link rel="stylesheet" href="style.css?v=20260818a">

<!-- 用户访问时加版本号 -->
https://news.heronwang.cn/?v=new3
```

优点：最可靠，浏览器看到不同 URL 就会重新请求。缺点：每次更新要改版本号。

### 方案 3：fetch 请求加 cache-busting（用于动态加载的 shader/资源）

```javascript
async function loadShaders() {
    const v = '?v=' + Date.now(); // 每次加载都带新时间戳
    const [rayFrag, compFrag, fullVert] = await Promise.all([
        fetch('./shaders/raytracer.frag' + v).then(r => r.text()),
        fetch('./shaders/composite.frag' + v).then(r => r.text()),
        fetch('./shaders/fullscreen.vert' + v).then(r => r.text()),
    ]);
    return { rayFrag, compFrag, fullVert };
}
```

适用于：shader 文件、动态加载的 JSON/TXT 资源、Python http.server 服务的所有静态文件。

## 方案对比

| 方案 | 覆盖层 | 可靠性 | 维护成本 |
|------|--------|--------|---------|
| meta 标签 | 浏览器 | 中（X5 可能不遵守） | 零 |
| URL 版本号 | 浏览器 + CDN | 高 | 每次更新改版本号 |
| fetch cache-busting | 浏览器 | 最高 | 代码中已处理 |
| CF API 清缓存 | CF 边缘 | 高 | 需 API 调用 |
| CF Page Rule Bypass | CF 边缘 | 高 | 用掉 1 条 Page Rule 配额 |

推荐组合：meta 标签（兜底）+ URL 版本号（用户面）+ fetch cache-busting（代码面）。

## 避坑提示

- **CF API Key 可能不对**：memory 中存的 CF Global API Key 可能过期或记错，不要依赖它做缓存清除。优先用 meta 标签 + 版本号
- **微信浏览器最顽固**：标准浏览器 Ctrl+F5 能强制刷新，微信内置浏览器没有这个功能。只能通过清理存储空间或"在浏览器打开"绕过
- **Python http.server 不发 Last-Modified 也不发 Cache-Control**：它只发 `Last-Modified`（文件修改时间），不发 `Cache-Control`。浏览器会用启发式缓存
- **CF Free 版只有 3 条 Page Rules**：不要浪费在缓存清除上，用通配符 `*heronwang.cn/*` 一条覆盖所有子域名
- **`?v=` 参数对 X5 内核有时无效**：X5 可能缓存 URL 时忽略 query string。改用路径版本号（`/v2/app.js`）更可靠，但需要服务端路由支持

## 相关知识

### HTTP 缓存头优先级

```
Cache-Control: no-cache, no-store, must-revalidate  ← 最高优先级
Pragma: no-cache                                     ← HTTP/1.0 回退
Expires: 0                                           ← HTTP/1.0 回退
```

`no-cache` = 每次使用前必须向服务器验证（可以发 304）
`no-store` = 完全不存储
`must-revalidate` = 过期后必须重新验证，不能用过期缓存

### Cloudflare 缓存层级

```
CF 边缘节点缓存 (TTL 由 Page Rule / Response Header 控制)
  → CF Tunnel 传输 (不缓存)
    → 本地 Python http.server (不设缓存头)
      → 浏览器缓存 (启发式 / meta 标签控制)
```

CF 边缘缓存是第一层，如果 CF 缓存了旧 HTML，即使后端更新了用户也看不到。清除 CF 缓存需要 API 调用或等待 TTL 过期。
