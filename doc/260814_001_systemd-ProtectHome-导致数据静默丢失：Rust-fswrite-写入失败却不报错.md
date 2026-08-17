# systemd ProtectHome 导致数据静默丢失：Rust fs::write 写入失败却不报错

> **日期**: 2026-08-14  
> **分类**: 踩坑记录  
> **标签**: systemd, Rust, Linux, 安全, 数据持久化  
> **来源**: hermes

---

## 背景/问题

你有一个 Rust 写的 Web 服务（比如个人网站后端），用 systemd 管理运行。服务运行时一切正常——API 能正常响应，数据能正常读写。但每次服务重启后（比如 `systemctl restart` 或机器重启），你发现**最近通过 API 新增的数据全丢了**——只剩下很久以前的旧数据。

更诡异的是：API 返回的是成功（HTTP 200），日志里也没有任何错误信息。数据就像凭空消失了一样。

这个问题非常隐蔽，因为它不会产生任何报错——服务正常启动、API 正常响应、写入操作返回成功，但数据实际上从未写入磁盘。你可能会花很长时间怀疑是 API 逻辑有问题、并发竞争导致数据覆盖、或者 JSON 序列化出错，但实际上根因完全在别处。

## 原因分析

这个问题的根因是两个因素叠加造成的：

### 因素 1：systemd 的 ProtectHome 安全沙箱

systemd 提供了一系列安全沙箱指令，其中 `ProtectHome` 有三个值：

| 值 | 效果 | 适用场景 |
|------|------|---------|
| `true` | /home, /root, /run/user 完全不可见（挂载为空） | 不需要访问用户目录的服务 |
| `read-only` | /home, /root, /run/user 只读（可以读但不能写） | 需要读取用户目录文件的服务 |
| `false`（默认） | 不限制 | 无安全要求的服务 |

很多 systemd 服务模板会默认加上 `ProtectHome=read-only` 或 `ProtectSystem=strict` 来做安全加固。这本身是好意——限制服务的文件系统访问范围，即使服务被攻破，攻击者也无法随意写文件。

但如果你的服务需要往 /home 目录下写数据（比如 `/home/user/projects/myapp/data/notes.json`），`ProtectHome=read-only` 就会让所有写入操作静默失败——**注意是"静默"失败，不是报错**。

### 因素 2：Rust 代码用 `let _ =` 忽略了错误

Rust 的 `fs::write()` 返回 `Result<(), io::Error>`。如果你用 `let _ =` 来忽略这个返回值：

```rust
// ❌ 危险写法 — 忽略写入结果
fn save_notes(notes: &[Note]) {
    let json = serialize_notes(notes);
    let _ = fs::write("data/notes.json", json);
    //                                    ↑
    // 如果 ProtectHome=read-only，这里写入失败
    // 但 let _ = 把错误丢弃了，函数正常返回
    // 调用方以为写入成功了
}
```

`let _ =` 是 Rust 中一个合法但危险的模式——它明确告诉编译器"我知道这个表达式有返回值，但我不关心"。编译器不会警告你，因为这是你"有意为之"的。

在正常环境下（没有 ProtectHome 限制），`fs::write` 几乎不会失败，所以这个 bug 可以潜伏很久。但一旦部署到有 systemd 沙箱的环境，写入就开始失败，而代码完全不知道。

### 两个因素叠加的效果

```
API 请求 → 处理逻辑 → 更新内存中的数据 → 调用 save_notes()
                                                    ↓
                                         fs::write() 尝试写入
                                                    ↓
                                         ProtectHome=read-only
                                         阻止写入 → 返回 Err
                                                    ↓
                                         let _ = 丢弃错误
                                                    ↓
                                         函数正常返回 ✓
                                         API 返回 200 OK ✓
                                                    ↓
                                         但磁盘上文件没更新 ✗
                                                    ↓
                                         服务重启 → 内存数据丢失
                                         → 从磁盘加载旧数据
                                         → 新数据彻底消失
```

## 排查过程

### 第一步：确认数据确实丢失

```bash
# 查看当前 API 返回的数据
curl -s http://localhost:8080/api/portfolio | python3 -m json.tool
# 发现只有 6 个项目，缺少了最近新增的 learn-rust

# 检查磁盘上的实际文件
cat /home/heron/workspace/heron-web/data/portfolio.json | python3 -m json.tool
# 磁盘上也只有 6 个 — 数据确实没写入
```

