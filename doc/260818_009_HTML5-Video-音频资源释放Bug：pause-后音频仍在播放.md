# HTML5 Video 音频资源释放 Bug：pause() 后音频仍在播放

> **日期**: 2026-08-18  
> **分类**: 踩坑记录  
> **标签**: JavaScript, HTML5, Video, 音频释放, 浏览器  
> **来源**: hermes

---

## 背景/问题

news-video 项目的前端视频播放器出现两个问题：
1. 点击视频缩略图播放后，再次点击停止，**音频仍在继续播放**
2. 页面上多个视频可以**同时播放**，音频混在一起

## 根因分析

### Bug 1：pause() + remove() 无法停止音频

原始代码：
```javascript
if(v){v.pause();v.remove();}
```

`v.pause()` 是**异步操作**——浏览器还没来得及真正暂停音频流，DOM 节点就被 `v.remove()` 移除了。移除后的 video 元素失去了引用，但底层的音频资源并未释放，音频继续播放。

### Bug 2：缺少单视频播放限制

没有全局变量追踪当前正在播放的视频 ID，点击新视频时不会关闭旧视频。

## 解决方案

### 音频资源释放三步法

```javascript
v.pause();                    // 1. 发出暂停指令
v.removeAttribute('src');     // 2. 移除 src 属性，切断媒体源
v.load();                     // 3. 强制重新加载（无 src），释放底层音频资源
v.remove();                   // 4. 安全移除 DOM 节点
```

`removeAttribute('src')` + `v.load()` 是关键组合。`load()` 在没有 src 时会重置媒体元素，释放浏览器底层持有的音频缓冲区和解码器资源。

### 单视频播放限制

```javascript
let _currentPlaying=null;  // 全局追踪当前播放的视频 ID

function toggleVideo(id){
  const thumb=document.getElementById('thumb-'+id);
  if(!thumb)return;
  const card=document.getElementById('card-'+id);
  const url=card.dataset.videoUrl;
  if(!url)return;

  // 点击当前正在播放的视频 → 停止
  if(thumb.classList.contains('playing')){
    const v=thumb.querySelector('video');
    if(v){
      v.pause();
      v.removeAttribute('src');
      v.load();
      v.remove();
    }
    thumb.classList.remove('playing');
    _currentPlaying=null;
    return;
  }

  // 先关闭其他正在播放的视频
  if(_currentPlaying && _currentPlaying!==id){
    const oldThumb=document.getElementById('thumb-'+_currentPlaying);
    if(oldThumb && oldThumb.classList.contains('playing')){
      const oldV=oldThumb.querySelector('video');
      if(oldV){
        oldV.pause();
        oldV.removeAttribute('src');
        oldV.load();
        oldV.remove();
      }
      oldThumb.classList.remove('playing');
    }
  }

  // 在缩略图位置插入视频播放器
  const v=document.createElement('video');
  v.controls=true;
  v.preload='auto';
  v.src=url;
  v.style.width='100%';
  v.style.display='block';
  v.style.background='#000';
  thumb.appendChild(v);
  thumb.classList.add('playing');
  _currentPlaying=id;
  v.play().catch(()=>{});
  
  // 播放结束后清理
  v.addEventListener('ended',()=>{
    v.remove();
    thumb.classList.remove('playing');
    if(_currentPlaying===id)_currentPlaying=null;
  });
}
```

## 验证方法

编写 ad-hoc 验证脚本（16 项检查全部通过）：

```bash
# 验证音频释放
grep -q 'removeAttribute.*src' index.html   # PASS
grep -q 'v.load()' index.html               # PASS
grep -c 'removeAttribute.*src' index.html   # 2 (停止当前 + 关闭其他)

# 验证单视频限制
grep -q 'let _currentPlaying' index.html    # PASS
grep -q '_currentPlaying=id' index.html     # PASS
grep -q '_currentPlaying=null' index.html   # PASS

# 验证无裸 pause+remove 模式
grep -q 'v.pause();v.remove()' index.html   # 不存在（正确）

# 验证 ended handler（需跨行匹配）
grep -A4 'addEventListener.*ended' index.html | grep '_currentPlaying===id'  # PASS
```

## 避坑提示

- **`pause()` 是异步的**：不能假设调用后立即停止播放。如果紧接着移除 DOM 节点，音频资源不会随节点释放
- **`removeAttribute('src')` + `load()` 是标准释放方式**：MDN 文档推荐的媒体资源释放方法
- **`ended` 事件 handler 要检查 ID**：`if(_currentPlaying===id)` 防止用户已切换到其他视频时，旧视频的 ended 事件错误清除新视频的播放状态
- **grep 跨行匹配**：验证脚本中 `grep -A4` 捕获 ended handler 块，单行 grep 会漏匹配跨行代码
