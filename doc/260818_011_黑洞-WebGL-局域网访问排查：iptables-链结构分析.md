# 黑洞 WebGL 局域网访问排查：iptables 链结构分析

> **日期**: 2026-08-18  
> **分类**: 运维记录  
> **标签**: Linux, iptables, nftables, WebGL, 局域网, 排查  
> **来源**: hermes

---

## 背景/问题

blackhole 项目（Three.js 黑洞光线追踪器）部署在 NAS 的 8093 端口，通过 `python3 -m http.server 8093 --bind 0.0.0.0` 服务。局域网内其他设备访问 `http://192.168.0.6:8093` 时页面加载正常但看不到黑洞渲染效果（黑屏）。

## 排查过程

### 第一步：确认服务状态

```bash
# 服务监听情况
ss -tlnp sport = :8093
# LISTEN 0  5  0.0.0.0:8093  0.0.0.0:*  ← 正常，绑定了 0.0.0.0

# systemd 服务
systemctl cat blackhole.service
# ExecStart=/usr/bin/python3 -m http.server 8093 --bind 0.0.0.0

# 本机 curl 测试
curl -s -o /dev/null -w '%{http_code}' http://192.168.0.6:8093/
# 200  ← 服务端正常
```

### 第二步：检查防火墙

NAS 同时有 iptables 和 nftables 在工作。iptables 的 INPUT 链结构：

```
Chain INPUT (policy ACCEPT)     ← 总策略是 ACCEPT
  → jump LIBVIRT_INP           ← 虚拟网络规则（virbr0/1/2 的 DNS/DHCP）
  → jump UG_INPUT              ← UG（UGREEN NAS 系统）自定义链

Chain UG_INPUT (1 references)
  1. ACCEPT  lo                ← 允许本地回环
  2. ACCEPT  thunderbolt0/1    ← 允许雷电接口
  3. ACCEPT  ctstate RELATED,ESTABLISHED  ← 允许已建立连接
  4. ACCEPT  tcp dpt:5443      ← 允许 5443 端口
  （没有 DROP/REJECT 规则，流量回到 INPUT 链，policy ACCEPT）
```

nftables 规则：
```
table inet filter {
  chain input   { type filter hook input priority filter; policy accept; }
  chain forward { type filter hook forward priority filter; policy accept; }
  chain output   { type filter hook output priority filter; policy accept; }
}
```

结论：**防火墙没有拦截 8093 端口**。所有链的 policy 都是 ACCEPT，没有 DROP/REJECT 规则。

### 第三步：检查网络连通性

```bash
# ARP 表显示局域网设备都在
ip neigh show dev bridge0
# 192.168.0.103 lladdr 7e:51:44:ab:09:96 STALE    ← 用户 PC
# 192.168.0.104 lladdr 52:54:00:56:2f:b0 REACHABLE ← 软路由
# 192.168.0.1   lladdr 04:f9:f8:fc:7b:e4 REACHABLE  ← 网关

# rp_filter（反向路径过滤）
cat /proc/sys/net/ipv4/conf/bridge0/rp_filter  # 0 = 关闭，不会误拦
cat /proc/sys/net/ipv4/conf/all/rp_filter      # 0 = 关闭

# somaxconn
cat /proc/sys/net/core/somaxconn  # 4096，连接队列充足
```

### 第四步：确认问题在浏览器端

检查黑洞页面 HTML，发现 WebGL 错误检测逻辑：

```html
<div id="error-overlay">
  <h1>WebGL Initialization Failed</h1>
  <p id="error-message">Your browser does not support the required WebGL features...</p>
  <code id="error-detail"></code>
</div>
```

```javascript
// Three.js WebGLRenderer 初始化失败时显示错误覆盖层
this.renderer = new THREE.WebGLRenderer({...});
// 如果失败 → showError('WebGL Initialization Error', e.message)
```

**结论**：NAS 端一切正常，问题在浏览器端——WebGL 不被支持或被禁用。

## 解决方案

在访问设备的浏览器中检查 WebGL 支持：

1. **Chrome 地址栏输入 `chrome://gpu`**
   - 查看 WebGL 状态是否显示 "Hardware accelerated"
   - 如果显示 "Software only" 或 "Disabled"，需要开启 GPU 加速

2. **显卡驱动**
   - 没有独显或驱动未正确安装时，WebGL 可能无法初始化

3. **手机端**
   - 部分老旧手机浏览器 WebGL 支持差

## 经验总结

### iptables 链跳转结构

NAS（绿联 UG 系统）的 iptables 结构值得记录：

```
INPUT (policy ACCEPT)
  ├─ jump LIBVIRT_INP   (libvirt 虚拟网络: virbr0/1/2)
  └─ jump UG_INPUT      (UGREEN 自定义: lo/thunderbolt/5443)
```

关键：UG_INPUT 链没有 DROP/REJECT 结尾规则，流量会**回到调用链继续处理**，最终命中 INPUT 的 policy ACCEPT。如果 UG_INPUT 末尾有 DROP，则非 5443 端口的流量会被拦截。

### nftables 与 iptables 共存

Debian 12 同时有 iptables 和 nftables 在工作：
- `iptables -L` 显示 iptables 规则
- `nft list ruleset` 显示 nftables 规则
- 两者通过各自的 hook 生效，不冲突

### 排查局域网访问问题的步骤

1. `ss -tlnp` 确认服务监听地址和端口
2. `curl` 从本机测试服务是否正常
3. `iptables -L -n -v` + `nft list ruleset` 检查防火墙
4. `ip neigh show` 确认局域网设备可达
5. 检查 `rp_filter`（反向路径过滤可能误拦）
6. 确认浏览器端兼容性（WebGL、JS 版本等）

## 避坑提示

- **Headless 浏览器无法渲染 WebGL**：Hermes Agent 的 browser 工具是 headless 的，无 GPU 支持，无法渲染 WebGL 内容。验证 WebGL 页面必须在有 GPU 的真实浏览器中操作
- **`python3 -m http.server` 的 backlog 只有 5**：`ss` 显示 `LISTEN 0 5`，高并发时可能拒绝连接。正式部署应使用 gunicorn 或 Rust HTTP 服务器
- **iptables 链没有 DROP 不代表安全**：policy ACCEPT + 无 DROP 规则意味着所有端口都对外开放（局域网内）。生产环境应加默认 DROP 策略
