# journald 日志配置导致 Gateway INFO 日志丢失

## 背景

排查 Hermes Gateway 微信连接问题时，发现 `journalctl --user -u hermes-gateway` 没有任何输出（`-- No entries --`），而 `~/.hermes/logs/gateway.log` 停留在 8 月 24 日（上次关机时间）。Gateway 明明在运行，但 INFO 级别日志全部丢失。

## 根因

### journald 配置

```bash
grep -v "^#" /etc/systemd/journald.conf | grep -v "^$"
```

```ini
[Journal]
Storage=persistent
SystemMaxUse=50M
MaxLevelStore=err
MaxLevelSyslog=warning
```

关键配置 `MaxLevelStore=err`：**journald 只存储 ERROR 及以上级别的日志**，WARNING 和 INFO 日志被直接丢弃。

### Hermes Gateway 日志路径

Hermes Gateway 的 systemd 服务配置：

```ini
[Service]
Type=simple
ExecStart=/home/heron/.hermes/hermes-agent/venv/bin/python -m hermes_cli.main gateway run
Restart=always
StandardOutput=journal
StandardError=journal
```

- stdout/stderr → systemd journal
- 文件日志（`~/.hermes/logs/gateway.log`）只在**前台运行**时写入
- systemd 管理时，文件日志不再追加

### 日志层次

```
应用日志
  ├── INFO  → journal (被 MaxLevelStore=err 丢弃) ❌
  ├── WARNING → journal (被 MaxLevelStore=err 丢弃) ❌
  ├── ERROR → journal (存储) ✅ → ~/.hermes/logs/errors.log (也存储) ✅
  └── 文件日志
       ├── gateway.log → 仅前台运行时写入，systemd 模式不更新
       └── errors.log → WARNING 及以上，始终写入
```

## 解决方案

### 查看日志的正确方式

```bash
# 1. 查 errors.log（最可靠，包含所有 WARNING+）
grep "2026-09-01" ~/.hermes/logs/errors.log | tail -20

# 2. 查 gateway.log（仅前台模式有效）
tail -20 ~/.hermes/logs/gateway.log

# 3. 查 journal（仅 ERROR 级别，因 MaxLevelStore=err）
journalctl --user -u hermes-gateway --since "today" --no-pager
```

### 修改 journald 配置（可选）

如果需要 journal 存储 WARNING 级别日志：

```bash
sudo sed -i 's/MaxLevelStore=err/MaxLevelStore=warning/' /etc/systemd/journald.conf
sudo systemctl restart systemd-journald
```

但注意：这会增加 journal 磁盘使用量。当前 `SystemMaxUse=50M` 的限制仍然生效。

## 当前配置的原因

`MaxLevelStore=err` 是之前为了减少 NAS 磁盘噪音而做的优化：

- journald 写入 `/var/log/journal/`（系统盘 mmcblk0）
- 减少日志写入量 = 减少磁盘 I/O = 减少噪音
- 50M 容量限制防止日志撑满系统盘

这是合理的取舍：WARNING 日志仍然写入 `~/.hermes/logs/errors.log`（NVMe 分区），只是不存储到 journal 中。

## Hermes 日志文件结构

```
~/.hermes/logs/
  ├── agent.log        # Agent 主进程日志
  ├── agent.log.1      # 轮转的旧日志
  ├── errors.log       # WARNING 及以上（所有组件）
  ├── gateway.log      # Gateway 日志（仅前台模式）
  ├── gateway-exit-diag.log  # Gateway 退出诊断
  ├── gateway-shutdown-diag.log  # Gateway 关机诊断
  ├── gateway_faulthandler.log  # 崩溃诊断
  └── gui.log          # GUI 日志
```

注意：`~/.hermes/logs/` 已通过符号链接迁移到 NVMe（`/volume2/hermes-data/logs/`），减少机械盘写入。

## 经验总结

| 场景 | 正确做法 |
|------|---------|
| 查 ERROR | `journalctl --user -u hermes-gateway` 或 `errors.log` |
| 查 WARNING | `grep` `~/.hermes/logs/errors.log`（journal 丢了） |
| 查 INFO | 不可能（journal 被 MaxLevelStore 丢弃，无文件写入） |
| 查 gateway 运行状态 | `systemctl --user status hermes-gateway` |
| 查 gateway 退出历史 | `~/.hermes/logs/gateway-exit-diag.log` |

## 坑点

1. **`-- No entries --` 不代表服务没运行**：journal 没有条目是因为日志级别被过滤，不是服务没输出
2. **gateway.log 停留在旧日期**：systemd 模式下文件日志不更新，不代表 gateway 没运行
3. **errors.log 是最可靠的日志源**：始终写入，不受 journald 配置影响，包含所有 WARNING+
4. **修改 journald 配置需权衡**：放开日志级别会增加磁盘写入，在 NAS 场景需考虑噪音问题

---

*来源：2026-09-01 Hermes Gateway 微信连接排查，发现 journald MaxLevelStore=err 导致 INFO 日志丢失*
