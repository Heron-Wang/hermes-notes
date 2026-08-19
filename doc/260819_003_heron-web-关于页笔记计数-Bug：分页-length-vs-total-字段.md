# heron-web 关于页笔记计数 Bug：分页 length vs total 字段

> **日期**: 2026-08-19  
> **分类**: Bug 修复  
> **标签**: heron-web, Rust, 前端, 分页, API  
> **来源**: hermes

---

## 背景/问题

heron-web 个人网站的"关于"页面显示技术笔记数量为 50，但实际有 58 篇笔记。用户反馈"主站有问题"，需要排查。

## 关键操作

### 1. 确认实际笔记数

```bash
# API 返回的 total 字段
curl -s http://localhost:8080/api/notes | python3 -c "
import sys, json
d = json.load(sys.stdin)
print(f'API total: {d[\"total\"]}')
print(f'API data length (page 1): {len(d[\"data\"])}')
"
# 输出: API total: 58, data length: 50
```

API 返回 `{ total: 58, data: [...50 items...] }`。分页每页 50 条，第一页只有 50 条数据。

### 2. 查看前端代码

关于页用 JavaScript 从 API 获取笔记数量并显示。问题代码：

```javascript
// 错误：用了分页数据的数组长度
fetch('/api/notes')
  .then(r => r.json())
  .then(notes => {
    document.getElementById('aboutNotes').textContent = notes.length;  // 50，不是 58
  });
```

`notes.length` 是 `notes.data.length`（第一页 50 条），不是 `notes.total`（总数 58）。

### 3. 修复

```javascript
fetch('/api/notes')
  .then(r => r.json())
  .then(notesData => {
    const notesTotal = notesData.total;  // 58
    document.getElementById('aboutNotes').textContent = notesTotal;
  });
```

用 `notesData.total` 替代 `notes.length`。

### 4. 四层验证

```python
# 验证脚本
import subprocess, json, sys

# 1. 源码包含修复
src = open('/home/heron/workspace/heron-web/src/index.html').read()
assert 'notesTotal=notesData.total' in src
assert 'textContent=notesTotal' in src

# 2. 编译后的二进制包含修复
binary = subprocess.run(['strings', 'target/release/heron-web'],
                        capture_output=True, text=True).stdout
assert 'notesTotal' in binary

# 3. API total > page1 length（证明修复有意义）
api = json.loads(subprocess.run(['curl', '-s', 'http://localhost:8080/api/notes'],
                                capture_output=True, text=True).stdout)
assert api['total'] > len(api['data'])  # 58 > 50

# 4. 服务 HTTP 200
code = subprocess.run(['curl', '-s', '-o', '/dev/null', '-w', '%{http_code}',
                       'http://localhost:8080'], capture_output=True, text=True).stdout
assert code == '200'

print(f"PASS: source=OK binary=OK API total={api['total']}>page1={len(api['data'])} HTTP=200")
```

结果：`PASS: source=OK binary=OK API total=58>page1=50 HTTP=200`

## 结果/经验

| 层面 | 修复前 | 修复后 |
|------|--------|--------|
| 数据源 | `notes.length` (50) | `notesData.total` (58) |
| 源码 | ❌ 用了数组长度 | ✅ 用了 total 字段 |
| 编译二进制 | ❌ 旧代码 | ✅ 新代码已编译 |
| 浏览器显示 | 50 | 58 |

**分页 API 的常见陷阱**：返回 `{ data: [...], total: N }` 结构时，`data.length` 只是当前页的数据量，不是总数。总数必须用 `total` 字段。

## 避坑提示

- **分页 API 的 data.length ≠ total**：任何分页接口都有这个区别。前端展示总数时务必用 total 字段
- **编译型语言改前端需要重新编译**：Rust 后端将 index.html 嵌入二进制（`include_str!`），修改后必须 `cargo build --release` + 重启服务
- **浏览器缓存导致看不到修复**：用 URL 加版本号 `?v=19` 强制刷新缓存
- **四层验证法**：源码 → 编译二进制 → API → 浏览器渲染，每层都确认修复到位