### 第二步：确认服务在运行且 API 可用

```bash
# 服务状态正常
systemctl status heron-web
# → active (running)

# API 能正常响应
curl -s http://localhost:8080/health
# → {"status":"ok"}

# 重新 POST 创建数据
curl -s -X POST http://localhost:8080/api/portfolio \
  -H "X-API-Token: ***REDACTED*** \
  -H "Content-Type: application/json" \
  -d '{"title":"Learn Rust","url":"https://learn-rust.heronwang.cn"}'
# → {"id":7, "status":"created"}  ← 返回成功！
```

### 第三步：重启后验证数据持久化

```bash
# 重启服务
sudo systemctl restart heron-web

# 再次查看数据
curl -s http://localhost:8080/api/portfolio | python3 -m json.tool
# 又只有 6 个了！id=7 又消失了！
```

**关键发现**：API 返回成功，但重启后数据丢失。这说明写入操作"看起来成功了"但实际上没落盘。

### 第四步：检查 systemd 服务配置

```bash
cat /etc/systemd/system/heron-web.service
```

发现关键配置：

```ini
[Service]
ProtectHome=read-only        # ← 这就是元凶！/home 目录只读
ProtectSystem=strict         # ← /usr, /boot, /etc 也只读
# 没有 ReadWritePaths        # ← 没有给 data 目录开白名单
```

### 第五步：检查 Rust 代码的错误处理

```bash
grep -n "fs::write" src/store.rs
```

发现所有 save 函数都用了 `let _ =`：

```rust
let _ = fs::write("data/notes.json", json);      // ← 静默忽略错误
let _ = fs::write("data/portfolio.json", json);   // ← 静默忽略错误
let _ = fs::write("data/guestbook.json", json);   // ← 静默忽略错误
```

### 第六步：手动验证写入是否真的失败

```bash
# 在服务运行时直接写文件测试
echo "test" > /home/heron/workspace/heron-web/data/test.txt
# → bash: test.txt: Read-only file system  ← 确认！
```

## 解决方案

### 步骤 1：给 systemd 服务添加 ReadWritePaths

在 service 文件中添加 `ReadWritePaths` 指令，让 data 目录在沙箱中可写：

```ini
# 修改前
[Service]
Type=simple
ExecStart=/home/heron/myapp/server
ProtectHome=read-only
ProtectSystem=strict

# 修改后
[Service]
Type=simple
ExecStart=/home/heron/myapp/server
ProtectHome=read-only
ProtectSystem=strict
ReadWritePaths=/home/heron/myapp/data    # ← 新增！白名单目录
```

`ReadWritePaths` 的作用是在 `ProtectHome=read-only` 和 `ProtectSystem=strict` 的沙箱中，为指定路径开一个"可写白名单"。服务对这个目录有完整的读写权限，其他受保护目录仍然只读。

```bash
# 修改后重新加载
sudo systemctl daemon-reload
sudo systemctl restart heron-web
```

### 步骤 2：修复 Rust 代码的错误处理

把所有 `let _ = fs::write(...)` 改为正确的错误处理：

```rust
// ❌ 修改前 — 静默忽略写入错误
fn save_notes(notes: &[Note]) {
    let json = serialize_notes(notes);
    let _ = fs::write("data/notes.json", json);
}

// ✅ 修改后 — 输出错误日志
fn save_notes(notes: &[Note]) {
    let json = serialize_notes(notes);
    if let Err(e) = fs::write("data/notes.json", json) {
        eprintln!("ERROR: Failed to write notes.json: {}", e);
    }
}
```

`if let Err(e) = fs::write(...)` 会捕获写入失败并通过 `eprintln!` 输出到 stderr，systemd 会将 stderr 记录到 journalctl 日志中。这样即使写入失败，也能在日志中看到原因。

### 步骤 3：验证修复效果

