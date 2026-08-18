# heron-web 前端手写 Markdown 渲染器架构分析

> **日期**: 2026-08-18  
> **分类**: 技术知识  
> **标签**: JavaScript, Markdown, 渲染器, 正则表达式, 语法高亮, heron-web  
> **来源**: hermes

---

## 背景/问题

heron-web 主站前端有一个纯手写的 Markdown→HTML 渲染器（`renderMarkdown` 函数），零第三方依赖。本文档完整分析其架构、处理流程和局限性，为后续是否迁移到 markdown-it 提供决策依据。

## 渲染器架构

### 数据流

```
hermes-notes/doc/*.md
  → Rust 后端 parse_markdown_note() 提取元信息+正文
  → /api/notes/{id} 返回 JSON { title, content, tags, category, ... }
  → 前端 showNote(id) 调 fetch 获取
  → renderMarkdown(n.content) 正则转 HTML
  → 写入 #noteBody div
```

### renderMarkdown() 处理顺序

手写的正则式 Markdown→HTML 转换器，按以下顺序处理（顺序很重要，后面的正则可能破坏前面的输出）：

| 步骤 | 语法 | 输出 | 说明 |
|------|------|------|------|
| 1 | ` ```lang ``` ` | `<pre><code>` | 先提取存储，最后还原（避免被后续正则误伤） |
| 2 | `\|...\|` 表格 | `<table>` | 管道表格，含 thead/tbody |
| 3 | `# ~ ####` | `<h1>~<h4>` | 带随机 id 供目录跳转 |
| 4 | `---` | `<hr>` | 分隔线 |
| 5 | `> text` | `<blockquote>` | 引用块 |
| 6 | `- [x]` / `- [ ]` | `<input checkbox>` | 任务列表 |
| 7 | `1. ` | `<ol>` | 有序列表 |
| 8 | `- ` / `* ` | `<ul>` | 无序列表 |
| 9 | `**bold**` | `<strong>` | 粗体 |
| 10 | `*italic*` | `<em>` | 斜体 |
| 11 | `` `code` `` | `<code>` | 行内代码 |
| 12 | `![alt](url)` | `<img>` | 图片，点击弹灯箱 |
| 13 | `[text](url)` | `<a>` | 链接 |
| 14 | `\n` | `<br>` | 换行 |
| 15 | 还原代码块 | `<pre><code>` | 带 highlightCode() 语法高亮 |

### highlightCode() 语法高亮

对代码块做关键字高亮，支持 Python / Rust / JavaScript / Go：

```javascript
const KEYWORDS={
  python:['def','class','import','from','return','if','elif','else',...],
  rust:['fn','let','mut','const','static','struct','enum','trait',...],
  javascript:['var','let','const','function','return','if','else',...],
  go:['func','var','const','type','struct','interface','map',...]
};

function highlightCode(code,lang){
  const kw=KEYWORDS[lang]||KEYWORDS.javascript;
  let h=escapeHtml(code);                    // 1. 先转义 HTML
  h=h.replace(/(\/\/[^\n]*)/g,...);          // 2. 注释 (// 和 #)
  h=h.replace(/("(?:[^"\\]|\\.)*")/g,...);   // 3. 字符串
  h=h.replace(/\b(\d+\.?\d*)\b/g,...);       // 4. 数字
  h=h.replace(kwRe,...);                      // 5. 关键字
  h=h.replace(/(\w+)\s*\(/g,...);            // 6. 函数调用名
  return h;
}
```

CSS 高亮颜色（暗色主题）：
```css
.tok-kw{color:#c792ea;font-weight:bold}   /* 关键字 - 紫色 */
.tok-str{color:var(--green)}               /* 字符串 - 绿色 */
.tok-num{color:var(--orange)}              /* 数字 - 橙色 */
.tok-com{color:#5c6370;font-style:italic}  /* 注释 - 灰色斜体 */
.tok-fn{color:var(--blue)}                 /* 函数名 - 蓝色 */
.tok-type{color:var(--yellow)}             /* 类型 - 黄色 */
```

