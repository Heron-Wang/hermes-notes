# HTML 幻灯片预览页怎么做手机端适配？iframe 16:9 自适应 + CDN 缓存绕过

> **日期**: 2026-08-14  
> **分类**: 经验总结  
> **标签**: 前端, CSS, CDN, Cloudflare, 响应式  
> **来源**: hermes

---

## 背景/问题

你做了一个自动化新闻视频生成系统，其中有一个 PPT 预览页面——多张 1920x1080 的 HTML 幻灯片通过 iframe 嵌入到一个预览页面中，方便在浏览器里查看当天生成的新闻幻灯片效果。

在电脑上看效果很好，但用手机打开后发现两个严重问题：

1. **幻灯片在手机上横向溢出**：iframe 设了固定 540px 高度，但手机屏幕只有 375px 宽，幻灯片内容被裁切，需要左右滑动才能看全
2. **更新了代码但手机上看到的还是旧版本**：你修改了幻灯片的 CSS 动画效果，在电脑上刷新能看到新效果，但手机上打开还是旧的——因为 CDN 缓存了旧版本的 HTML 文件

这两个问题在移动端 Web 开发中非常常见。一个是响应式布局问题（如何让固定尺寸的内容在任意屏幕宽度下正确显示），一个是 CDN 缓存问题（如何确保用户看到最新版本）。

## 原因分析

### 问题 1：iframe 固定尺寸不适配手机屏幕

最初的预览页面代码是这样的：

```html
<!-- 固定高度 iframe -->
<div class="slide-frame">
  <iframe src="slides/slide_0.html" style="width:100%;height:540px;"></iframe>
</div>
```

问题在于：
- 幻灯片内容是 1920x1080（16:9 比例）设计的
- iframe 设了 width:100%（会随屏幕宽度缩放），但 height:540px 是固定的
- 在电脑上（960px 容器宽度），16:9 比例的高度应该是 540px，刚好匹配
- 但在手机上（375px 容器宽度），16:9 比例的高度应该是 210px，而 iframe 仍然占了 540px——内容被拉伸变形，或者底部出现大片空白

更根本的问题是：**iframe 的高度无法根据宽度自动按比例缩放**。CSS 没有简单的 height = width * 9/16 语法。

### 问题 2：Cloudflare CDN 缓存了旧版本

网站通过 Cloudflare Tunnel 暴露到公网，Cloudflare 的边缘节点会缓存 HTML 静态文件。当你更新了 slides/ 目录下的 HTML 文件后：

浏览器请求 -> Cloudflare CDN -> 检查缓存 -> 缓存命中(旧版本) -> 返回旧的 HTML 内容

Cloudflare 的缓存策略默认会缓存 .html 文件（特别是配置了 Cache Everything Page Rule 的情况下）。缓存有效期由 Cache-Control 响应头或 Page Rule 设置的 TTL 决定。在缓存过期之前，所有用户拿到的都是旧版本。

这意味着你改了代码、重启了服务、甚至清了浏览器缓存都没用——因为缓存不在浏览器端，而在 Cloudflare 的 CDN 边缘节点上。

## 排查过程

### 第一步：确认手机端布局问题

用 Chrome DevTools 的设备模拟器（F12 -> 切换设备工具栏）查看 iPhone SE (375px 宽度)：

- iframe 宽度：375px（正确，跟随屏幕宽度）
- iframe 高度：540px（固定值，不正确）
- 幻灯片内容：1920px 宽的 HTML 被压缩进 375px 宽的 iframe
- 结果：内容变形 + 高度不匹配

### 第二步：确认 CDN 缓存问题

```bash
# 本地直接访问（绕过 CDN）
curl -s http://localhost:8092/slides/slide_0.html | grep 'glowPulse|fadeInUp'
# -> 能搜到新动画 CSS  本地是新版本

# 外网通过 CDN 访问
curl -s https://ppt.heronwang.cn/slides/slide_0.html | grep 'glowPulse|fadeInUp'
# -> 搜不到  CDN 返回的是旧版本！

# 检查 CDN 缓存状态
curl -sI https://ppt.heronwang.cn/slides/slide_0.html | grep -i 'cf-cache'
# -> cf-cache-status: HIT  确认命中缓存
```

