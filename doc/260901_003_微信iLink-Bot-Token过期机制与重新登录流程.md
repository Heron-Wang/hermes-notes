# 微信 iLink Bot Token 过期机制与重新登录流程

## 背景

NAS 重启后 Hermes Gateway 手动启动成功，微信平台已连接（长轮询正常运行），但所有发送消息均失败。日志中反复出现：

```
[Weixin] send chunk failed to=o9cq808- attempt=1/5, retrying in 1.00s:
[Weixin] send chunk failed to=o9cq808- attempt=2/5, retrying in 2.00s:
[Weixin] send chunk failed to=o9cq808- attempt=3/5, retrying in 3.00s:
[Weixin] send failed to=o9cq808-:
```

错误消息为空（末尾冒号后无内容），导致排查方向一度偏离。

## 关键排查过程

### 1. 直接测试 iLink API

绕过 Hermes，直接用 curl 调用 iLink sendmessage API：

```bash
curl -s -w "\nHTTP %{http_code}\n" --connect-timeout 10 \
  -X POST "https://ilinkai.weixin.qq.com/ilink/bot/sendmessage" \
  -H "Content-Type: application/json" \
  -H "AuthorizationType: ilink_bot_token" \
  -H "iLink-App-Id: ilink_xbot" \
  -d '{"base_info":{"channel_version":"1.0.0"},
       "msg":{"from_user_id":"","to_user_id":"o9cq808-***@im.wechat",
       "client_id":"test","message_type":1,"message_state":1,
       "item_list":[{"type":1,"text_item":{"text":"test"}}]}}'
```

返回：
```json
{"errcode":-14,"errmsg":"session timeout"}
HTTP 200
```

### 2. 确认 DNS 和网络路径

```bash
nslookup ilinkai.weixin.qq.com
# Name: ilinkai.weixin.qq.com
# Address: 198.18.39.160

ip route get 198.18.39.160
# 198.18.39.160 via 192.168.0.104 dev bridge0 src 192.168.0.6
```

`198.18.0.0/15` 是保留地址段（Clash TUN fake-ip），流量经过软路由 192.168.0.104 的 Clash 代理。网络可达，但 API 返回 `session timeout`。

### 3. 源码分析

在 `weixin.py` 中查找 `SESSION_EXPIRED_ERRCODE`：

```python
# weixin.py 第 113 行
SESSION_EXPIRED_ERRCODE = -14
RATE_LIMIT_ERRCODE = -2  # iLink frequency limit — backoff and retry

# 第 118-126 行：stale session 检测
def _is_stale_session_ret(ret, errcode, errmsg):
    """True when iLink returns ret=-2 / errcode=-2 with 'unknown error',
    which is a stale-session signal (same as errcode=-14) rather than
    a genuine rate limit."""
    if ret != RATE_LIMIT_ERRCODE and errcode != RATE_LIMIT_ERRCODE:
        return False
    return (errmsg or "").lower() == "unknown error"
```

`errcode=-14` 即 `SESSION_EXPIRED_ERRCODE`，表示 bot token 已过期。

### 4. 发送失败日志为空的原因

源码第 1860-1875 行：

```python
except Exception as exc:
    last_error = exc
    ...
    logger.warning(
        "[%s] send chunk failed to=%s attempt=%d/%d, retrying in %.2fs: %s",
        self.name, _safe_id(chat_id), attempt+1, self._send_chunk_retries+1,
        wait, exc,  # ← exc 的 str() 为空
    )
```

发送异常 `exc` 的字符串表示为空（`str(exc) == ""`），导致日志末尾冒号后无内容。

## 根因

**iLink bot token 过期**。Token 约每 3-4 周自动过期，返回 `errcode=-14, errmsg="session timeout"`。这是 iLink 平台的正常 session 管理机制，不是网络或配置问题。

## 解决方案

### 重新扫码登录

```bash
hermes gateway setup
```

在交互菜单中选择 "Weixin / WeChat"（显示为 configured），重新走 QR 登录流程：

1. Hermes 调用 iLink API 获取二维码
2. 终端打印二维码（ASCII 或 URL）
3. 用手机微信扫码并确认
4. 获取新 token，自动写入 `~/.hermes/.env` 的 `WEIXIN_TOKEN`
5. Gateway 自动用新 token 重连

### QR 登录源码流程

```python
# weixin.py 第 1037-1117 行
async def qr_login(hermes_home, *, bot_type="3", timeout_seconds=480):
    # 1. 获取二维码
    qr_resp = await _api_get(session, endpoint=f"{EP_GET_BOT_QR}?bot_type={bot_type}")
    qrcode_value = qr_resp.get("qrcode")
    qrcode_url = qr_resp.get("qrcode_img_content")  # 完整扫码 URL

    # 2. 渲染二维码
    qr = qrcode.QRCode()
    qr.add_data(qrcode_url if qrcode_url else qrcode_value)
    qr.print_ascii(invert=True)

    # 3. 轮询扫码状态
    while time.monotonic() < deadline:
        status_resp = await _api_get(session,
            endpoint=f"{EP_GET_QR_STATUS}?qrcode={qrcode_value}")
        status = status_resp.get("status")
        # wait → 扫码中
        # scaned → 已扫码，等待确认
        # scaned_but_redirect → 重定向到新 host
        # expired → 二维码过期，刷新
```

## iLink API 架构

| 组件 | 说明 |
|------|------|
| Base URL | `https://ilinkai.weixin.qq.com` |
| 认证 | `Authorization: Bearer <token>` |
| App ID | `ilink_xbot` |
| 消息端点 | `ilink/bot/sendmessage` |
| 长轮询端点 | `ilink/bot/getupdates` |
| 二维码端点 | `ilink/bot/getqrcode` + `ilink/bot/get_qrcode_status` |
| 超时 | 长轮询 35s，API 15s，配置 10s，QR 35s |

### 错误码

| errcode | 含义 | 处理 |
|---------|------|------|
| 0 | 成功 | — |
| -2 | 频率限制 / stale session | 指数退避重试（3x backoff） |
| -14 | session timeout（token 过期） | 暂停 10 分钟，需重新扫码 |

### 发送重试机制

- 最大重试次数：5 次（`_send_chunk_retries`）
- 重试间隔：递增 1s, 2s, 3s, 4s
- Rate limit 退避：3 倍正常间隔
- Session expired：先尝试去掉 context_token 重试一次，失败则抛异常

## 经验总结

1. **token 3-4 周过期是常态**：iLink bot session 不是永久有效的，定期需要重新扫码
2. **发送失败日志为空≠无错误**：异常对象的 `str()` 可能为空，需直接调 API 确认
3. **先直接测 API**：遇到平台问题时，绕过框架直接 curl 测 API 是最快的定位方式
4. **`hermes gateway setup` 是万能工具**：微信平台配置、重新登录、token 刷新都通过这一个命令

## 坑点

1. **发送失败日志看起来像网络问题**：空错误消息 + 重试退避，容易误判为网络超时，实际是 token 过期
2. **长轮询正常≠token 有效**：`getupdates`（接收）和 `sendmessage`（发送）使用同一个 token，但 token 过期时接收侧可能不立即报错
3. **Clash fake-ip 干扰排查**：DNS 解析到 `198.18.x.x`（Clash TUN fake-ip），看似异常但实际是正常代理路径

---

*来源：2026-09-01 微信连接失败排查会话*
