# VSCode Remote-SSH 连不上 NAS？查 AllowTcpForwarding

> **日期**: 2026-08-09  
> **分类**: 踩坑记录  
> **标签**: VSCode, SSH, NAS, 运维  
> **来源**: hermes

---

## 背景/问题

在 Windows 上用 VSCode Remote-SSH 连接远端 Linux 服务器（比如 NAS），SSH 能登录，VSCode Server 进程也启动了，但就是卡在"Connecting"，最后报错：

> Failed to set up dynamic port forwarding connection over SSH to the VS Code Server

## 原因分析

VSCode Remote-SSH 的工作原理分三步：

1. SSH 连上远端服务器 → **成功**
2. 在远端启动 VSCode Server 进程 → **成功**
3. 通过 SSH **本地端口转发（local port forward）** 和远端 Server 通信 → **被拒绝！**

问题出在服务器的 `/etc/ssh/sshd_config` 里有一行：

```
AllowTcpForwarding no
```

这个选项直接禁止了所有端口转发。SSH 认证能过、Server 能启动，但客户端和 Server 之间的通信隧道建不起来，表现就是"连不上"。

在 auth 日志里能看到明确的拒绝记录：

```
refused local port forward: originator 127.0.0.1 port 1789, target 127.0.0.1 port 38219
```

## 解决方案

把 `AllowTcpForwarding no` 改为 `local`：

```bash
# 修改配置（用 local 而不是 yes，更安全 - 只允许本地转发，不允许远程转发）
sudo sed -i 's/^AllowTcpForwarding no/AllowTcpForwarding local/' /etc/ssh/sshd_config

# 重启 SSH 服务（不会断开当前会话，只影响新连接）
sudo systemctl restart ssh

# 验证
grep AllowTcpForwarding /etc/ssh/sshd_config
# 应该输出: AllowTcpForwarding local
```

## 避坑提示

### 坑1：NAS / 嵌入式系统重启后配置被重置

很多 NAS 系统（如绿联 UGOS）在重启或系统更新时，会用出厂配置覆盖 `/etc/ssh/sshd_config`。具体机制是系统服务执行 `cp /rom/etc/ssh/sshd_config /etc/ssh/sshd_config`，把只读分区里的出厂配置拷回来。

**解决方案：用 drop-in 配置文件做持久化**

OpenSSH 的 `sshd_config` 顶部通常有 `Include /etc/ssh/sshd_config.d/*.conf`，而且 OpenSSH 的规则是"先出现的指令优先"。重置只复制主文件，不会清 `.d` 目录。所以放一个 drop-in 文件就能持久覆盖：

```bash
# 创建 drop-in 配置（主防线 - 即使主文件被重置，drop-in 仍然生效）
echo 'AllowTcpForwarding local' | sudo tee /etc/ssh/sshd_config.d/99-vscode-remote.conf
sudo chmod 644 /etc/ssh/sshd_config.d/99-vscode-remote.conf
```

再加一个 systemd oneshot 服务做备份防线（开机自动检查修复）：

```bash
sudo tee /etc/systemd/system/ssh-preserve-vscode.service << 'EOF'
[Unit]
Description=Preserve AllowTcpForwarding for VSCode Remote SSH
After=ssh.service
Wants=ssh.service

[Service]
Type=oneshot
ExecStart=/bin/bash -c 'grep -q "AllowTcpForwarding local" /etc/ssh/sshd_config || sed -i "s/^AllowTcpForwarding no/AllowTcpForwarding local/" /etc/ssh/sshd_config; systemctl reload ssh || systemctl restart ssh'
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable ssh-preserve-vscode.service
```

### 坑2：怎么确认是端口转发被禁了

查 auth 日志，搜 "refused" 或 "port forward"：

```bash
grep -i 'refused\|port forward' /var/log/auth.log | tail -10
```

如果看到 `refused local port forward`，就是 `AllowTcpForwarding` 的问题。

### 坑3：ForceCommand 也会干扰

有些 NAS 的 sshd_config 里有 `ForceCommand /etc/ssh/force_command.sh`，这个脚本可能限制了非 admin 用户的命令执行。如果你不是 admin 用户，即使开了 AllowTcpForwarding，VSCode Remote 可能还是连不上。检查自己是否在 admin 组：`id -nG $USER`。
