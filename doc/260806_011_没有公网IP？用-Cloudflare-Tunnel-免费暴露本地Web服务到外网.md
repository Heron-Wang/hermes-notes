# 没有公网IP？用 Cloudflare Tunnel 免费暴露本地Web服务到外网

> **日期**: 2026-08-06  
> **分类**: 经验总结  
> **标签**: 无  
> **来源**: hermes

---

## 背景/问题

家里有一台 NAS，想在上面跑个 Web 服务让外网访问。但没有公网 IP（运营商分配的是大内网地址），路由器端口转发行不通。手上有一个域名，怎么破？

## 方案对比

| 方案 | 需要 | 费用 | 难度 | 推荐度 |
|------|------|------|------|--------|
| **Cloudflare Tunnel** | 域名托管在 CF | 免费 | 低 | 最推荐 |
| frp + VPS | 云服务器 | VPS费用 | 中 | 适合有VPS的 |
| ngrok | 无 | 免费有限制 | 低 | 临时测试 |
| 路由器端口转发 | 公网IP | 免费 | 低 | 不适用 |

## 原理

```
外网用户 -> 你的域名 -> Cloudflare CDN -> 隧道 -> 本机:8080

本机 (无公网IP, NAT后面)
  python3 server.py        (Web服务，监听本地端口)
  cloudflared tunnel run   (反向隧道，出站连接到CF)
```

关键点：`cloudflared` 只做**出站连接**到 Cloudflare，不需要公网 IP、不需要开端口、不需要入站连接。Cloudflare 免费提供 HTTPS 证书 + CDN + DDoS 防护。

## 解决方案（4步走）

### 第一步：Quick Tunnel 快速验证（30秒）

不需要域名和账号，先验证链路通不通：

```bash
# 安装 cloudflared
curl -fSL -o /tmp/cloudflared \
  "https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64"
chmod +x /tmp/cloudflared
mkdir -p ~/bin && cp /tmp/cloudflared ~/bin/cloudflared

# 启动本地 Web 服务
python3 server.py &

# 启动 Quick Tunnel——会分配一个随机域名
~/bin/cloudflared tunnel --url http://localhost:8080
# 输出类似：https://xxx-yyy-zzz.trycloudflare.com
```

用手机打开那个 trycloudflare 链接，能访问就说明链路通了。

> Quick Tunnel 的域名每次重启都会变，只适合临时测试。

### 第二步：域名迁移到 Cloudflare

1. 注册 Cloudflare 账号（免费）
2. 添加你的域名，选 Free 计划
3. CF 会给你两个 NS 服务器地址
4. 去域名注册商，把 NS 改成 CF 的
5. 等待 DNS 生效（通常 10-60 分钟）

### 第三步：创建命名隧道 + 绑定域名

```bash
# 登录授权（会打开浏览器）
~/bin/cloudflared tunnel login

# 创建隧道
~/bin/cloudflared tunnel create my-tunnel

# 绑定域名到隧道
~/bin/cloudflared tunnel route dns my-tunnel yourdomain.com

# 写配置文件
cat > ~/.cloudflared/config.yml << 'EOF'
tunnel: <你的隧道UUID>
credentials-file: /home/你的用户名/.cloudflared/<你的隧道UUID>.json

ingress:
  - hostname: yourdomain.com
    service: http://localhost:8080
  - service: http_status:404    # catch-all，必须放最后
EOF

# 启动隧道
~/bin/cloudflared tunnel run my-tunnel
```

### 第四步：systemd 持久化（开机自启）

```ini
# /etc/systemd/system/cloudflared.service
[Unit]
Description=Cloudflare Tunnel
After=network-online.target web-service.service
Wants=network-online.target

[Service]
Type=simple
User=你的用户名
ExecStart=/home/你的用户名/bin/cloudflared tunnel run my-tunnel
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

```bash
sudo cp cloudflared.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now cloudflared
```

## 避坑提示

- **GitHub 下载 cloudflared 在国内很慢**（~120KB/s）——用后台下载，别设短超时
- **二进制下载可能损坏导致 segfault**——用 `curl -fSL`（`-f` 标志在 HTTP 错误时会让 curl 失败）
- **SSL 证书签发有延迟**——域名绑定后第一次访问可能 SSL 报错，等几分钟自动签发
- **QUIC 协议被防火墙拦截也没关系**——cloudflared 会自动回退到 HTTP/2，日志里看到 "degraded transport" 不用慌
- **日志中的 `failed to sufficiently increase receive buffer size` 警告**是 UDP 缓冲区问题，不影响功能
- **sudo 需要密码的机器无法自动安装 systemd 服务**——需要手动在终端执行 sudo 命令
- **验证外网访问**：`curl -s https://你的域名/health` 返回正常就说明整条链路通了
