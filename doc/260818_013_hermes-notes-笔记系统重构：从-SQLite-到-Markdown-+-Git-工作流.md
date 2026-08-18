# hermes-notes 笔记系统重构：从 SQLite 到 Markdown + Git 工作流

> **日期**: 2026-08-18  
> **分类**: 架构设计  
> **标签**: Rust, SQLite, Markdown, Git, cron, 热重载, heron-web  
> **来源**: hermes

---

## 背景/问题

原笔记系统将技术笔记存储在主站 SQLite 数据库中，通过 API 写入。存在以下问题：
- 笔记没有版本管理（数据库单点）
- 无法离线编辑
- 备份依赖数据库导出
- cron 推送笔记时需调 API，依赖主站在线

## 架构重构

### 修改前 vs 修改后

| 项目 | 修改前 | 修改后 |
|------|--------|--------|
| **存储位置** | 主站 SQLite 数据库 | `~/workspace/hermes-notes/doc/` 目录（.md 文件） |
| **推送方式** | API 调用写入数据库 | 写 .md 文件 + git push 到 GitHub |
| **主站更新** | 直接写数据库 | 热重载自动检测文件数变化 |
| **备份** | 仅数据库 | GitHub 仓库（版本管理） |
| **微信通知** | 推送完整笔记内容 | 简要汇报（标题+篇数） |

### 新工作流

```
23:30 cron 触发
  ↓
session_search 搜索当天对话
  ↓
提取有价值内容（不限主题）
  ↓
每条经验写成独立 .md 文件
  → ~/workspace/hermes-notes/doc/YYMMDD_NNN_标题.md
  ↓
git add + commit + push 到 GitHub
  ↓
主站热重载检测到文件数变化 → 自动重新加载
  ↓
微信通知推送结果（标题+篇数）
```

### hermes-notes 仓库结构

```
~/workspace/hermes-notes/
├── doc/                    # 笔记目录（主站监控此目录）
│   ├── 260309_001_重要密码笔记.md
│   ├── 260806_001_Docker-部署踩坑记录.md
│   ├── ...
│   └── 260818_013_*.md     # 最新笔记
├── .gitignore
└── README.md
```

GitHub 仓库：`Heron-Wang/hermes-notes`

### Rust 主站热重载机制

heron-web 的 `store.rs` 实现了基于文件计数的热重载：

```rust
// store.rs
pub struct Store {
    notes_file_count: Arc<Mutex<usize>>,  // 文件计数器
    // ...
}

impl Store {
    fn notes_doc_dir(&self) -> PathBuf {
        PathBuf::from("../hermes-notes/doc")  // 相对路径
    }
    
    fn load_notes(&self) {
        let dir = self.notes_doc_dir();
        // 读取目录下所有 .md 文件
        // 解析每篇笔记的元信息+正文
        // 更新 notes_file_count
    }
    
    /// 热重载校验：比较当前文件数与上次记录
    pub fn check_reload_notes(&self) {
        let dir = self.notes_doc_dir();
        let current = /* 读取目录文件数 */;
        let last = *self.notes_file_count.lock().unwrap();
        if current != last {
            self.load_notes();  // 文件数变化 → 重新加载
        }
    }
}
```

每次 API 请求时调用 `check_reload_notes()`，检测到 `doc/` 目录文件数量变化后自动重新加载所有笔记，**无需重启服务**。

### 文件命名规范

```
YYMMDD_NNN_标题.md
```

- `YYMMDD`：日期，如 `260818`
- `NNN`：当天序号，从 `001` 开始
- `标题`：中文标题，特殊字符用连字符替换
- 示例：`260818_009_HTML5-Video-音频资源释放Bug.md`

### Markdown 文件格式

```markdown
# 标题

> **日期**: YYYY-MM-DD  
> **分类**: 踩坑记录/技术知识/架构设计  
> **标签**: tag1, tag2, tag3  
> **来源**: hermes

---

## 背景/问题
...

## 关键操作
...

## 结果/经验
...

## 避坑提示
...
```

敏感信息（密码/Token/私钥）用 `***REDACTED***` 替换。

## cron 任务配置

```yaml
job_id: 46eefe83a590
name: daily-notes-summary
schedule: "30 23 * * *"    # 每天 23:30
deliver: origin             # 推送到微信
skill: hermes-agent
```

cron prompt 核心要求：
1. `session_search` 搜索当天对话
2. 提取有价值内容（不限主题）
3. 每条写成独立 .md 文件
4. git add + commit + push
5. 主站热重载自动更新
6. 微信汇报（标题+篇数+总字数）

## 避坑提示

- **热重载只检测文件数量变化**：如果修改已有笔记内容但不增减文件数，不会触发热重载。需要 touch 一个新文件或重启服务
- **文件名中的特殊字符**：冒号 `:` 在 Windows 上非法，管道符 `|` 在 shell 中有歧义。统一用连字符 `-` 替换
- **git push 需要 SSH 密钥或 token**：NAS 上已配置 SSH 密钥推送到 GitHub，无需每次输入密码
- **`NOTES_DOC_DIR` 相对路径**：`../hermes-notes/doc` 是相对于 heron-web 编译时的 cwd。如果服务 cwd 变化，路径会断裂
- **主站笔记来源标记**：笔记 JSON 中 `source: "hermes"` 字段，前端展示 "via Hermes" 标签