### 第三步：尝试 Cloudflare 缓存清除

```bash
# 通过 Cloudflare API 清除缓存
curl -s -X POST \
  "https://api.cloudflare.com/client/v4/zones/ZONE_ID/purge_cache" \
  -H "X-Auth-Email: ***REDACTED***" \
  -H "X-Auth-Key: ***REDACTED***" \
  -H "Content-Type: application/json" \
  --data '{"purge_everything":true}'
# -> {"success":true}  清除成功
```

但全量清除缓存有个问题：**会影响所有用户的所有页面**。而且 CDN 节点重新缓存需要时间（用户第一次访问时是 MISS，需要回源获取）。对于频繁更新的预览页面，每次改代码都要清 CDN 缓存太麻烦了。

## 解决方案

### 步骤 1：用 padding-top 实现 16:9 自适应容器

CSS 中实现固定比例容器最经典的方法是 **padding-top 百分比** 技术：

```html
<!-- 16:9 自适应容器 -->
<div class="slide-wrapper">
  <div class="ratio">
    <iframe src="slides/slide_0.html?v=7" loading="lazy"></iframe>
  </div>
</div>
```

```css
.slide-wrapper {
  width: 100%;
  max-width: 960px;        /* 桌面端最大宽度 */
  position: relative;
  border-radius: 12px;
  overflow: hidden;
  background: #000;
}

/* 16:9 比例容器 */
.slide-wrapper .ratio {
  padding-top: 56.25%;      /* 9/16 = 0.5625 = 56.25% */
  position: relative;
}

.slide-wrapper iframe {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  border: none;
  display: block;
}
```

**原理详解：**

padding-top 的百分比是相对于**父元素宽度**计算的（注意：不是高度！）。所以：
- 父元素宽度 375px（手机）-> padding-top: 56.25% = 210.9px -> 容器高度 210.9px
- 父元素宽度 960px（桌面）-> padding-top: 56.25% = 540px -> 容器高度 540px

容器的高度永远 = 宽度 * 9/16，完美保持 16:9 比例。iframe 用 position:absolute 填满整个容器。

### 步骤 2：用版本号参数绕过 CDN 缓存

```html
<!-- 不带版本号 - CDN 可能返回旧缓存 -->
<iframe src="slides/slide_0.html"></iframe>

<!-- 带版本号 - CDN 视为新 URL，绕过缓存 -->
<iframe src="slides/slide_0.html?v=7"></iframe>
```

**原理：** CDN 缓存的 key 是完整 URL（含查询参数）。slide_0.html 和 slide_0.html?v=7 是两个不同的缓存 key。加 ?v=7 后，CDN 发现这个 URL 没有缓存记录（MISS），就会回源获取最新内容。

每次更新代码后，把版本号 +1（v=8, v=9...），所有用户都能立即看到新版本，不需要清 CDN 缓存。

### 步骤 3：添加懒加载优化性能

```html
<iframe src="slides/slide_0.html?v=7" loading="lazy"></iframe>
```

loading="lazy" 是 HTML 原生懒加载属性，浏览器会在 iframe 滚动到可视区域附近时才发起请求。对于有 8 张幻灯片的预览页面，初始加载只需要加载第一张，其余的按需加载。

### 步骤 4：手机端专项优化

```css
/* 默认（手机端）样式 */
body {
  padding: 12px;       /* 手机端小间距 */
  gap: 10px;            /* 幻灯片之间间距小 */
}
h1 {
  font-size: 1.1rem;   /* 手机端小标题 */
}
.slide-label {
  font-size: 0.75rem;  /* 小标签 */
}

/* 桌面端（>=768px）样式 */
@media (min-width: 768px) {
  body {
    padding: 20px;      /* 桌面端大间距 */
    gap: 16px;
  }
  h1 {
    font-size: 1.4rem;  /* 桌面端大标题 */
  }
  .slide-label {
    font-size: 0.8rem;
  }
}
```

同时设置 viewport 允许缩放：

```html
<meta name="viewport" content="width=device-width,initial-scale=1.0,maximum-scale=5.0">
```

