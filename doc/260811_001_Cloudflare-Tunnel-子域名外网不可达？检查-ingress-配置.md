# Cloudflare Tunnel 子域名外网不可达？检查 ingress 配置

> **日期**: 2026-08-11  
> **分类**: 踩坑记录  
> **标签**: Cloudflare, Cloudflare Tunnel, 网络配置, 运维  
> **来源**: hermes

---

## 背景/问题

把好几个本地服务通过 Cloudflare Tunnel 暴露到外网，有一天突然发现其中一个子域名（比如 `blog.heronwang.cn`）访问不了，浏览器直接报「无法解析域名」。但其他子域名都好好的，本地服务也没挂。

## 原因分析

Cloudflare Tunnel 的工作原理是：本地跑一个 `cloudflared` 进程，它跟 Cloudflare 的边缘服务器建立隧道，然后根据 **ingress 配置文件** 把不同子域名的请求路由到本地不同端口的服务。

关键点：**每个子域名需要在两个地方都配置好才能访问**：

1. **Cloudflare DNS 面板** — 添加一条 CNAME 记录，把子域名指向你的 Tunnel ID
2. **本地 ingress 配置文件** — 写明该子域名的请求转发到哪个本地端口

如果只在 DNS 加了记录但 ingress 配置里忘了加路由规则，`cloudflared` 收到请求后不知道转给谁，就会返回 404。如果连 DNS 记录都没加，那就根本解析不了域名。

## 解决方案

### 第 1 步：查看当前 ingress 配置

```bash
# 配置文件通常在这两个位置之一
cat ~/.cloudflared/config.yml
# 或
cat /etc/cloudflared/config.yml
```

配置文件长这样：

```yaml
tunnel: 8b5c78ea-xxxx-xxxx-xxxx-xxxxxxxxxxxx
credentials-file: /home/user/.cloudflared/8b5c78ea-xxxx.json

ingress:
  - hostname: heronwang.cn        # 主站
    service: http://localhost:8080
  - hostname: snake.heronwang.cn  # 子域名1
    service: http://localhost:8081
  # ← 如果这里没有 blog 的规则，那就是漏配了
  - service: http_status:404      # 兜底规则，必须在最后
```

### 第 2 步：添加缺失的子域名规则

在 ingress 列表里加一条（注意必须在兜底的 `http_status:404` 之前）：

```yaml
ingress:
  - hostname: heronwang.cn
    service: http://localhost:8080
  - hostname: blog.heronwang.cn    # 新增！
    service: http://localhost:8090  # 指向本地 blog 服务端口
  - service: http_status:404       # 兜底，必须放最后
```

### 第 3 步：确认 Cloudflare DNS 有对应记录

```bash
# 测试 DNS 是否能解析
dig +short blog.heronwang.cn A
# 如果返回空，说明 Cloudflare DNS 面板里还没加 CNAME 记录
# 去 Cloudflare Dashboard -> DNS -> 添加 CNAME：
#   Name: blog
#   Target: <tunnel-id>.cfargotunnel.com
#   Proxy: 开启（橙色云朵）
```

### 第 4 步：重启 cloudflared 让配置生效

```bash
sudo systemctl restart cloudflared
# 验证
curl -sS -o /dev/null -w "HTTP %{http_code}, time %{time_total}s\n" https://blog.heronwang.cn
```

## 避坑提示

- **ingress 规则顺序很重要**：`http_status:404` 这条兜底规则必须放在最后，否则它后面的规则都不会匹配
- **DNS 和 ingress 缺一不可**：光加 DNS 不加 ingress → cloudflared 返回 404；光加 ingress 不加 DNS → 域名根本不解析
- **排查工具推荐**：用 `curl -sS -o /dev/null -w "%{http_code}" --noproxy '*' URL` 绕过代理直连测试，避免本地代理环境变量干扰判断
- **Free 套餐限制**：Cloudflare 免费版的 Page Rules 上限是 3 条，但 ingress 规则没有数量限制，别搞混了
