# CSS rgba() 语法陷阱：hex 值放入 rgba() 导致整个 background 声明失效

## 问题/背景

news-video 项目的 PPT 幻灯片生成器（slide_gen.py）中，新闻页背景应该是深色渐变（暗色主题），但实际截图显示新闻页全部是**白色背景**。片头和片尾页是正确的深色，只有新闻内容页出了问题。

## 关键操作

### 定位过程

1. 用 PIL 采样截图的像素亮度，确认新闻页亮度 255（纯白）
2. 检查生成的 HTML 文件，发现 CSS `background` 声明

### 根因

slide_gen.py 的 `_css()` 函数中，颜色变量 `c1` 和 `c2` 是 **hex 格式**（如 `#00f5d4`），但 CSS 中直接放进了 `rgba()`：

```css
/* 错误写法 */
background: radial-gradient(
    ellipse at 30% 20%,
    rgba(#00f5d4, #00bbf9, 0.06),   /* ← hex 值放进 rgba()！ */
    transparent 60%
);
```

`rgba()` 需要 4 个**数字参数**（R, G, B, A），但这里传入了 hex 字符串。整个 `background` 声明语法错误，浏览器**静默回退到默认值**（白色背景），没有任何报错。

### 修复方案

添加 `hex_rgba()` 辅助函数，正确转换 hex → rgba：

```python
def hex_rgba(hex_color, alpha=1.0):
    """hex (#00f5d4) → rgba(0, 245, 212, alpha)"""
    h = hex_color.lstrip("#")
    r = int(h[0:2], 16)
    g = int(h[2:4], 16)
    b = int(h[4:6], 16)
    return f"rgba({r},{g},{b},{alpha})"

def _css(category):
    c1 = cat["color"]      # #00f5d4
    c2 = cat["color2"]     # #00bbf9
    return f"""
    body {{
        background: linear-gradient(135deg, #08080f 0%, #101020 50%, #181028 100%);
    }}
    .slide {{
        background: radial-gradient(
            ellipse at 30% 20%,
            {hex_rgba(c1, 0.06)},     /* ← 正确的 rgba() */
            transparent 60%
        );
    }}
    """
```

### 验证结果

修复后重新生成截图，采样像素亮度从 255（白色）降到 98（深色渐变），视频背景统一为暗色主题。

## 经验/坑

### CSS 静默失败的特点

CSS 语法错误**不会报错**，浏览器会静默回退到默认值：
- `background` 声明失效 → 回退到 `transparent`（白色背景）
- `color` 声明失效 → 回退到默认文字颜色
- 这类问题在 JS 控制台里看不到任何错误

### 常见的 CSS 颜色函数陷阱

| 错误写法 | 问题 | 正确写法 |
|----------|------|----------|
| `rgba(#fff, 0.5)` | hex 不能放进 rgba() | `rgba(255,255,255,0.5)` |
| `rgba(#00f5d4, 0.06)` | 少了 2 个参数 | `rgba(0,245,212,0.06)` |
| `rgba(#00f5d4, #00bbf9, 0.06)` | hex 替代了数字参数 | 拆成两个 rgba() |
| `opacity: #00f5d4` | opacity 只接受数字 | `opacity: 0.5` |

### 排查技巧

当页面背景颜色不对时：
1. **PIL 采样像素**：`Image.open(screenshot).getpixel((x, y))` 快速确认实际颜色
2. **检查 CSS 语法**：grep 生成的 HTML 中的 `background` 声明
3. **浏览器 DevTools**：Elements 面板看计算后的样式（computed style）
4. **不要只看中心像素**：白色文字在中心区域会干扰判断，采样边缘区域更准确
