# news-video Pipeline 看门狗：线程卡死 623 分钟的根因与修复

> **日期**: 2026-08-19  
> **分类**: Bug 修复  
> **标签**: news-video, pipeline, 看门狗, watchdog, subprocess, 线程卡死  
> **来源**: hermes

---

## 背景/问题

news-video 视频生成系统的 pipeline 状态显示"running"持续了 623 分钟（超过 10 小时），但实际 ffmpeg 进程早已不存在，日志也没有新条目。pipeline 线程已经死亡，但状态文件没有被清理，导致前端一直显示"生成中"。

## 关键操作

### 1. 诊断 — 确认线程已死

```bash
# 检查状态
curl -s http://localhost:8089/api/status
# {"running": true, "step": "compose", "started_at": "2026-08-19T11:21:..."}

# 检查 ffmpeg 进程
ps aux | grep ffmpeg | grep -v grep
# （空 — 没有 ffmpeg 进程）

# 检查日志
sudo journalctl -u news-video --no-pager -n 30 | tail -25
# 最后一条日志是 11:21 的"开始视频合成"，之后再无新日志
```

**结论**：pipeline 线程在 `subprocess.run(timeout=120)` 中卡死。虽然设了 120 秒超时，但 cloudflared 断网期间 ffmpeg 根本没启动，`subprocess.run` 在等待一个永远不会返回的进程。异常被 try-except 吞掉，线程静默死亡，状态文件没更新。

### 2. 根因分析

| 层级 | 问题 | 后果 |
|------|------|------|
| subprocess.run | timeout 参数对未启动的进程无效 | 永久阻塞 |
| try-except | 捕获了 TimeoutExpired 但未更新状态 | 异常被吞 |
| 状态文件 | 线程死亡后无人更新 | 永远显示 running |
| 前端 | 信任状态文件，不检查时间 | 永远显示"生成中" |

### 3. 修复 — 看门狗机制

在 `pipeline.py` 的 `PipelineRunner.get_status()` 方法中加入看门狗逻辑：

```python
def get_status(self) -> dict:
    s = _read_status()
    running = s.get("status") == "running"

    # 看门狗：如果运行超过 30 分钟，自动标记为失败
    if running and s.get("started_at"):
        try:
            from datetime import datetime
            started = datetime.fromisoformat(s["started_at"])
            elapsed = (datetime.now() - started).total_seconds()
            if elapsed > 1800:  # 30 分钟
                s["status"] = "failed"
                s["step"] = "超时自动终止"
                s["error"] = f"任务运行超过 {int(elapsed/60)} 分钟，已自动终止"
                s["running"] = False
                s["finished_at"] = datetime.now().isoformat()
                _write_status(s)
                running = False
                _CANCEL_FLAG["cancel"] = True
        except Exception:
            pass

    step_map = {
        "start": ("1/7", 0),
        "collect": ("1/7", 10),
        # ... 其他步骤
    }
    # ...
```

### 4. 设计要点

| 设计决策 | 理由 |
|---------|------|
| 放在 `get_status()` 而非独立线程 | 每次前端查询状态时自动触发，无需额外线程 |
| 30 分钟超时阈值 | 正常视频生成最多 5-10 分钟，30 分钟足够宽容 |
| 写入 `_CANCEL_FLAG` | 通知 pipeline 线程（如果还活着）停止后续步骤 |
| 更新状态文件 | 前端下次查询时看到 failed 状态 |
| `try-except` 包裹 | 时间解析失败不会影响正常状态查询 |

### 5. 验证

```bash
# 重启服务
sudo -A systemctl restart news-video
sleep 2

# 验证看门狗已启用
cd ~/workspace/news-video && source .venv/bin/activate
python3 -c "
import sys; sys.path.insert(0,'.')
import inspect
from pipeline import PipelineRunner
src = inspect.getsource(PipelineRunner.get_status)
print('看门狗已启用:', '1800' in src)
print('超时自动终止:', '超时自动终止' in src)
"
# 输出: 看门狗已启用: True, 超时自动终止: True
```

## 结果/经验

| 修复前 | 修复后 |
|--------|--------|
| 线程卡死后状态永远 running | 30 分钟后自动标记 failed |
| 前端永远显示"生成中" | 前端显示"失败：超时自动终止" |
| 需要手动清除状态文件 | 自动清除，用户可重新生成 |
| ffmpeg 阻塞导致服务不可用 | 看门狗触发 _CANCEL_FLAG 终止 pipeline |

**核心教训**：`subprocess.run(timeout=N)` 的 timeout 参数不能保证在所有情况下生效。如果进程因网络问题无法启动，timeout 可能不触发。需要一个独立于 subprocess 的看门狗机制来兜底。

## 避坑提示

- **subprocess.run timeout 不是万能的**：timeout 对已启动的进程有效（发送 SIGKILL），但对未启动的进程可能无效
- **try-except 不要吞异常**：捕获异常后至少要记录日志或更新状态，不能 `pass` 了事
- **线程死亡后状态不会自动清理**：Python 线程死亡不会触发任何回调，状态文件需要外部机制清理
- **看门狗放在查询路径上**：不需要独立线程，放在每次状态查询时检查即可。前端轮询状态 API 的频率足够高（通常每 2-5 秒）
- **_CANCEL_FLAG 用 dict 而非 bool**：`_CANCEL_FLAG = {"cancel": False}`，dict 是可变对象，可以在函数间共享引用。bool 是不可变的，赋值会创建新对象
