# VSCode Remote-SSH 连接报错：Failed to set up dynamic port forwarding

> **日期**: 2026-08-07  
> **分类**: 踩坑记录  
> **标签**: 无  
> **来源**: hermes

---

## 背景/问题

在 Windows 上用 VSCode Remote-SSH 连接 NAS（Linux 服务器）开发项目，连接时报错：

> Failed to set up dynamic port forwarding connection over SSH to the VS Code Server.

SSH 本身能连上，但 VSCode 就是卡在这里，无法打开远程项目。

## 原因分析

VSCode Remote-SSH 的工作原理和普通 SSH 不一样。它不是简单地连上去执行命令，而是：

1. 在远程服务器上自动下载并启动一个 **VSCode Server** 进程
2. 通过 **SSH 动态端口转发（dynamic port forwarding）** 在本地和远程 Server 之间建立通信隧道
3. 本地的 VSCode 界面通过这个隧道跟远程 Server 交互

关键就在第 2 步——它需要 SSH 的端口转发功能。而很多 NAS 系统或加固过的 Linux 默认会在 `/etc/ssh/sshd_config` 里写一行：

```
AllowTcpForwarding no
```

这行的意思是"禁止 SSH 做端口转发"，本来是安全加固用的，但直接把 VSCode Remote-SSH 的通信隧道给掐断了。

## 解决方案

### 第 1 步：检查 SSH 配置

```bash
# 查看 sshd_config 中跟转发相关的配置
grep -iE 'AllowTcpForwarding|PermitTunnel|GatewayPorts' /etc/ssh/sshd_config
```

如果看到 `AllowTcpForwarding no`，就是它了。

### 第 2 步：修改配置

```bash
# 把 no 改成 yes
sudo sed -i 's/^AllowTcpForwarding no/AllowTcpForwarding yes/' /etc/ssh/sshd_config

# 确认修改
grep -n 'AllowTcpForwarding' /etc/ssh/sshd_config
```

### 第 3 步：重启 SSH 服务

```bash
sudo systemctl restart ssh
systemctl is-active ssh  # 确认服务正常
```

### 第 4 步：重新连接

回到 Windows VSCode，先 Close Remote Connection（如果有残留连接），然后重新 Connect to Host。

## 避坑提示

- **不要把整个 sshd_config 的安全加固都拆了**。只需要改 `AllowTcpForwarding` 这一项就够了，其他安全配置（如 `PermitRootLogin no`）保持不动。
- **改完一定要重启 SSH 服务**，不然配置不生效。`systemctl restart ssh` 不会断开你当前的 SSH 连接，放心执行。
- **首次连接会比较慢**，因为 VSCode Server 要下载到远程服务器上（约 30MB），耐心等待。如果超时，多重试几次。
- **建议配置 SSH 免密登录**，不然每次连接都要输密码很烦。在 Windows 上执行 `ssh-keygen -t ed25519` 生成密钥，然后把公钥传到服务器的 `~/.ssh/authorized_keys`。
- **如果改了 AllowTcpForwarding 还不行**，检查 `PermitTunnel` 是否也是 `no`，同样需要改成 `yes`。
