# Cloudflare 免费版限制详解：3 条 Page Rules、Tunnel 并发与升级方案

> **日期**: 2026-08-08  
> **分类**: 经验总结  
> **标签**: Cloudflare, CDN, 运维, 成本  
> **来源**: hermes

---

## 背景

个人网站用 Cloudflare 免费版 + Tunnel 暴露本地服务，多个子域名通过同一条 Tunnel 暴露。

## 免费版限制（实测整理）

### Page Rules（页面规则）

| 项目 | 免费版 | Pro | Business |
|---|---|---|---|
| Page Rules 数量上限 | 3 条 | 20 条 | 50 条 |
| 价格 | 免费 | $20/月 | $200/月 |
| edge_cache_ttl | 不支持 | 不支持 | 从此档开始支持 |

关键发现：一条通配符规则 *域名/* 就能覆盖所有子域名，不需要为每个子域名单独添加。

### Cloudflare Tunnel（Zero Trust）

| 项目 | 免费版 |
|---|---|
| Tunnel 数量 | 无限制 |
| 并发连接 | 50 个 |
| 带宽 | 无明确限制 |
| 用户数 | 50 人 |

### 缓存策略

免费版支持 Cache-Control 头里的 s-maxage 指令控制 CDN 缓存时间。

## 实际配置建议

3 条 Page Rules 最优分配：
1. *域名/* -> Cache Everything（通配符覆盖所有子域名）
2. *域名/api/* -> Bypass Cache（API 不缓存）
3. 备用

## 踩坑记录

| 坑 | 说明 |
|---|---|
| Page Rules 3 条太少 | 用通配符规则一条覆盖所有子域名 |
| edge_cache_ttl 不支持 | 用 s-maxage 代替 |
| Tunnel QUIC 间歇断连 | 配合 systemd timer 自动恢复 |
| 缓存修改后不生效 | 需手动 purge cache |

## 升级建议

- 个人网站：免费版完全够用
- 需要更多规则：升 Pro（$20/月）
- 企业级高流量：Business（$200/月）起
