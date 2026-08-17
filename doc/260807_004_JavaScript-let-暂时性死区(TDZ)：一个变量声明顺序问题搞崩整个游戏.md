# JavaScript let 暂时性死区(TDZ)：一个变量声明顺序问题搞崩整个游戏

> **日期**: 2026-08-07  
> **分类**: 踩坑记录  
> **标签**: 无  
> **来源**: hermes

---

## 背景/问题

一个网页小游戏（Canvas 飞机大战），点击"开始游戏"按钮完全没反应。页面能正常加载，API 接口也正常，但按钮就像死了一样。

浏览器控制台只有一个模糊的 JS 异常提示，没有具体错误信息。

## 原因分析

游戏代码用一个 **IIFE（立即执行函数）** 包裹所有逻辑：

```javascript
(function(){
  // ...几百行游戏代码...

  // 触摸位置驱动玩家移动
  function updatePlayerPos(){
    if((touchId!==null||(mouseInCanvas&&gameRunning))&&gameRunning){
      // ...
    }
    requestAnimationFrame(updatePlayerPos);
  }
  updatePlayerPos();  // ← 第704行：在这里调用了函数

  // 鼠标控制（桌面端）
  let mouseInCanvas=false;  // ← 第707行：变量在这里才声明！

  // ...事件绑定...
  document.getElementById('startBtn').onclick=startGame;  // ← 第791行
})();
```

问题出在 `mouseInCanvas` 变量的声明位置。`updatePlayerPos()` 函数在第 704 行被调用，函数内部引用了 `mouseInCanvas`，但这个变量在第 707 行才用 `let` 声明。

### 什么是暂时性死区（TDZ）？

`let` 和 `const` 声明的变量有一个特性叫 **Temporal Dead Zone（暂时性死区）**。跟 `var` 不同：

- `var` 声明的变量会被"提升"到作用域顶部，值为 `undefined`，提前访问不会报错
- `let`/`const` 声明的变量虽然也会提升，但在声明语句执行之前，**访问它会直接抛出 `ReferenceError`**

所以 `updatePlayerPos()` 执行时引用 `mouseInCanvas`，JS 引擎说"这个变量还没初始化，你不能用"，直接抛出：

```
ReferenceError: Cannot access 'mouseInCanvas' before initialization
```

这个错误导致整个 IIFE 在第 704 行崩溃，后面的代码（事件绑定、排行榜加载）**全部没有执行**。所以按钮的 onclick 没绑上，点自然没反应。

## 解决方案

把 `let mouseInCanvas=false;` 移到 `updatePlayerPos()` 调用之前：

```javascript
// ✅ 正确：先声明，再使用
let mouseInCanvas=false;  // ← 移到前面

function updatePlayerPos(){
  if((touchId!==null||(mouseInCanvas&&gameRunning))&&gameRunning){
    // ...
  }
  requestAnimationFrame(updatePlayerPos);
}
updatePlayerPos();
```

修改后重启服务，游戏正常启动。

## 调试技巧：如何在浏览器中定位这种错误

这种 IIFE 内部的错误很难发现，因为变量作用域被封在 IIFE 里，从外部看不到。调试方法：

### 方法 1：检查副作用判断 IIFE 是否执行完

```javascript
// 在浏览器控制台执行
var btn = document.getElementById('startBtn');
btn.onclick  // 如果是 null，说明 IIFE 在 onclick 绑定之前就崩了
```

### 方法 2：重新执行 IIFE 捕获异常

```javascript
// 取出 script 内容，用 new Function 重新执行并捕获错误
var scripts = document.querySelectorAll('script');
var scriptContent = scripts[scripts.length-1].textContent;
try {
  var fn = new Function(scriptContent);
  fn();
} catch(e) {
  console.error('IIFE crashed:', e.message, e.stack);
  // 输出: ReferenceError: Cannot access 'mouseInCanvas' before initialization
}
```

这个方法能精确找到崩溃位置。

## 避坑提示

- **`let`/`const` 声明的变量一定要在使用之前**。虽然这听起来是常识，但在大段代码中很容易把声明写在调用之后。
- **IIFE 里的错误是静默的**——不会冒泡到全局错误处理，因为 IIFE 自己就是一个作用域。如果 IIFE 中途崩溃，后面的代码全部跳过，表现就是"页面加载了但功能不工作"。
- **不要把函数调用放在变量声明之前**。即使函数定义在前，调用时如果引用了后面才声明的 `let` 变量，就会触发 TDZ。
- **调试 IIFE 崩溃最快的方法**：检查 IIFE 最后几行的副作用（事件绑定、DOM 修改）是否生效，如果不生效，用 `new Function(scriptContent)()` 重新执行来捕获异常。
