# news-video 取消任务 API 与状态字段不一致导致误报"启动超时"

> **日期**: 2026-08-21  
> **分类**: 踩坑记录  
> **标签**: news-video, Python, 状态管理, 前端轮询, 竞态条件  
> **来源**: hermes

---

## 背景/问题

news-video WebUI 点击"生成视频"后，前端报"Pipeline 启动超时，请检查日志"，但实际 pipeline 在正常运行（步骤 4/7 截图渲染）。用户看到错误提示但视频实际在生成。

## 根因分析

### 问题 1：前端启动检测窗口太短

前端 `startStatusPolling()` 改进了启动检测逻辑（每 1 秒查一次，最多 10 秒），但 10 秒可能不够——LLM 阶段（步骤 2/7）本身就需要 60 秒，启动检测在 LLM 开始前可能还未写入 `running` 状态。

### 问题 2：状态字段名不一致

`run_async()` 写入状态时用的字段是 `"status": "running"`，而 `is_running()` 检查的也是 `"status" == "running"`。但前端 `startStatusPolling` 检查的是 `s.running` 字段——后端返回的 JSON 中 `running` 字段由 `get_status()` 计算：

```python
def get_status(self):
    s = _read_status()
    running = s.get("status") == "running"  # 从 status 字段推导
    return {
        "running": running,  # 前端用这个
        "step": s.get("step", ""),
        ...
    }
```

### 问题 3：CF 缓存 /api/status 响应

前端轮询 `/api/status` 被 Cloudflare 缓存了旧响应，导致前端看到的是旧的 `running: false`。

## 修复方案

### 1. 延长启动检测窗口到 30 秒

```javascript
function startStatusPolling(){
  if(statusTimer)clearInterval(statusTimer);
  let startupRetries=0;
  const startupCheck=setInterval(()=>{
    fetch('/api/status').then(r=>r.json()).then(s=>{
      if(s.running){
        clearInterval(startupCheck);
        addLog('running','Pipeline 已启动: '+(s.step||'初始化中'));
        // 开始正常轮询（每 2 秒）
        statusTimer=setInterval(()=>{ /* ... */ },2000);
        return;
      }
      startupRetries++;
      if(startupRetries>30){  // 30 秒超时（原 10 秒）
        clearInterval(startupCheck);
        addLog('error','Pipeline 启动超时，请检查日志');
      }
    }).catch(()=>{});
  },1000);
}
```

### 2. 添加 cache-busting 参数

前端轮询 URL 加时间戳参数，绕过 CF 缓存：

```javascript
fetch('/api/status?_=' + Date.now())
```

### 3. 取消任务 API 实现

pipeline 中添加 `_CANCEL_FLAG` 全局标志：

```python
_CANCEL_FLAG = {"cancel": False}

def request_cancel():
    _CANCEL_FLAG["cancel"] = True

def is_cancelled() -> bool:
    return _CANCEL_FLAG["cancel"]
```

每个阶段开始时检查：

```python
if is_cancelled():
    log("❌ 任务已取消")
    _write_status({**_read_status(), "status": "cancelled", "step": "已取消"})
    return
```

web_server.py 添加 `/api/cancel` 端点：

```python
def _handle_cancel(self):
    if not pipeline.is_running():
        self._json(400, {"error": "没有运行中的任务"})
        return
    pipeline.request_cancel()
    self._json(200, {"message": "取消请求已发送"})
```

## 经验总结

1. **状态字段命名要一致**：后端内部用 `status: "running"`，API 返回用 `running: true/false`，前端要对应 `running` 字段
2. **CF 会缓存 API 响应**：动态状态接口必须加 cache-busting 参数或 no-cache 头
3. **启动检测窗口要足够长**：pipeline 的第一步（采集新闻 + LLM）可能需要 60+ 秒，启动检测至少 30 秒
4. **取消机制用全局标志**：Python threading 中用 dict 作为可变标志，在每个阶段边界检查，比 thread.kill() 更安全