### CSS 样式系统

所有 Markdown 元素使用 `md-` 前缀的自定义 CSS 类，适配暗色主题：

```css
.md-h1{font-size:1.5rem;color:var(--cyan);margin:1.5rem 0 0.8rem;font-weight:bold}
.md-h2{font-size:1.3rem;color:var(--cyan);margin:1.2rem 0 0.6rem;font-weight:bold}
.md-pre{background:rgba(0,0,0,0.35);border:1px solid var(--border);border-radius:8px;padding:1rem;overflow-x:auto}
.md-code{background:rgba(0,0,0,0.3);padding:0.15rem 0.4rem;border-radius:4px;color:var(--pink)}
.md-table{width:100%;border-collapse:collapse;display:block;overflow-x:auto}
.md-table th{background:rgba(0,245,212,0.1);color:var(--cyan)}
.md-quote{border-left:3px solid var(--purple);padding:0.5rem 1rem;background:rgba(155,93,229,0.08)}
.md-link{color:var(--blue);border-bottom:1px dotted var(--blue)}
```

亮色主题适配：
```css
body.light .md-pre{background:rgba(0,0,0,0.05)}
body.light .md-code{background:rgba(0,0,0,0.06)}
```

### showNote() 笔记详情页渲染入口

```javascript
async function showNote(id){
  const r=await fetch('/api/notes/'+id);
  const n=await r.json();
  fetch('/api/notes/'+id+'/view',{method:'POST'}).catch(()=>{});  // 记录阅读次数
  
  const el=document.getElementById('notesContent');
  el.innerHTML='<div class="note-detail">'+
    '<div class="font-ctrl"><button onclick="adjustFont(-1)">A-</button><button onclick="adjustFont(1)">A+</button></div>'+
    '<span class="back" onclick="loadNotesList()">← 返回列表</span>'+
    '<h2>'+escapeHtml(n.title)+'</h2>'+
    '<div class="meta">...'+(n.source==='hermes'?'<span>via Hermes</span>':'')+'</div>'+
    '<div class="note-detail-content" id="noteBody" style="--reader-font:'+readerFont+'px">'+
      renderMarkdown(n.content)+  // ← 核心渲染调用
    '</div>'+
    '</div>';
  
  loadPrevNext(id);   // 上一篇/下一篇
  loadRelated(id);    // 相关推荐
  buildToc();         // 生成目录
  initScrollListener();  // 滚动监听（阅读进度条+目录高亮）
}
```

## 局限性

纯正则实现的渲染器有以下已知限制：

- **不支持嵌套列表**：二级缩进子项无法正确渲染
- **不支持嵌套引用块**：引用块内的列表、代码块等无法嵌套
- **`*斜体*` 可能误匹配列表项的 `*`**：列表标记和斜体标记冲突
- **HTML 先转义再正则替换**：某些边界情况可能异常
- **代码块语法高亮是简单正则**：不是真正的 tokenizer，复杂代码可能高亮错误
- **表格不支持对齐语法**：`:---:` 等对齐标记不被处理

对于当前的技术笔记内容来说够用。如果后续需要更完整的 Markdown 支持，可以迁移到 markdown-it（详见 `260818_008_marked.js-vs-markdown-it` 笔记）。

## 避坑提示

- **正则替换顺序很重要**：代码块必须先提取（placeholder），其他正则处理完后再还原，否则代码块内容会被破坏
- **`escapeHtml` 要在最前面做**：先转义 HTML 特殊字符，再做 Markdown 正则替换，防止 XSS
- **`md-` 前缀 CSS 类**：使用自定义前缀避免与标准 HTML 标签样式冲突。迁移到 markdown-it 时需要改为标准标签选择器或用渲染规则插件
- **阅读字号控制**：`--reader-font` CSS 变量 + `adjustFont()` 函数，localStorage 持久化用户偏好（12-24px 范围）