maximum-scale=5.0 允许双指放大查看细节。默认有些网页设 maximum-scale=1.0 禁止缩放，对用户不友好。

### 步骤 5：验证修复效果

```bash
# 1. 本地验证
curl -s http://localhost:8092/preview.html | grep '56.25|ratio|lazy'
# -> .slide-wrapper .ratio{padding-top:56.25%
# -> loading="lazy"

# 2. 外网验证（带版本号）
curl -s 'https://ppt.heronwang.cn/slides/slide_0.html?v=7' | grep 'glowPulse|fadeInUp'
# -> 能搜到新动画 CSS  CDN 返回了新版本！

# 3. 外网验证预览页面
curl -s 'https://ppt.heronwang.cn/preview.html?v=7' | grep 'ratio|56.25|lazy'
# -> 确认预览页也是新版本
```

## 方案对比

### 16:9 自适应布局方案对比

| 方案 | 原理 | 优点 | 缺点 | 兼容性 |
|------|------|------|------|--------|
| **padding-top 百分比** | padding-top 相对父宽度计算 | 零 JS、纯 CSS、完美比例 | 需要嵌套 div | 所有浏览器 |
| aspect-ratio CSS | aspect-ratio: 16/9 | 一行 CSS 搞定 | 旧浏览器不支持 | IE 不支持 |
| JS 计算高度 | resize 监听器动态算 | 精确控制 | 需要 JS、性能差 | 所有浏览器 |
| 固定高度 | height: 540px | 简单 | 不适配不同屏幕 | 不推荐 |

### CDN 缓存绕过方案对比

| 方案 | 操作 | 优点 | 缺点 | 适用场景 |
|------|------|------|------|----------|
| **URL 版本号** | ?v=7 | 即时生效、无需清缓存 | 需手动改版本号 | 频繁更新 |
| 文件名哈希 | slide_abc123.html | 自动化、永久缓存 | 需构建工具 | 生产环境 |
| Cloudflare API 清缓存 | purge_cache | 全量清除 | 影响所有用户 | 紧急修复 |
| Cache-Control: no-cache | 响应头 | 浏览器不缓存 | CDN 可能仍缓存 | 不彻底 |
| 等待缓存自然过期 | 等 TTL 过期 | 零操作 | 可能等几小时 | 不推荐 |

## 避坑提示

- **padding-top 百分比是相对父元素宽度算的，不是高度**：这是 CSS 规范的一个特性，很多人第一次用会搞混。padding-top: 56.25% 在 375px 宽容器里 = 210.9px，不是 56.25% 的容器高度
- **iframe 内部内容不受外层 CSS 影响**：iframe 是独立的浏览上下文，外层的 font-size、color 等 CSS 不会影响 iframe 内部。只能通过 iframe 的宽高来控制显示区域
- **loading="lazy" 对 iframe 同样有效**：不只是 img，iframe 也支持原生懒加载，但需要浏览器支持（Chrome 77+、Firefox 121+）
- **?v=7 不是清除缓存，而是让 CDN 认为是新 URL**：CDN 缓存的 key 包含查询参数，不同参数 = 不同缓存条目。旧版本的缓存仍然存在，只是不会被请求到
- **maximum-scale=1.0 禁止缩放是不友好的做法**：虽然能防止用户双击放大，但也剥夺了用户查看细节的能力。设 maximum-scale=5.0 既防止意外缩放又保留手动缩放能力
- **@media (min-width: 768px) 的断点选择**：768px 是平板/手机的分界线（iPad 竖屏宽度），小于 768px 用手机样式，大于等于用桌面样式
- **多个 iframe 页面会影响首屏加载速度**：即使加了 loading="lazy"，首屏的 iframe 仍会立即加载。可以考虑只加载第一张，其余用 IntersectionObserver 控制

## 相关知识

### CSS aspect-ratio 新特性

除了 padding-top 百分比方案，CSS 现在有了原生的 aspect-ratio 属性：

```css
.slide-wrapper {
  aspect-ratio: 16 / 9;
  width: 100%;
  max-width: 960px;
}
```

