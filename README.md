# hermes-notes

技术笔记归档仓库。所有笔记统一存放在 `doc/` 目录下。

## 文件命名规范

```
YYMMDD_NNN_标题.md
```

- `YYMMDD` — 笔记日期（如 260806 = 2026年8月6日）
- `NNN` — 当日序号，从 001 开始
- `标题` — 简短中文标题

示例：`260806_001_Docker部署踩坑记录.md`

## 目录结构

```
hermes-notes/
├── README.md          ← 本文件
├── doc/               ← 所有笔记
│   ├── 260806_001_Docker部署踩坑记录.md
│   ├── 260806_002_Cloudflare-Tunnel暴露本地服务.md
│   └── ...
└── .gitignore
```

## 笔记格式

每篇笔记为 Markdown，包含以下结构：

- 背景或问题描述
- 原因分析
- 解决方案
- 避坑提示

## 来源

- heron-web/data/notes.json 中 41 条历史笔记
- ~/Docs/ 下散落的笔记文件
