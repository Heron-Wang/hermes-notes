# systemd ProtectHome 导致数据静默丢失 — API 返回成功但文件没写盘

> **日期**: 2026-08-09  
> **分类**: 踩坑记录  
> **标签**: systemd, Rust, Linux, 运维  
> **来源**: hermes

---

## 背景/问题

一个 Rust 写的 Web 服务部署在 NAS 上，通过 systemd 管理。用户通过 API 创建笔记、添加作品集项目后，API 返回成功，但**服务重启后数据就丢了**——只剩下很早以前的旧数据。

## 原因分析

这个问题有两层原因叠加，缺一不可：

### 原因1：systemd 的 ProtectHome=read-only

服务的 systemd 配置文件里有：

```ini
[Service]
ProtectHome=read-only
```

这个安全选项把 `/home` 目录挂载为只读。服务运行时，代码里所有对 `/home` 下文件的写入操作都会**静默失败**（返回 permission denied）。

### 原因2：Rust 代码用 `let _ =` 吞掉了写入错误

数据持久化代码这样写的：

```rust
let _ = fs::write(&path, &json_content);
```

`let _ =` 的作用是"我知道这个函数返回 Result，但我不关心它成功还是失败，把返回值丢掉"。这意味着 `fs::write` 失败时，程序不会报错、不会 panic、不会日志，完全静默。

两层叠加的效果：
1. systemd 让文件写入失败 → 返回 Err
2. 代码忽略了 Err → API 认为写入成功，返回 200 OK
3. 内存中的数据是对的（API 能读到），但磁盘上没写入
4. 服务重启 → 内存数据丢失 → 只剩磁盘上的旧数据

## 解决方案

### 修复1：systemd service 加 ReadWritePaths

```ini
[Service]
ProtectHome=read-only
# 在只读沙箱中打开 data 目录的写入权限
ReadWritePaths=/path/to/your/data
```

这样既保留了 ProtectHome 的安全隔离，又让数据目录可写。

### 修复2：不要吞掉 IO 错误

把所有 `let _ = fs::write(...)` 改为：

```rust
if let Err(e) = fs::write(&path, &json_content) {
    eprintln!("写入失败: path={}, error={}", path.display(), e);
}
```

这样以后写入失败会在 journalctl 日志中可见，不再静默吞错。

### 验证方法

推送数据后检查磁盘文件的实际修改时间：

```bash
# 推送前记录时间戳
stat /path/to/data/notes.json | grep Modify

# 通过 API 推送一条笔记
curl -X POST http://localhost:8080/api/notes \
  -H "Content-Type: application/json" \
  -d '{"title":"test","content":"test"}'

# 推送后再看时间戳 - 应该更新了
stat /path/to/data/notes.json | grep Modify
```

如果时间戳没变，说明写入还是失败了。

## 避坑提示

### `let _ =` 是 Rust 中的反模式

`let _ =` 会丢弃任何返回值，包括 `Result`。这在 Rust 社区被认为是反模式（anti-pattern），因为它会让错误静默消失。正确做法：
- 想忽略错误但记录日志：用 `if let Err(e) = ...`
- 想在测试中忽略：用 `let _ = ...` 只在 `#[cfg(test)]` 里用
- 想完全处理：用 `match` 或 `?` 传播错误

### systemd 安全沙箱的常见限制

| 选项 | 效果 | 如需写入怎么办 |
|------|------|--------------|
| `ProtectHome=read-only` | /home 只读 | 加 `ReadWritePaths=/home/xxx/data` |
| `ProtectSystem=strict` | /usr, /boot, /etc 只读 | 同上 |
| `ReadOnlyPaths=` | 指定路径只读 | 用 `ReadWritePaths` 覆盖 |
| `PrivateTmp=true` | /tmp 独立 | 不影响业务目录 |

**原则：安全沙箱要配，但一定要同时配 ReadWritePaths 指向真正需要写入的目录。**

### 静默失败的通用教训

"API 返回成功"不等于"数据持久化了"。涉及文件写入的服务，一定要验证磁盘文件的实际状态，不能只看 API 返回值。