```bash
# 1. 推送测试数据
curl -s -X POST http://localhost:8080/api/notes \
  -H "X-API-Token: ***REDACTED*** \
  -H "Content-Type: application/json" \
  -d '{"title":"persist_test","content":"verify"}'

# 2. 检查磁盘文件是否更新
stat /home/heron/myapp/data/notes.json
# → Modify: 2026-08-09 ... (时间戳应刚刚更新)

# 3. 重启服务
sudo systemctl restart heron-web

# 4. 验证数据还在
curl -s http://localhost:8080/api/notes | grep "persist_test"
# → 应该能找到，数据持久化成功！

# 5. 检查日志中是否有写入错误
journalctl -u heron-web --since "5 min ago" | grep ERROR
# → 应该没有 ERROR 日志
```

### 步骤 4（可选）：编写自动化验证脚本

写一个 Python 脚本自动验证持久化：

```python
import urllib.request, json, os, time

BASE = "http://localhost:8080"
DATA_DIR = "/home/heron/myapp/data"
TOKEN = ***REDACTED***

# 记录推送前的文件时间戳
before = os.path.getmtime(f"{DATA_DIR}/notes.json")

# 推送测试数据
req = urllib.request.Request(
    f"{BASE}/api/notes",
    data=json.dumps({"title": "verify", "content": "test"}).encode(),
    headers={"X-API-Token": TOKEN, "Content-Type": "application/json"},
    method="POST"
)
result = json.loads(urllib.request.urlopen(req).read())
test_id = result["id"]

time.sleep(0.5)  # 等待 fs::write 完成

# 检查文件时间戳是否更新
after = os.path.getmtime(f"{DATA_DIR}/notes.json")
if after > before:
    print(f"PASS: 文件已更新 (id={test_id})")
else:
    print(f"FAIL: 文件未更新 — 写入失败！")

# 清理测试数据
del_req = urllib.request.Request(
    f"{BASE}/api/notes/{test_id}",
    headers={"X-API-Token": TOKEN},
    method="DELETE"
)
urllib.request.urlopen(del_req)
```

## 方案对比

### systemd 安全沙箱指令对比

| 指令 | 效果 | 写入 /home | 写入 /tmp | 适用场景 |
|------|------|-----------|----------|---------|
| 无沙箱 | 无限制 | ✅ | ✅ | 无安全要求 |
| `ProtectHome=read-only` | /home 只读 | ❌ | ✅ | 需要读取用户文件 |
| `ProtectHome=true` | /home 不可见 | ❌ | ✅ | 不需要用户文件 |
| `ProtectSystem=strict` | /usr, /boot 只读 | ✅ | ❌ | 系统文件保护 |
| `ProtectHome=read-only` + `ReadWritePaths=/path` | /home 只读，白名单可写 | ⚠️ 仅白名单 | ✅ | ✅ **推荐** |

### Rust 错误处理模式对比

| 模式 | 写法 | 错误可见性 | 推荐度 |
|------|------|-----------|--------|
| 忽略错误 | `let _ = fs::write(...)` | 完全不可见 | ❌ 危险 |
| 打印日志 | `if let Err(e) = fs::write(...) { eprintln!(...) }` | journalctl 可见 | ✅ 推荐 |
| 返回 Result | `fn save() -> Result<(), io::Error>` | 调用方决定 | ✅ 最佳 |
| unwrap | `fs::write(...).unwrap()` | panic 崩溃 | ⚠️ 仅调试 |

## 避坑提示

- **`let _ =` 是 Rust 中的"沉默杀手"**：它合法地忽略任何返回值，编译器不警告。对于 I/O 操作（文件写入、网络请求），绝对不要用 `let _ =`，因为 I/O 失败是常态而非异常
- **`ProtectHome=read-only` 不报错而是静默失败**：写入操作返回 `Err(PERMISSION_DENIED)`，但如果你不检查返回值，就完全不知道。这与 `ProtectHome=true`（目录完全不可见）不同——只读模式下目录存在且可读，只是不能写，更具迷惑性
- **`ReadWritePaths` 必须是绝对路径**：不能写相对路径，必须写完整的 `/home/user/myapp/data`
- **修改 service 文件后必须 `daemon-reload`**：systemd 缓存了旧的 service 配置，不 reload 就不生效
- **验证持久化要"推数据 → 重启 → 查数据"三步走**：只推数据不重启，可能只是内存更新了没落盘；只查文件不重启，可能文件有但服务加载的是旧版本
- **`eprintln!` 比 `println!` 更适合错误日志**：stderr 在 systemd 中会被单独标记为错误级别，`journalctl` 中更容易筛选
- **不要用 `ReadWritePaths=/` 给整个根目录开权限**：这等于完全放弃沙箱保护。只给真正需要写的目录开白名单