aspect-ratio 属性让浏览器自动根据宽度计算高度，保持指定比例。比 padding-top 方案更简洁直观。但目前 IE 完全不支持，Safari 15+ 才支持。如果你的目标用户不需要兼容旧浏览器，aspect-ratio 是更好的选择。

### CDN 缓存机制详解

CDN（Content Delivery Network）的缓存机制可以分为几个层次：

1. **浏览器缓存**：浏览器本地缓存，由 Cache-Control 头控制（如 max-age=3600 表示缓存 1 小时）
2. **CDN 边缘缓存**：CDN 节点上的缓存，由 CDN 的缓存规则控制（如 Cloudflare 的 Page Rules）
3. **源站**：你的原始服务器，CDN 缓存未命中时回源获取

当用户请求一个 URL 时：浏览器 -> 检查本地缓存 -> 检查 CDN 边缘缓存 -> 回源

缓存 key 通常是完整 URL（含查询参数）。所以 file.html?v=1 和 file.html?v=2 是两个独立的缓存条目。这就是版本号方案能绕过缓存的根本原因。

### viewport meta 标签详解

| 参数 | 含义 | 推荐值 |
|------|------|--------|
| width=device-width | 视口宽度 = 设备宽度 | 必须设 |
| initial-scale=1.0 | 初始缩放比例 | 1.0 |
| maximum-scale=5.0 | 最大缩放比例 | 5.0（允许放大） |
| user-scalable=yes | 是否允许用户缩放 | yes（默认） |

不设 viewport 的话，手机浏览器会假设页面宽度为 980px（虚拟视口），导致页面被缩小显示，文字变小难以阅读。

## 关键命令速查

### 16:9 自适应 iframe 容器

```html
<div class="slide-wrapper">
  <div class="ratio">
    <iframe src="content.html?v=1" loading="lazy"></iframe>
  </div>
</div>
```

```css
/* padding-top 方案（兼容性好） */
.slide-wrapper { width:100%; max-width:960px; position:relative; overflow:hidden; }
.slide-wrapper .ratio { padding-top:56.25%; position:relative; }
.slide-wrapper iframe { position:absolute; top:0; left:0; width:100%; height:100%; border:none; }

/* 或 aspect-ratio 方案（更简洁，需新浏览器） */
.slide-wrapper { width:100%; max-width:960px; aspect-ratio:16/9; }
.slide-wrapper iframe { width:100%; height:100%; border:none; }
```

### CDN 缓存绕过

```html
<!-- 每次更新后改版本号 -->
<link rel="stylesheet" href="style.css?v=20260814">
<iframe src="page.html?v=7"></iframe>
<script src="app.js?v=7"></script>
```

```bash
# 检查 CDN 缓存状态
curl -sI https://example.com/page.html | grep -i 'cf-cache'
# HIT = 命中缓存（可能不是最新）
# MISS = 未命中（回源获取最新）

# Cloudflare API 清除缓存（紧急情况用）
curl -s -X POST \
  "https://api.cloudflare.com/client/v4/zones/ZONE_ID/purge_cache" \
  -H "X-Auth-Email: ***REDACTED***" \
  -H "X-Auth-Key: ***REDACTED***" \
  -H "Content-Type: application/json" \
  --data '{"purge_everything":true}'
```

### 移动端 viewport 配置

```html
<meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=5.0">

<style>
  /* 手机优先（默认） */
  body { padding: 12px; font-size: 14px; }

  /* 桌面端（>=768px） */
  @media (min-width: 768px) {
    body { padding: 20px; font-size: 16px; }
  }
</style>
```

### 验证清单

```bash
# 1. 本地内容正确
curl -s http://localhost:PORT/page.html | grep '关键词'

# 2. CDN 返回最新版本（带版本号）
curl -s 'https://domain.com/page.html?v=N' | grep '关键词'

# 3. CDN 缓存状态
curl -sI 'https://domain.com/page.html?v=N' | grep -i 'cf-cache'
# -> MISS 或 EXPIRED 表示会回源获取最新

# 4. 响应式布局检查
# Chrome DevTools -> F12 -> 设备工具栏 -> 选 iPhone SE (375px)
# 确认：无横向滚动条、内容不溢出、间距合理
```
