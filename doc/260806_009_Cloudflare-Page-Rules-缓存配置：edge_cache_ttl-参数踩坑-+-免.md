# Cloudflare Page Rules 缓存配置：edge_cache_ttl 参数踩坑 + 免费版3条限制

> **日期**: 2026-08-06  
> **分类**: 踩坑记录  
> **标签**: 无  
> **来源**: hermes

---

## 背景/问题

网站通过 Cloudflare Tunnel 部署后，每次访问都要穿越隧道回源，TTFB（首字节时间）高达 1.3~3.4 秒。想用 Cloudflare 的边缘缓存来加速，但配置 Page Rules 的过程中踩了两个坑。

## 原因分析

Cloudflare 免费版默认**不缓存 HTML 和 JSON**（只缓存静态资源如 CSS/JS/图片）。要缓存动态内容，必须通过 Page Rules 设置 `Cache Everything`。

但 Page Rules API 有两个隐藏的坑：
1. **`edge_cache_ttl` 参数不能和 `cache_level` 一起用**，加了就会验证失败（返回 `code: 1004`）
2. **免费版 Page Rules 限制 3 条**，超过会返回 `code: 1008: Page Rule limit has been met`

## 解决方案

### 第一步：去掉 edge_cache_ttl，只用 cache_level

```json
// ❌ 会验证失败的写法
{
  "actions": [
    {"id": "cache_level", "value": "cache_everything"},
    {"id": "edge_cache_ttl", "value": 300}  // ← 这个参数导致验证失败！
  ]
}

// ✅ 正确写法：只用 cache_level
{
  "actions": [{"id": "cache_level", "value": "cache_everything"}],
  "priority": 1,
  "status": "active"
}
```

### 第二步：用通配符合并规则（绕过3条限制）

免费版只能创建 3 条 Page Rules。用通配符 `*域名/*` 一条规则覆盖所有：

| 优先级 | 匹配规则 | 动作 |
|--------|---------|------|
| 1 | `*example.com/*` | Cache Everything（覆盖所有子域名所有路径） |
| 2 | `example.com/api/notes*` | Cache Everything |
| 3 | `example.com/api/portfolio*` | Cache Everything |

### 第三步：缓存需要预热

配好规则后，第一次请求仍然是 `cf-cache-status: MISS`。需要**多请求几次**让 CF 缓存上：

```bash
# 预热缓存
for i in 1 2 3; do
  curl -s -o /dev/null https://example.com/api/notes
  sleep 1
done

# 验证缓存状态
curl -sI https://example.com/ | grep -i "cf-cache"
# 输出：cf-cache-status: HIT  ← 缓存命中！
```

## 关键命令：通过 API 创建 Page Rules

```bash
CF_EMAIL="你的邮箱"
CF_KEY="你的Global API Key"  # CF控制台 → My Profile → API Tokens
ZONE="你的Zone ID"  # CF控制台 → 域名概述页右侧

curl -s -X POST \
  -H "X-Auth-Email: $CF_EMAIL" \
  -H "X-Auth-Key: $CF_KEY" \
  -H "Content-Type: application/json" \
  "https://api.cloudflare.com/client/v4/zones/$ZONE/pagerules" \
  -d '{"targets":[{"target":"url","constraint":{"operator":"matches","value":"*example.com/*"}}],"actions":[{"id":"cache_level","value":"cache_everything"}],"priority":1,"status":"active"}'
```

## 避坑提示

- ⚠️ **`edge_cache_ttl` 不要和 `cache_level` 混用**——会直接验证失败。控制缓存时间请用源站的 `Cache-Control` 响应头（`s-maxage`）
- ⚠️ **免费版只有 3 条 Page Rules**——用通配符覆盖尽量多的路径
- ⚠️ **缓存不是立即生效的**——第一次一定是 `MISS`，需要预热
- ⚠️ **`*域名/` 和 `*域名/*` 不同**——前者只匹配首页，后者匹配所有路径
- 💡 **效果**：配置后 TTFB 从 1.3~3.4s 降到 0.67~0.69s，提升约 2-5 倍
