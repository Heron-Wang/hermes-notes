# 本地 Web 服务通过 Cloudflare Tunnel 暴露到外网全过程

> **日期**: 2026-08-06  
> **分类**: 经验总结  
> **标签**: 无  
> **来源**: hermes

---

## 背景

家里有一台 NAS（DXP4800），上面跑着各种 Python Web 服务。想把本地服务暴露到公网，让外部可以访问，但又不想用传统的端口映射（运营商封端口 + 安全风险）。

最终方案：**Cloudflare Tunnel（零端口映射）**，免费、安全、自带 HTTPS。

## 整体架构

```
访客浏览器
    ↓ HTTPS
Cloudflare 边缘节点（自动分配最近节点）
    ↓ 加密隧道
本地 cloudflared 进程
    ↓ HTTP
本地 Web 服务（:8080 / :8081 ...）
```

外部完全看不到服务器真实 IP，也不需要开放任何入站端口。

## 步骤一：安装 cloudflared

```bash
# 下载 ARM64 版本（NAS 是 ARM 架构，按实际架构选择）
mkdir -p ~/bin
wget https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-arm64 -O ~/bin/cloudflared
chmod +x ~/bin/cloudflared
```

验证安装：
```bash
~/bin/cloudflared --version
```

## 步骤二：登录 Cloudflare 授权

```bash
~/bin/cloudflared tunnel login
```

会弹出一个 URL，在浏览器中打开并授权你的域名（需要域名已托管在 Cloudflare DNS 上）。

授权后会在 `~/.cloudflared/` 生成 `cert.pem`，这是后续创建隧道的凭据。

> **注意：** 域名必须先迁移到 Cloudflare 的 DNS 管理。在 Cloudflare 控制台添加你的域名，按提示修改域名注册商的 NS 记录即可。

## 步骤三：创建隧道

```bash
~/bin/cloudflared tunnel create heron-web
```

执行后会：
1. 生成一个 Tunnel ID（UUID）
2. 在 `~/.cloudflared/` 生成 `<UUID>.json` 凭据文件

## 步骤四：配置路由

创建 `~/.cloudflared/config.yml`：

```yaml
tunnel: <你的Tunnel-ID>
credentials-file: /home/heron/.cloudflared/<Tunnel-ID>.json

ingress:
  - hostname: heronwang.cn
    service: http://localhost:8080
  - hostname: snake.heronwang.cn
    service: http://localhost:8081
  - service: http_status:404    # 兜底规则，必须有
```

**关键点：**
- 每个 hostname 对应一个本地端口
- 最后必须有一条 `http_status:404` 兜底规则
- 可以随时添加新的 hostname 映射

## 步骤五：创建 DNS 记录

```bash
# 手动方式
~/bin/cloudflared tunnel route dns heron-web heronwang.cn
~/bin/cloudflared tunnel route dns heron-web snake.heronwang.cn
```

这会在 Cloudflare DNS 自动创建 CNAME 记录，指向你的隧道。**不需要手动去面板添加。**

> **自动化方式：** 后续封装了 `add-project.sh` 脚本，一条命令搞定 DNS + 配置 + 重启 + 注册。

## 步骤六：启动隧道

```bash
~/bin/cloudflared tunnel run heron-web
```

或配置为 systemd 服务实现开机自启：

```ini
# /etc/systemd/system/cloudflared.service
[Unit]
Description=Cloudflare Tunnel
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=heron
ExecStart=/home/heron/bin/cloudflared tunnel run heron-web
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl enable --now cloudflared
```

## 步骤七：验证访问

```bash
# 本地验证
curl http://localhost:8080/health

# 外网验证
curl https://heronwang.cn/health
```

浏览器直接访问 `https://heronwang.cn`，Cloudflare 自动提供 HTTPS 证书。

## 子域名自动化添加

封装了 `add-project.sh` 脚本，新增作品时一条命令完成：

```bash
./add-project.sh <slug> <port> <标题> <描述> [技术栈] [repo_url]

# 示例
./add-project.sh snake 8081 "贪吃蛇" "Canvas贪吃蛇游戏" "Python,JavaScript"
```

脚本自动执行：
1. `cloudflared tunnel route dns` 创建 DNS CNAME
2. 编辑 `config.yml` 添加 ingress 规则
3. 重启 cloudflared 进程
4. 调用主站 API 注册作品到作品集

## Cloudflare Tunnel 免费版限制

- **用户数：** 最多 50 个 Zero Trust 用户
- **流量：** 无明确上限，但禁止视频流/大文件 CDN 用途
- **性能：** 不能选节点，自动分配最近边缘节点
- **协议：** 仅支持 HTTP/HTTPS/TCP，不支持 UDP
- **SLA：** 无保障

对于个人 Web 服务完全够用。

## 踩坑记录

### 1. curl 通过代理访问失败
本地配置了 HTTP 代理（`https_proxy` 环境变量），curl 访问自己的 Cloudflare 域名时走了代理导致 SSL 错误。

**解决：** 在 `no_proxy` 环境变量中加上自己的域名，或直接用 `curl --noproxy *` 测试。

### 2. sudo 密码问题
systemctl restart 需要 sudo，但远程环境没有交互终端。

**解决：** 直接 kill 进程后用 `nohup` 或 `background` 方式重启。生产环境建议配好 systemd + sudoers 免密。

### 3. config.yml 缩进
YAML 对缩进敏感，`sed` 插入 ingress 规则时要注意空格。

**解决：** 使用 `\n` 确保换行和缩进正确，或直接用 Python 写配置文件。

### 4. DNS 传播延迟
创建 DNS 记录后可能有几十秒的传播延迟。

**解决：** `cloudflared tunnel route dns` 执行后等几秒再测试，Cloudflare 的 DNS 传播通常很快（< 1 分钟）。

## 最终目录结构

```
/home/heron/
├── bin/cloudflared                    # tunnel 客户端
├── .cloudflared/
│   ├── config.yml                     # 隧道路由配置
│   ├── cert.pem                       # 登录凭据
│   └── <UUID>.json                    # 隧道凭据
├── web-service/                       # 主站 :8080
│   ├── server.py
│   ├── db.py
│   ├── sanitize.py
│   ├── config.py
│   └── add-project.sh
└── projects/
    └── snake/                         # 贪吃蛇 :8081
        └── server.py
```

## 总结

| 方案 | 端口映射 | FRP | Cloudflare Tunnel |
|------|---------|-----|-------------------|
| 安全性 | 低 | 中 | 高（隐藏真实IP） |
| 配置难度 | 低 | 中 | 低 |
| HTTPS | 需自配 | 需自配 | 自动 |
| 多域名路由 | 不支持 | 复杂 | 简单 |
| 免费可用 | 取决于运营商 | 是 | 是 |

Cloudflare Tunnel 是个人开发者暴露本地服务到公网的最佳方案，零成本、高安全、易维护。
