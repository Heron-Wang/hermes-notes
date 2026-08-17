# 设备重启后 Hermes Gateway 微信为什么连不上？systemd 用户服务自动启动排查

> **日期**: 2026-08-16  
> **分类**: 踩坑记录  
> **标签**: Linux, systemd, Hermes, 微信, 运维  
> **来源**: hermes

---

## 背景/问题

你有一个 AI Agent（Hermes Agent）通过 systemd 用户服务运行 Gateway 网关，负责连接微信平台收发消息。平时一切正常——你在微信里给 Agent 发消息，它能自动回复。但某天你发现微信消息发过去完全没有反应，Agent 像消失了一样。

这种情况往往发生在**设备重启之后**。你明明配置了 `Restart=always` 和 `loginctl enable-linger`（让用户服务在开机时自动启动），但 Gateway 就是没有运行。更令人困惑的是，日志里没有任何报错——服务根本就没有尝试启动过。

这个问题涉及 systemd 用户级服务的启动机制、网络就绪时序、以及 DNS 解析的时序问题。对于任何用 systemd 用户服务管理长驻进程的人来说都很常见。

## 原因分析

### 因素 1：systemd 用户服务 vs 系统服务

systemd 有两层服务管理：

| 层级 | 管理命令 | 启动时机 | 典型服务 |
|------|---------|---------|--------|
| 系统服务 | `systemctl status xxx` | 开机时启动 | nginx, docker, sshd |
| 用户服务 | `systemctl --user status xxx` | 用户登录时启动 | Agent gateway, IDE server |

用户服务默认在**用户首次登录时**启动，而不是开机时。`loginctl enable-linger $USER` 可以让用户服务在开机时就启动（不需要等登录），但这依赖于 systemd 的 `user@.service` 被正确触发。

### 因素 2：网络就绪时序问题

Gateway 服务文件中配置了网络依赖：

```ini
[Unit]
Description=Hermes Agent Gateway
After=network-online.target
Wants=network-online.target
```

`After=network-online.target` 告诉 systemd："等网络就绪后再启动"。但问题在于：

- **系统级**的 `network-online.target` 在网络接口激活后触发
- **用户级**的 `network-online.target` 可能和系统级的行为不同
- 如果 DNS 解析器（systemd-resolved 或 /etc/resolv.conf 配置的 DNS）还没就绪，即使网络接口 UP 了，域名解析仍然会失败

### 因素 3：DNS 间歇性解析失败

Gateway 日志中出现了大量 DNS 解析失败错误：

```
2026-08-14 10:32:02 ERROR [Weixin] poll error (1/3):
  Cannot connect to host ilinkai.weixin.qq.com:443
  ssl:default [Temporary failure in name resolution]
```

`Temporary failure in name resolution` 意味着 DNS 查询超时或 DNS 服务器不可达。这在以下情况会发生：

1. DNS 服务器（如 223.5.5.5）暂时不可达
2. 系统刚启动，DNS 解析器还未完全初始化
3. 网络通过软路由/代理，DNS 配置在启动过程中变化

### 因素 4：非优雅退出导致状态文件过期

Gateway 维护一个 `gateway_state.json` 文件记录运行状态。当 Gateway 非正常退出（如系统强制关机、进程被 kill），这个文件不会被清理，导致状态记录为 "running" 但实际进程已不存在：

```
$ hermes gateway status
⚠ Stale gateway_state.json: recorded state 'running' but the
  recorded process is gone (likely an ungraceful shutdown)
```

这种过期状态可能阻止 systemd 正确判断服务是否需要重启。

### 因素 5：退出码与 Restart 策略不匹配

Gateway 的 service 文件配置了：

```ini
Restart=always
RestartSec=5
RestartForceExitStatus=75
RestartPreventExitStatus=78
```

`Restart=always` 意味着不管什么退出码都应该重启。但实际行为是：Gateway 在收到关闭信号后，执行了优雅关闭流程，然后以**退出码 1** 退出。日志显示：

```
Exiting with code 1 (signal-initiated shutdown without restart request)
so systemd Restart=on-failure can revive the gateway.
```

虽然 `Restart=always` 应该处理退出码 1，但如果 systemd 在服务退出时自身也在关闭过程中（比如系统关机），重启就不会执行。等系统重新启动后，如果用户 systemd 实例没有正确触发服务启动，Gateway 就一直处于 dead 状态。

## 排查过程

### 第一步：确认 Gateway 服务状态

```bash
$ systemctl --user status hermes-gateway
○ hermes-gateway.service - Hermes Agent Gateway
     Active: inactive (dead)

$ hermes gateway status
✗ User gateway service is stopped
  Run: hermes gateway start
⚠ Stale gateway_state.json: recorded state 'running'
  but the recorded process is gone
✓ Systemd linger is enabled (service survives logout)
```