## 相关知识

### systemd 安全沙箱体系

systemd 提供了丰富的安全沙箱指令，分为几个层次：

**文件系统隔离：**
- `ProtectSystem=`：保护 /usr, /boot, /etc
- `ProtectHome=`：保护 /home, /root, /run/user
- `ReadWritePaths=`：在保护范围内开白名单
- `ReadOnlyPaths=`：强制只读
- `InaccessiblePaths=`：完全隐藏
- `TemporaryFileSystem=`：挂载临时空文件系统
- `PrivateTmp=`：给 /tmp 和 /var/tmp 做独立挂载

**权限隔离：**
- `User=`/`Group=`：以非 root 用户运行
- `NoNewPrivileges=true`：禁止提权
- `CapabilityBoundingSet=`：限制 Linux capabilities
- `SystemCallFilter=`：限制可用的系统调用
- `RestrictAddressFamilies=`：限制网络协议

**这些指令组合使用可以构建很强的安全边界，但每个限制都可能导致服务功能异常。关键是：加限制的同时要配好白名单（ReadWritePaths），并且代码层面要做好错误处理。**

### Rust 的错误处理哲学

Rust 没有 exception 机制，所有错误都通过 `Result<T, E>` 类型返回。这意味着错误是"显式"的——你必须在代码中决定是处理它还是传播它。

但 Rust 也提供了"逃避"机制：
- `let _ = expr`：丢弃返回值（包括 Result）
- `.unwrap()`：失败时 panic
- `.expect("msg")`：失败时 panic 带消息
- `?` 运算符：传播错误给调用方

其中 `let _ =` 是最危险的，因为它既不 panic 也不传播——错误被完全吞掉，就像从来没发生过。对于 I/O 操作，这是定时炸弹。

## 关键命令速查

### 检查 systemd 沙箱配置

```bash
# 查看服务完整配置（含沙箱指令）
systemctl cat myapp.service

# 查看运行时沙箱状态
systemctl show myapp.service | grep -E "ProtectHome|ProtectSystem|ReadWritePaths"

# 查看服务文件
cat /etc/systemd/system/myapp.service
```

### 修改沙箱配置

```bash
# 编辑 service 文件
sudo nano /etc/systemd/system/myapp.service

# 添加 ReadWritePaths 后重新加载
sudo systemctl daemon-reload
sudo systemctl restart myapp
```

### 验证写入权限

```bash
# 手动测试目录是否可写
sudo -u myapp-user touch /home/user/myapp/data/test.txt

# 检查文件权限
ls -la /home/user/myapp/data/

# 查看进程的文件系统视图
ls -l /proc/$(pgrep myapp)/root/home/user/myapp/data/
```

### Rust 错误处理模式

```rust
// ❌ 危险：忽略错误
let _ = fs::write(path, data);

// ✅ 安全：打印错误日志
if let Err(e) = fs::write(path, data) {
    eprintln!("ERROR: write {} failed: {}", path, e);
}

// ✅ 最佳：返回 Result 给调用方
fn save(data: &str) -> Result<(), std::io::Error> {
    fs::write("data.json", data)?;
    Ok(())
}

// ✅ 调用方处理错误
match save(data) {
    Ok(()) => log::info!("saved successfully"),
    Err(e) => log::error!("save failed: {}", e),
}
```

### 持久化验证三步法

```bash
# 1. 推数据
curl -s -X POST http://localhost:8080/api/notes \
  -H "X-API-Token: ***REDACTED*** \
  -H "Content-Type: application/json" \
  -d '{"title":"test","content":"verify"}'

# 2. 重启服务
sudo systemctl restart myapp

# 3. 查数据
curl -s http://localhost:8080/api/notes | grep "test"
# 能找到 = 持久化成功 ✅
# 找不到 = 写入失败 ❌ → 查 journalctl -u myapp | grep ERROR
```
