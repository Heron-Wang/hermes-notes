# Cloudflare Tunnel 530 故障深度排查：Clash TUN fake-ip 与边缘节点连接冲突

> **日期**: 2026-08-21  
> **分类**: 运维操作  
> **标签**: Cloudflare-Tunnel, Clash, TUN模式, fake-ip, 530错误, 网络排查  
> **来源**: hermes

---

## 背景/问题

所有 heronwang.cn 子域名外网返回 530 错误，本地服务正常（200）。cloudflared 日志显示 `hard_fail=true`——QUIC 和 HTTP/2 都连不上 Cloudflare 边缘节点。

## 排查过程

### 第 1 步：检查服务状态

```bash
# 本地正常
curl http://localhost:8080/  → 200
curl http://localhost:8089/  → 200

# 外网 530
curl https://heronwang.cn/   → 530
curl https://news.heronwang.cn/ → 530
```

### 第 2 步：检查 cloudflared 日志

```bash
sudo journalctl -u cloudflared --no-pager -n 15
```

关键日志：
```
precheck component="TCP Connectivity" details="HTTP/2 connection is blocked or unreachable" status=fail
precheck component="UDP Connectivity" details="QUIC connection successful" status=pass
precheck complete hard_fail=false suggested_protocol=quic
ERR Unable to establish connection error="TLS handshake with edge error: EOF"
```

→ HTTP/2 被 TLS 阻断，QUIC 可用。切换 protocol 为 quic。

### 第 3 步：切换协议后 QUIC 也断了

切换到 QUIC 后短暂恢复（200），但随后又出现 530：

```
ERR Failed to dial a quic connection error="timeout: no recent network activity"
ERR Connection terminated error="there are no free edge addresses left to resolve to"
```

### 第 4 步：网络连通性测试

```bash
# 直连 Cloudflare
curl https://www.cloudflare.com → 000 (失败!)

# ping CF 边缘 IP
ping 198.18.41.88 → 0.4ms (通!)

# 国内网站
curl https://www.baidu.com → 200 (正常)

# Clash API
curl http://192.168.0.104:9090/version → v1.19.21 (正常)
```

### 第 5 步：发现根因——Clash TUN fake-ip

```bash
curl http://192.168.0.104:9090/configs
```

返回：
```json
{
  "tun": {
    "enable": true,
    "device": "nikki",
    "stack": "System",
    "inet4-address": ["198.18.0.1/30"],
    "strict-route": true
  }
}
```

**关键发现**：`198.18.0.1/30` 是 Clash TUN 的 fake-ip 地址段。cloudflared 连的 `198.18.41.88` 不是真实的 Cloudflare 边缘 IP，而是 Clash TUN 分配的 fake-ip！

TUN 模式拦截了所有流量（包括 cloudflared 的 QUIC/HTTP2），但处理失败导致 530。

### 第 6 步：尝试关闭 TUN

```bash
# 通过 Clash API 关闭 TUN
curl -X PATCH http://192.168.0.104:9090/configs \
  -H "Authorization: Bearer ***REDACTED***" \
  -d '{"tun":{"enable":false}}'

# 测试
curl https://heronwang.cn → 530 (仍然失败)
```

关闭 TUN 后仍然 530，说明问题不仅是 TUN——Cloudflare 边缘节点本身连不上（可能是 ISP 层面阻断）。

### 第 7 步：最终恢复

用户重启了软路由后，所有网站恢复 200：

```bash
curl https://heronwang.cn → 200 ✅
curl https://news.heronwang.cn → 200 ✅
```

## 完整诊断流程图

```
530 错误
  ├─ 本地服务正常？ → 是 → 问题在 cloudflared 或网络
  │   ├─ cloudflared 日志 hard_fail=true？
  │   │   ├─ HTTP/2 fail + QUIC pass → 切换 protocol: quic
  │   │   └─ QUIC 也 fail → 网络层面问题
  │   │       ├─ curl cloudflare.com → 000 → 出网被阻断
  │   │       ├─ ping 198.18.x.x → 通 → 可能是 fake-ip
  │   │       ├─ curl baidu.com → 200 → 国内正常
  │   │       └─ 检查 Clash TUN 配置 → inet4-address 198.18.0.1/30
  │   └─ 重启软路由/Clash 服务
  └─ 本地服务也挂？ → 检查 systemd 服务状态
```

## 经验总结

1. **530 = Cloudflare Tunnel 连接问题**：不是源站问题，是 cloudflared ↔ CF 边缘的连接断开
2. **198.18.x.x 是 Clash fake-ip**：ping 通不代表网络通，这只是 Clash TUN 的虚拟地址
3. **协议切换是临时方案**：http2 → quic 可绕过 TLS 阻断，但 QUIC 本身也可能被阻断
4. **Clash TUN 影响 cloudflared**：TUN 拦截所有流量，cloudflared 的 QUIC/UDP 流量经过 TUN 后可能被错误处理
5. **重启软路由是最有效的恢复手段**：当 Clash 配置复杂且难以定位时，重启软路由可恢复所有网络连接
6. **CF 缓存加剧问题**：530 期间 CF 还缓存了错误页面（age: 5568），恢复后需要加 `?v=N` 绕过缓存