关键发现：服务已停止，状态文件过期，但 linger 已启用。

### 第二步：检查 DNS 连通性

```bash
$ nslookup ilinkai.weixin.qq.com
Server:     223.5.5.5
Address:    223.5.5.5#53
Name:       aewebpodproxy.weixin.qq.com
Address:    36.155.189.158

$ ping -c 2 ilinkai.weixin.qq.com
64 bytes from 36.155.189.158: icmp_seq=1 ttl=51 time=23.6 ms
```

DNS 现在正常——说明之前的解析失败是间歇性的，可能只在系统启动初期出现。

### 第三步：检查 Gateway 启动历史

```bash
$ cat ~/.hermes/gateway-starts.log
# 时间戳转换为可读时间后：
2026-08-06 00:04:20 CST   # 首次安装
2026-08-10 00:02:50 CST   # 上次正常运行
2026-08-17 02:03:00 CST   # 手动重启（今天）
```

关键发现：8月10日到8月17日之间没有任何启动记录！而系统在8月15日重启过：

```bash
$ last reboot | head -3
reboot  system boot  6.12.30+  Sat Aug 15 21:21  still running
```

### 第四步：检查开机后 Gateway 是否尝试启动

```bash
$ journalctl --user -u hermes-gateway 
  --since "2026-08-15 21:00" 
  --until "2026-08-16 03:00"
-- No entries --
```

**关键发现**：系统重启后，Gateway 服务完全没有尝试启动！没有任何日志记录。这意味着 systemd 用户实例可能没有在开机时启动，或者启动了但没有触发 Gateway 服务。

### 第五步：检查用户 systemd 实例

```bash
$ loginctl show-user $USER | grep -i linger
Linger=yes

$ systemctl --user is-enabled hermes-gateway
enabled
```

服务是 enabled 的，但开机后没有启动。这可能是因为：
1. 用户 systemd 实例启动时机太晚
2. `network-online.target` 在用户实例中行为异常
3. 系统关机时 Gateway 退出码导致 systemd 标记了失败状态

## 解决方案

### 步骤 1：手动启动 Gateway（立即恢复）

```bash
# 方法一：使用 hermes CLI
hermes gateway start

# 方法二：使用 systemctl
systemctl --user start hermes-gateway

# 验证
systemctl --user status hermes-gateway
# 应显示 active (running)
```

### 步骤 2：验证微信连接

```bash
# 检查 gateway 日志确认微信已连接
grep -i "weixin.*connect" ~/.hermes/logs/gateway.log | tail -5
# 应看到：[Weixin] Connected account=xxx base=https://ilinkai.weixin.qq.com
```

### 步骤 3：修复 DNS 间歇性失败（/etc/hosts 兜底）

在 DNS 服务器暂时不可达时，可以通过 /etc/hosts 固定绑定微信 API 域名的 IP：

```bash
# 先查询当前 IP
nslookup ilinkai.weixin.qq.com | grep Address | tail -1
# 假设解析到 36.155.189.158

# 添加到 /etc/hosts（需要 sudo）
echo "36.155.189.158 ilinkai.weixin.qq.com" | sudo tee -a /etc/hosts
```

注意：微信的 IP 可能会变化，这种硬编码方式不是长久之计。更好的方案是确保 DNS 解析器在 Gateway 启动前就绪。

### 步骤 4：添加开机自动检测和重启脚本

创建一个 cron job 或 systemd timer 定期检查 Gateway 状态：

```bash
#!/bin/bash
# /home/heron/scripts/check-gateway.sh
if ! systemctl --user is-active hermes-gateway >/dev/null 2>&1; then
    echo "$(date): Gateway is down, starting..."
    systemctl --user start hermes-gateway
    sleep 5
    if systemctl --user is-active hermes-gateway >/dev/null 2>&1; then
        echo "$(date): Gateway started successfully"
    else
        echo "$(date): Gateway failed to start"
    fi
fi
```

```bash
# 添加到 crontab
(crontab -l 2>/dev/null; echo "*/5 * * * * /home/heron/scripts/check-gateway.sh >> ~/.hermes/logs/gateway-check.log 2>&1") | crontab -
```

### 步骤 5：清理过期状态文件

如果 `gateway_state.json` 显示 stale，手动删除让 Gateway 重新创建：

```bash
rm ~/.hermes/gateway_state.json
rm ~/.hermes/state/gateway.lifecycle.json
systemctl --user restart hermes-gateway
```

## 方案对比

### systemd 用户服务开机自启方案对比

