# Hermes Gateway 重启后未自启动导致微信断连

## 背景

NAS（DXP4800-5644）于 2026-09-01 08:00 重启后，用户发现 Hermes 微信连接失败，微信消息无响应。通过 CLI 启动排查。

## 排查过程

### 1. 确认 Gateway 进程状态

```bash
# 检查 systemd 用户服务
systemctl --user status hermes-gateway
# 输出：○ hermes-gateway.service ... Active: inactive (dead)

# 检查进程
ps aux | grep hermes | grep -v grep
# 只有 CLI 进程在运行，gateway 进程不存在
```

关键发现：`hermes-gateway.service` 状态为 `inactive (dead)`，尽管服务已 `enabled`。

### 2. 检查服务配置

```bash
cat ~/.config/systemd/user/hermes-gateway.service
```

服务文件关键配置：
```ini
[Service]
Type=simple
ExecStart=/home/heron/.hermes/hermes-agent/venv/bin/python -m hermes_cli.main gateway run
Restart=always
RestartSec=5
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=default.target
```

服务已 `enabled`，linger 也已启用（`systemctl --user status` 显示 `✓ Systemd linger is enabled`），按理应该在开机时自启动。

### 3. 分析 Gateway 退出历史

通过 `gateway-exit-diag.log` 追踪：

```
2026-08-23T17:13:15Z  gateway.start   PID=1885931
2026-08-23T17:25:46Z  gateway.exit_nonzero  PID=1885931  ← 收到 SIGTERM 退出
--- 8天空窗期 ---
2026-09-01T15:14:52Z  gateway.start   PID=132939  ← 手动启动
```

Gateway 在 8 月 24 日凌晨收到 SIGTERM（NAS 关机时的正常 shutdown 信号）退出，退出码为 1。

### 4. 根因分析

退出日志中的关键信息：
```
Exiting with code 1 (signal-initiated shutdown without restart request)
so systemd Restart=on-failure can revive the gateway.
```

虽然服务配置了 `Restart=always`，但 SIGTERM 是 systemd 在关机时发送的**正常停止信号**，不算"异常退出"。systemd 在关机流程中停止所有服务后，`Restart=always` 不会在关机期间触发重启。

**重启后服务未自启动的原因**：虽然服务 `enabled` + `WantedBy=default.target` + linger 已启用，理论上应该在用户会话初始化时启动。但实际没有启动——可能是以下原因之一：

1. NAS 关机时 systemd 先停了 gateway（SIGTERM），服务状态变为 `inactive`
2. 重启后 `default.target` 的 wants 列表应该包含该服务，但实际未触发
3. 可能是 `network-online.target` 未就绪导致启动条件不满足（服务配置了 `After=network-online.target`）

## 解决方案

### 即时恢复

```bash
# 手动启动 gateway
systemctl --user start hermes-gateway

# 确认状态
systemctl --user status hermes-gateway
# Active: active (running) ✓
```

启动后微信连接恢复，Gateway 进程 PID=132939 正常运行。

### 验证微信连接

```bash
# 检查 gateway 日志（注意：新 gateway 日志可能写入 systemd journal 而非 gateway.log）
hermes gateway status
# ✓ User gateway service is running
# ✓ Systemd linger is enabled (service survives logout)
```

## 经验总结

| 要点 | 说明 |
|------|------|
| NAS 重启后检查 gateway | `systemctl --user status hermes-gateway` 确认是否 active |
| 手动启动 | `systemctl --user start hermes-gateway` |
| `Restart=always` 的局限 | 不能跨越系统关机/重启，只在运行时进程退出时生效 |
| 日志位置变化 | 新版 gateway 日志输出到 systemd journal（`StandardOutput=journal`），旧 `gateway.log` 文件不再追加 |
| 检查 journal 日志 | `journalctl --user -u hermes-gateway --since "today" --no-pager` |
| linger 必须启用 | `sudo loginctl enable-linger $USER` 确保用户服务在开机时启动 |

## 坑点

1. **gateway.log 不更新**：新版 gateway 通过 systemd 运行时，stdout/stderr 重定向到 journal（`socket`），而非旧的 `gateway.log` 文件。排查时看 `journalctl --user -u hermes-gateway`，不要只看 `~/.hermes/logs/gateway.log`（该文件可能停留在上次前台运行的记录）。

2. **state.db 符号链接**：`~/.hermes/state.db` → `/volume2/hermes-data/state.db`（NVMe 分区），迁移后权限正常（`-rw-r--r-- heron:admin`），writable。但如果 NVMe 分区未挂载，会导致 state.db 不可写，出现 `attempt to write a readonly database` 错误。

3. **建议添加开机自启动检查**：可添加一个简单的 cron job 或 systemd timer，在开机后检查 gateway 状态，如果 inactive 则自动启动。
