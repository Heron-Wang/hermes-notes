# news-video PPT 多主题系统：4 套预设风格与 Web 界面选择

## 问题/背景

news-video 新闻视频管线的 PPT 幻灯片此前只有单一暗色主题。用户希望支持多套不同风格（极简/科技/杂志/暗黑），并实现 Web 界面选择主题后生成的功能。

## 关键操作

### 主题系统设计

在 `config.py` 中添加 4 套主题预设：

```python
THEMES = {
    "科技": {
        "name": "科技霓虹",
        "bg_gradient": "linear-gradient(135deg,#08080f 0%,#101020 50%,#181028 100%)",
        "bg_overlay": "radial-gradient(ellipse at 30% 20%,rgba(0,245,212,0.06),transparent 60%)",
        "text_color": "#e0e0e0",
        "title_color": "#00f5d4",
        "card_bg": "rgba(255,255,255,0.03)",
        "card_border": "rgba(0,245,212,0.15)",
    },
    "极简": {
        "name": "极简白",
        "bg_gradient": "linear-gradient(135deg,#fafafa 0%,#f0f0f0 100%)",
        "bg_overlay": "none",
        "text_color": "#333",
        "title_color": "#1a1a1a",
        "card_bg": "rgba(0,0,0,0.02)",
        "card_border": "rgba(0,0,0,0.08)",
    },
    "杂志": {
        "name": "杂志风",
        "bg_gradient": "linear-gradient(135deg,#1a1a2e 0%,#16213e 100%)",
        "bg_overlay": "radial-gradient(ellipse at 70% 30%,rgba(255,107,107,0.08),transparent 50%)",
        "text_color": "#e8e8e8",
        "title_color": "#ff6b6b",
        "card_bg": "rgba(255,255,255,0.04)",
        "card_border": "rgba(255,107,107,0.2)",
    },
    "暗黑": {
        "name": "纯黑暗黑",
        "bg_gradient": "linear-gradient(135deg,#000 0%,#0a0a0a 100%)",
        "bg_overlay": "none",
        "text_color": "#ccc",
        "title_color": "#fff",
        "card_bg": "rgba(255,255,255,0.02)",
        "card_border": "rgba(255,255,255,0.1)",
    },
}
```

### slide_gen.py 主题适配

`_css()` 函数接受 `theme` 参数，根据主题生成不同 CSS：

```python
def _css(category, theme="科技"):
    t = THEMES.get(theme, THEMES["科技"])
    cat = CATEGORIES.get(category, CATEGORIES["科技"])
    c1 = cat["color"]      # 分类主色
    c2 = cat["color2"]     # 分类辅色

    return f"""
    * {{ margin:0; padding:0; box-sizing:border-box; }}
    body {{
        background: {t["bg_gradient"]};
        color: {t["text_color"]};
        font-family: -apple-system, "PingFang SC", "Microsoft YaHei", sans-serif;
    }}
    .slide {{
        width: 1920px;
        height: 1080px;
        background: {t["bg_gradient"]};
        position: relative;
        overflow: hidden;
    }}
    .slide::after {{
        content: "";
        position: absolute;
        inset: 0;
        background: {t["bg_overlay"]};
        pointer-events: none;
    }}
    .news-title {{
        color: {t["title_color"]};
        font-size: 48px;
        font-weight: 700;
    }}
    .news-card {{
        background: {t["card_bg"]};
        border: 1px solid {t["card_border"]};
        border-radius: 12px;
        padding: 24px;
    }}
    """
```

### Web 界面主题选择器

在 `web/index.html` 中添加主题下拉选择器：

```html
<select id="themeSelect">
    <option value="科技">科技霓虹</option>
    <option value="极简">极简白</option>
    <option value="杂志">杂志风</option>
    <option value="暗黑">纯黑暗黑</option>
</select>
```

### pipeline 主题参数传递

```python
# pipeline.py
def run_pipeline(category="科技", theme="科技", ...):
    # 生成 PPT 时传入 theme
    slides = slide_gen.generate_slides(news_items, category, theme=theme)
    # 截图
    screenshot.render_all(slides, theme=theme)
    # 视频合成
    video_composer.compose(clips, srt_path, bgm_path, output_path)
```

## 经验/坑

### PPT 主题设计要点

| 要素 | 科技主题 | 极简主题 | 杂志主题 | 暗黑主题 |
|------|----------|----------|----------|----------|
| 背景 | 深色渐变+光晕 | 浅色纯色 | 深蓝+暖色光晕 | 纯黑 |
| 文字 | 浅色 | 深色 | 浅色 | 浅灰 |
| 标题 | 分类主色（青/紫/蓝） | 黑色 | 暖色（红/橙） | 白色 |
| 卡片 | 半透明+发光边框 | 浅灰边框 | 半透明+暖色边框 | 极淡边框 |
| 字体 | 系统字体 | 系统字体 | 系统字体 | 系统字体 |

### 关键设计决策

1. **不用 Google Fonts CDN**：CDN 在国内访问超时，改用系统字体 `-apple-system, PingFang SC, Microsoft YaHei`
2. **hex_rgba() 辅助函数**：避免 CSS rgba() 语法错误（详见 `260820_002` 笔记）
3. **分类配色与主题分离**：分类（科技/AI/财经）决定主色调，主题决定整体风格（背景/文字/卡片样式）
4. **PPT 设计迭代经验**：v1(霓虹特效)→v2(去特效)→v3(Linear简洁)→v4(Framer+Runway编辑式)→v5(卡片式)→v8(时间线+统计)。参考了 popular-web-designs skill 的 Linear/Vercel/Spotify/Framer/Runway 模板

### 主题优化方向

| 方向 | 说明 | 实现难度 |
|------|------|----------|
| 分类配色 | 科技(青)/AI(紫)/财经(蓝) 不同主色调 | ✅ 已实现 |
| 多套主题模板 | 极简/科技/杂志/暗黑 | ✅ 已实现 |
| 动态主题切换 | Web 界面选择主题后生成 | ✅ 已实现 |
| AI 生成配色 | 根据新闻内容自动选配色方案 | 可扩展 |
| 自定义主题色 | 用户在 Web 界面输入 hex 值 | 可扩展 |