| 方案 | 原理 | 优点 | 缺点 | 适用场景 |
|------|------|------|------|----------|
| **loginctl enable-linger** | 让用户 systemd 实例在开机时启动 | 官方推荐、零依赖 | 可能有时序问题 | 大多数场景 |
| **cron 定时检测** | 定期检查并重启服务 | 简单可靠、兜底方案 | 有最多 5 分钟延迟 | 作为补充方案 |
| **系统级服务** | 改用 /etc/systemd/system | 启动时机更早 | 需要 root 权限运行 | 不适合用户级 Agent |
| **systemd timer** | 用 systemd timer 替代 cron | 与 systemd 集成更好 | 配置更复杂 | 推荐方案 |

## 避坑提示

- **`Restart=always` 不等于绝对不会停**：在系统关机过程中，服务退出时 systemd 自身也在关闭，无法执行重启。开机后如果用户 systemd 实例没有正确触发，服务就一直是 dead 状态
- **linger=yes 是必要条件但不是充分条件**：linger 让用户 systemd 实例在开机时启动，但不保证所有服务都能正确启动。网络时序、依赖顺序都可能导致启动失败
- **`journalctl --user` 查不到日志不等于没发生**：如果用户 systemd 实例在某个时间点还没启动，那段时间的日志确实不存在。检查 `journalctl --list-boots` 确认启动时间线
- **DNS 间歇性失败最难排查**：你测试时 DNS 正常，但服务启动那一刻 DNS 可能不可用。如果服务启动失败后不重试，就会永久处于 dead 状态
- **/etc/hosts 绑定 IP 是临时方案**：微信 API 的 IP 可能变化（CDN 负载均衡），长期方案是确保 DNS 解析器在服务启动前就绪
- **`gateway_state.json` 过期会误导排查**：状态文件显示 running 但进程已不存在，检查时要用 `systemctl --user status` 而非读取状态文件
- **`After=network-online.target` 不等于 DNS 就绪**：network-online 只表示网络接口已激活，DNS 解析器可能还没初始化完成。如果你的服务依赖域名解析，需要额外等待或加重试逻辑

## 相关知识

### systemd 用户服务启动流程

```
开机 → systemd (PID 1) 启动
  → 触发 user@1000.service (用户 systemd 实例)
    → 执行 default.target
      → 按依赖顺序启动各服务
        → hermes-gateway.service (After=network-online.target)
```

如果 `user@1000.service` 没有被触发（linger 未启用），用户级服务不会启动。即使用户登录了，某些服务可能因为依赖未满足而跳过启动。

### systemd Restart 策略详解

| 策略 | 行话 |
|------|------|
| `Restart=always` | 任何退出码都重启（除 RestartPreventExitStatus 指定的） |
| `Restart=on-failure` | 仅非零退出码重启 |
| `Restart=on-abnormal` | 信号杀死或超时才重启 |
| `Restart=no` | 不重启（默认） |

关键：`Restart=always` 在**系统关机**时不会执行重启。systemd 在关机时会先停止所有服务，此时重启请求被忽略。

### DNS 解析与网络就绪

`network-online.target` 表示网络接口已激活，但不保证 DNS 解析器已就绪。完整的网络就绪应该包括：
1. 网络接口 UP
2. IP 地址分配（DHCP 或静态）
3. DNS 解析器配置完成（/etc/resolv.conf 或 systemd-resolved）
4. DNS 服务器可达

在软路由/代理环境中，DNS 配置可能在网络接口 UP 之后才通过 DHCP 推送，存在时间窗口。

## 关键命令速查

```bash
# 检查 Gateway 状态
systemctl --user status hermes-gateway
hermes gateway status

# 启动/停止/重启 Gateway
hermes gateway start
systemctl --user start hermes-gateway
systemctl --user restart hermes-gateway

# 检查 linger 状态
loginctl show-user $USER | grep Linger

# 检查服务是否 enabled
systemctl --user is-enabled hermes-gateway

# 查看 Gateway 日志
grep -i "weixin\|error\|connect" ~/.hermes/logs/gateway.log | tail -20
journalctl --user -u hermes-gateway --no-pager | tail -30

# 检查 DNS 解析
nslookup ilinkai.weixin.qq.com
ping -c 2 ilinkai.weixin.qq.com

# 检查系统启动历史
last reboot | head -5
journalctl --list-boots

# 清理过期状态文件
rm ~/.hermes/gateway_state.json ~/.hermes/state/gateway.lifecycle.json
systemctl --user restart hermes-gateway

# DNS 兜底（/etc/hosts 绑定）
echo "36.155.189.158 ilinkai.weixin.qq.com" | sudo tee -a /etc/hosts

# 定时检测脚本（crontab）
(crontab -l 2>/dev/null; echo "*/5 * * * * /home/heron/scripts/check-gateway.sh >> ~/.hermes/logs/gateway-check.log 2>&1") | crontab -
```
