# 没有公网IP怎么让外网访问本地服务？Cloudflare Tunnel 方案

> **日期**: 2026-08-06  
> **分类**: 经验总结  
> **标签**: 无  
> **来源**: hermes

---

## 背景：家里 NAS 没有公网 IP

很多人家里有个 NAS 或者小服务器，想从外面访问上面跑的服务，但遇到了两个问题：
1. 运营商不给公网 IP（大多是内网地址）
2. 路由器端口转发需要公网 IP，没法用

这时候就需要「内网穿透」——让外网能访问到内网的设备。

## 为什么选 Cloudflare Tunnel？

| 方案 | 需要公网IP | 需要改路由器 | HTTPS | 费用 |
|------|-----------|-------------|-------|------|
| 端口转发 | 需要 | 需要 | 自己搞 | 免费 |
| frp | 需要VPS | 不需要 | 自己搞 | VPS费用 |
| ngrok | 不需要 | 不需要 | 自带 | 有限制 |
| **Cloudflare Tunnel** | **不需要** | **不需要** | **自动** | **免费** |

Cloudflare Tunnel 的原理很简单：你的机器**主动**连到 Cloudflare 的服务器（出站连接，不需要开端口），Cloudflare 帮你把外网请求转发过来。

```
外网用户 → Cloudflare CDN（自动HTTPS）→ 加密隧道 → 你的机器:8080
```

外人完全看不到你的真实 IP，也不用开任何入站端口。

## 怎么做？分 7 步

### 第 1 步：安装 cloudflared

```bash
mkdir -p ~/bin
wget https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64 -O ~/bin/cloudflared
chmod +x ~/bin/cloudflared
```

### 第 2 步：把域名迁移到 Cloudflare DNS

> **为什么？** Tunnel 需要管理你的域名 DNS 记录，所以域名得托管在 Cloudflare 上。

1. 注册 Cloudflare 账号（免费）
2. 添加你的域名，CF 会给两个 NS 服务器地址
3. 去域名注册商把 NS 改成 CF 给的

### 第 3 步：登录授权

```bash
~/bin/cloudflared tunnel login
```

会弹出一个 URL，浏览器打开并授权。授权后生成 `~/.cloudflared/cert.pem`。

### 第 4 步：创建隧道

```bash
~/bin/cloudflared tunnel create heron-web
```

生成 Tunnel ID 和凭据文件 `~/.cloudflared/<UUID>.json`。

### 第 5 步：写配置文件

创建 `~/.cloudflared/config.yml`：

```yaml
tunnel: <你的Tunnel-ID>
credentials-file: /home/yourname/.cloudflared/<Tunnel-ID>.json

ingress:
  - hostname: yourdomain.cn
    service: http://localhost:8080
  - hostname: app.yourdomain.cn
    service: http://localhost:8081
  - service: http_status:404    # 兜底规则（必须有）
```

> **关键点：** 最后一条 http_status:404 是必须的，否则启动报错。

### 第 6 步：创建 DNS 记录

```bash
~/bin/cloudflared tunnel route dns heron-web yourdomain.cn
```

自动在 Cloudflare 创建 CNAME 记录，**不需要手动去面板添加**。

### 第 7 步：启动 + 设为开机自启

```ini
# /etc/systemd/system/cloudflared.service
[Unit]
Description=Cloudflare Tunnel
After=network-online.target

[Service]
Type=simple
User=yourname
ExecStart=/home/yourname/bin/cloudflared tunnel run heron-web
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl enable --now cloudflared
```

## 踩过的坑

### 坑1：curl 访问自己的域名报 SSL 错误
**原因：** 本机配了 HTTP 代理，curl 走了代理。
**解决：** no_proxy 加上自己的域名。

### 坑2：DNS A 记录冲突
**原因：** 域名迁移时自动生成 A 记录，和 CNAME 冲突。
**解决：** 去 CF 面板删掉冲突的 A 记录，重新执行 route dns。

### 坑3：隧道偶尔断开（502）
**原因：** QUIC 连接超时。
**解决：** systemd 配置 Restart=always，断开后自动重连。

### 坑4：sudo 需要密码无法自动化
**解决：** 用 SUDO_ASKPASS 指定密码脚本，配合 sudo -A。
