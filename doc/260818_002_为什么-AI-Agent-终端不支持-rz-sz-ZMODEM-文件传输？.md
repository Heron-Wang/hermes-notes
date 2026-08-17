# 为什么 AI Agent 终端不支持 rz/sz (ZMODEM) 文件传输？

> **日期**: 2026-08-18  
> **分类**: 技术知识  
> **标签**: Hermes, ZMODEM, rz, sz, PTY, 终端, 文件传输  
> **来源**: hermes

---

## 背景/问题

在使用 Hermes Agent（或类似 AI Agent 终端如 Claude Code、Codex）时，用户尝试输入 `rz -E` 想从本地传文件到 NAS，但命令直接卡住或无反应。`rz`/`sz` 是经典的 ZMODEM 协议文件传输工具，在 SSH 终端（如 iTerm2、xshell、SecureCRT）中广泛使用。为什么在 AI Agent 的终端里就不行了？

## 原因分析

### ZMODEM 协议的工作原理

ZMODEM 是一种**终端级**的文件传输协议，它依赖于终端模拟器（iTerm2、xterm 等）和远程 shell 之间的**双向字节流通道**：

```
本地终端模拟器 ←→ SSH 通道 ←→ 远程 shell (rz/sz)
```

传输流程：
1. 用户在远程 shell 输入 `rz`
2. `rz` 通过 stdout 发送 ZMODEM 起始帧：`**\x18B00000000...`
3. **终端模拟器拦截**这个转义序列（不显示在屏幕上）
4. 终端模拟器弹出文件选择对话框
5. 用户选择文件后，终端模拟器将文件二进制数据通过**同一条通道**的 stdin 发回给 `rz`
6. `rz` 接收数据，写入磁盘

关键点：**终端模拟器必须主动参与 ZMODEM 协议**。它需要识别 ZMODEM 起始帧、拦截后续输出、并通过 stdin 发送文件数据。

### Hermes Agent 终端的架构

Hermes 的终端工具执行命令有两条路径：

| 路径 | 实现 | 输出处理 |
|------|------|---------|
| 默认（管道模式） | `subprocess.Popen` + stdout/stderr PIPE | 后台 reader 线程读取输出到缓冲区 |
| PTY 模式（`pty=true`） | `ptyprocess.PtyProcess.spawn` | 后台 reader 线程读取 PTY 输出到缓冲区 |

两种模式的 reader 线程都会**把所有输出当作文本缓冲**。ZMODEM 的二进制转义序列（`**\x18B...`）被 reader 线程当作普通输出吃掉了，用户的终端模拟器永远看不到 ZMODEM 起始帧，所以不会弹出文件选择框。`rz` 一直等待终端模拟器回传文件数据，但永远等不到，最终超时。

这不是配置问题，是**架构层面的不兼容**：

```
用户终端 ←→ Hermes Agent (reader 线程吃掉 ZMODEM 序列) ←→ subprocess (rz)
```

中间多了一层"读者"拦截了 ZMODEM 协议所需的直接通道。

### 为什么 SSH 终端可以

在传统 SSH 场景中：

```
用户终端模拟器 ←→ SSH 隧道 ←→ 远程 shell (rz)
```

SSH 隧道是一个**透明双向管道**，不解析也不拦截内容。终端模拟器直接和 rz 通信，ZMODEM 协议正常工作。

## 替代方案

在 AI Agent 终端环境中传文件的替代方式：

| 方案 | 操作 | 适用场景 |
|------|------|---------|
| **直接路径** | 告诉 Agent 文件路径，用 `read_file` 读取 | 文件已在服务器上 |
| **scp/sftp** | 在 Agent 之外用 `scp user@host:/path .` 下载 | SSH 可达的环境 |
| **base64** | Agent 执行 `base64 file.ext`，用户解码 | 小文件（<1MB） |
| **HTTP 下载** | Agent 把文件放到 web 目录，用户浏览器下载 | 有 web 服务器 |
| **剪贴板粘贴** | 直接粘贴文件内容到对话 | 纯文本文件 |

## 避坑提示

- **不要尝试在 Agent 终端里用 rz/sz**：无论是否加 `-E` 参数，都不会工作。reader 线程会吃掉 ZMODEM 协议序列
- **PTY 模式也不能解决**：即使 `pty=true`，reader 线程仍然会拦截 PTY 输出中的 ZMODEM 序列
- **lrzsz 已安装但无法使用**：`which rz sz` 可能显示 `/usr/bin/rz /usr/bin/sz`，但协议层不兼容
- **不要修改 Agent 的 reader 线程来支持 ZMODEM**：这需要在 reader 中实现完整的 ZMODEM 协议解析 + 文件选择 UI，复杂度极高且不通用

## 相关知识

### ZMODEM vs 其他文件传输协议

| 协议 | 传输方式 | 依赖终端模拟器 | 速度 | 可靠性 |
|------|---------|---------------|------|--------|
| **ZMODEM** | 终端级（转义序列） | 是（必须支持 ZMODEM） | 快 | 高（自动重传） |
| **XMODEM** | 终端级（转义序列） | 是 | 慢 | 中 |
| **YMODEM** | 终端级（转义序列） | 是 | 中 | 中 |
| **SFTP** | 独立通道（SSH 子系统） | 否 | 快 | 高 |
| **SCP** | 独立通道（SSH） | 否 | 快 | 高 |

ZMODEM 是唯一依赖终端模拟器主动参与的协议，这也是它在 Agent 终端中无法工作的根本原因。

### 哪些终端模拟器支持 ZMODEM

| 终端 | ZMODEM 支持 | 备注 |
|------|------------|------|
| iTerm2 (macOS) | 原生支持 | 最常用 |
| xshell | 原生支持 | Windows |
| SecureCRT | 原生支持 | 跨平台 |
| Terminator | 需插件 | |
| tmux | 不支持 | 多路复用器会破坏 ZMODEM 通道 |
| screen | 不支持 | 同上 |
| GNOME Terminal | 不支持 | |
| Windows Terminal | 不支持 | |
