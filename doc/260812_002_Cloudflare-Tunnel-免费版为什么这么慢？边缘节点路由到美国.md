# Cloudflare Tunnel 免费版为什么这么慢？边缘节点路由到美国

> **日期**: 2026-08-12  
> **分类**: 踩坑记录  
> **标签**: Cloudflare, 网络, 运维  
> **来源**: hermes

---

## 背景/问题

通过 Cloudflare Tunnel 把 NAS 上的多个网站暴露到公网。访问时发现所有网站都很慢，首字节响应时间（TTFB）在 1-7 秒之间波动。但本地直连 NAS 只要 0.001 秒，说明问题不在服务器，而在 Cloudflare 的网络链路上。

## 原因分析

通过检查 HTTP 响应头发现了关键线索：

```
cf-ray: a29947f80a493050-LAX
```

`cf-ray` 头最后的地名缩写表示处理请求的 Cloudflare 边缘节点。`LAX` = 美国洛杉矶！也就是说，从国内访问网站，请求先飞到美国洛杉矶的 CF 节点，再从那里回到国内 NAS，来回一圈上万公里。

### 尝试过的方案（都没用）

**1. 切换协议：QUIC → HTTP/2**

```yaml
# ~/.cloudflared/config.yml
protocol: http2  # 强制用 HTTP/2 替代 QUIC
```

结果：TTFB 没有明显改善（0.65-5.5 秒仍然波动大）。

**2. 重启 cloudflared 服务**

```bash
sudo systemctl restart cloudflared
```

结果：短暂好转，但很快又恢复到慢的状态。

**3. 检查 CF 缓存**

响应头显示 `cf-cache-status: HIT`，说明 CF 已经缓存了页面。但即使命中缓存，TTFB 仍然慢——因为慢在客户端到 CF 边缘节点的网络延迟，不在 CF 到服务器的回源延迟。

## 根本原因

**Cloudflare 免费版不让你选边缘节点位置。** CF 根据自己的路由算法分配节点，国内用户的请求可能被路由到美国、欧洲等远端节点。只有付费版（Enterprise）才能指定边缘节点位置。

## 解决方案/折中

目前没有完美的免费解决方案，但可以减轻影响：

1. **开启 CF 缓存**：设置 Page Rules `Cache Everything`，让 CF 缓存静态页面。虽然首次加载慢，但后续请求命中缓存会快一些
2. **前端加 loading 动画**：用户体感上不那么卡
3. **考虑国内 CDN**：如果主要面向国内用户，用国内 CDN（如腾讯云、阿里云）回源到 NAS，延迟会低很多
4. **保持 cloudflared 健康检查**：设置 systemd timer 定期检测，QUIC 连接断掉时自动重启

```bash
# 定期检测 CF Tunnel 连通性
curl -s -o /dev/null -w "%{http_code}" --max-time 10 https://your-domain.com/health
# 不通就重启
sudo systemctl restart cloudflared
```

## 避坑提示

- `cf-ray` 头的结尾地名 = 实际处理请求的 CF 边缘节点，用它来判断是否被路由到远端
- `protocol: http2` 配在 cloudflared 的 config.yml 顶层，不是在 originRequest 下面
- CF 免费版最多 3 条 Page Rules，要合理分配
- 如果所有子域名都慢，问题在 CF Tunnel 层面；如果只有一个慢，才需要查具体服务
