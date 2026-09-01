# SQLite 3.40.1 WAL-reset 损坏 Bug 与 Hermes 自动降级

## 背景

在排查 Hermes Gateway 启动日志时，发现多条关于 SQLite WAL-reset corruption bug 的警告。该 Bug 影响系统中多个数据库文件，Hermes 自动采取了降级措施。

## 问题详情

### 警告信息

Hermes 启动时输出的典型警告：

```
WARNING hermes_state: state.db: linked SQLite 3.40.1 is vulnerable to the
WAL-reset corruption bug (https://sqlite.org/wal.html#walresetbug) — using
journal_mode=DELETE instead of enabling WAL. Upgrade to SQLite 3.51.3+
(or backports 3.50.7 / 3.44.6); Hermes-managed installs can repair the
embedded runtime with `hermes update`. See `hermes doctor`.
This warning fires once per process per database.
```

### 受影响的数据库

| 数据库 | 路径 | 用途 |
|--------|------|------|
| state.db | `/volume2/hermes-data/state.db` | 会话存储（SQLite + FTS5） |
| async_delegation | state.db 内部表 | 异步子代理任务 |
| delivery_ledger | state.db 内部表 | 消息投递记录 |
| cron/executions.db | cron 目录下 | 定时任务执行记录 |
| kanban.db | kanban 目录下 | 多代理看板 |

每个数据库在进程启动时触发一次警告（`fires once per process per database`）。

### Bug 原理

SQLite 3.40.1 存在 [WAL-reset corruption bug](https://sqlite.org/wal.html#walresetbug)：

- **WAL 模式**（Write-Ahead Logging）是 SQLite 的高性能并发写入模式
- 在特定版本的 SQLite 中，WAL 文件重置时可能导致数据库损坏
- 受影响版本：SQLite 3.40.1（NAS Debian 12 自带版本）
- 修复版本：SQLite 3.51.3+，或向后移植版本 3.50.7 / 3.44.6

### Hermes 的自动降级

Hermes 检测到 SQLite 版本存在此 Bug 后，自动将 `journal_mode` 从 WAL 降级为 `DELETE`：

- **WAL 模式**：读写并发性能好，但存在损坏风险
- **DELETE 模式**：传统回滚日志模式，更安全但并发性能略低

这意味着 Hermes 在不修复 SQLite 的情况下仍能正常运行，只是数据库 I/O 性能可能略有降低。

## 解决方案

### 方案一：升级 SQLite（推荐但需系统级操作）

```bash
# 检查当前 SQLite 版本
sqlite3 --version
# 3.40.1 2022-12-28 ...

# Hermes 内置的 Python SQLite 版本
python3 -c "import sqlite3; print(sqlite3.sqlite_version)"
# 3.40.1
```

升级系统 SQLite 需要从源码编译或等待 Debian 更新，对 NAS 系统可能有风险。

### 方案二：通过 Hermes 更新修复

```bash
# Hermes 管理的 Python 运行时可能包含更新的 SQLite
hermes update

# 更新后检查
hermes doctor
```

### 方案三：保持现状（当前选择）

Hermes 的自动降级机制已经生效，使用 `journal_mode=DELETE` 避免了损坏风险。对于单用户 NAS 场景，DELETE 模式的性能影响可以忽略。

## 验证当前状态

```bash
# 检查 state.db 的 journal_mode
sqlite3 /volume2/hermes-data/state.db "PRAGMA journal_mode;"
# 应输出: delete

# 检查数据库完整性
sqlite3 /volume2/hermes-data/state.db "PRAGMA integrity_check;"
# 应输出: ok
```

## 经验总结

1. **Hermes 有完善的数据库安全检测**：自动检测 SQLite 版本 Bug 并降级 journal_mode，无需用户干预
2. **警告不是错误**：这些 WARNING 不会阻止 Hermes 运行，只是提示用户可以升级以获得更好性能
3. **多数据库统一管理**：Hermes 对所有打开的 SQLite 数据库都执行版本检查，确保一致性
4. **符号链接不影响检测**：state.db 通过符号链接指向 NVMe 分区（`/volume2/hermes-data/state.db`），Hermes 能正确解析实际路径并检测
5. **长期建议**：在合适时机运行 `hermes update` 升级内置运行时，消除所有 WAL-reset 警告
