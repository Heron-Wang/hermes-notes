# PC 能 ping 通但浏览器访问不了：Clash 兜底规则导致国内网站走代理

> **日期**: 2026-08-19  
> **分类**: 问题排查  
> **标签**: Clash, 路由规则, ping, 浏览器, DIRECT, Match  
> **来源**: hermes

---

## 背景/问题

PC（192.168.0.106）能 `ping baidu.com` 成功（说明 DNS 解析和网络层正常），但浏览器访问 baidu.com 打不开。NAS 上 curl 直连 baidu.com 也超时（HTTP 000, 5s timeout）。

## 关键操作

### 1. 确认 DNS 和网络层正常

```bash
ping baidu.com   # 正常，能解析 IP 并收到回复
```

DNS 解析正常 → 问题不在 DNS。能 ping 通 → 网络层（ICMP）正常。

### 2. 测试直连 vs 代理

```bash
# 通过代理访问 baidu.com
curl -s --proxy http://192.168.0.104:7890 --proxy-user nikki:***REDACTED*** \
  https://www.baidu.com -o /dev/null -w "代理: %{http_code} %{time_total}s\n"
# 结果: HTTP 200, 0.08s ✅

# 直连 baidu.com（不走代理）
curl -s --noproxy '*' https://www.baidu.com \
  -o /dev/null -w "直连: %{http_code} %{time_total}s\n" --max-time 5
# 结果: HTTP 000, 5s timeout ❌
```

**关键发现**：通过代理可以访问，直连超时。说明 PC 的 HTTP/HTTPS 流量被路由到了代理，但 Clash 对 baidu.com 的路由规则没有走 DIRECT。

### 3. 检查 Clash 路由规则

```bash
curl -s -H "Authorization: Bearer ***REDACTED***" \
  "http://192.168.0.104:9090/rules" | python3 -c "
import sys, json
data = json.load(sys.stdin)
rules = data.get('rules', [])
# 查看最后10条规则（兜底规则）
print('=== 最后10条规则 ===')
for r in rules[-10:]:
    print(f'{r.get(\"type\"):20s} {r.get(\"payload\"):40s} -> {r.get(\"proxy\")}')
# 搜索 baidu 相关规则
print('=== baidu 相关规则 ===')
for r in rules:
    if 'baidu' in r.get('payload','').lower():
        print(f'{r.get(\"type\"):20s} {r.get(\"payload\"):40s} -> {r.get(\"proxy\")}')
"
```

输出：
```
=== 最后10条规则 ===
RuleSet   China-IP    -> 中国大陆网站
Match                 -> 主代理          ← 兜底规则

=== baidu 相关规则 ===
（空 — 没有 baidu 专属规则）
```

**根因找到了**：Clash 规则列表中没有 baidu.com 的 DIRECT 规则，也没有 GEOSITE 规则来自动识别国内网站。最后的兜底规则 `Match -> 主代理` 把所有未匹配的流量都发到了代理。

### 4. 为什么 ping 能通但浏览器不行

| 协议 | 端口 | Clash 处理 | 结果 |
|------|------|-----------|------|
| ICMP (ping) | — | 不拦截（Clash 只处理 TCP/UDP） | ✅ 直通 |
| HTTP/HTTPS | 80/443 | 匹配 Match 规则 → 走主代理 → 代理节点访问 baidu | ❌ 代理可能不允许访问国内网站 |

PC 的流量通过透明代理或系统代理被 Clash 拦截。ICMP 不被 Clash 处理所以 ping 正常，但 HTTP/HTTPS 被 Match 规则发送到海外代理节点，海外节点访问 baidu.com 可能被拒绝或超时。

## 解决方案

### 方案 A：添加 DIRECT 规则（推荐）

在 Clash 配置文件的 rules 部分添加国内域名直连规则：

```yaml
rules:
  - DOMAIN-SUFFIX,baidu.com,DIRECT
  - DOMAIN-SUFFIX,qq.com,DIRECT
  - DOMAIN-SUFFIX,bilibili.com,DIRECT
  # ... 其他国内域名
  # 或者使用 GEOIP 规则
  - GEOIP,CN,DIRECT
  - MATCH,主代理
```

### 方案 B：使用 RuleSet 规则集

Clash Premium 支持 RuleSet，可以引用外部规则集：

```yaml
rule-providers:
  china:
    type: http
    behavior: classical
    url: "https://raw.githubusercontent.com/.../china.yml"
    path: ./ruleset/china.yaml
    interval: 86400

rules:
  - RULE-SET,china,DIRECT
  - MATCH,主代理
```

### 方案 C：PC 绕过代理

在 PC 的浏览器/系统代理设置中，将 baidu.com 等国内网站加入不代理列表（No Proxy for）。

## 避坑提示

- **ping 正常不代表网络完全正常**：ping 用 ICMP 协议，大多数代理只处理 TCP/UDP，ICMP 直通。必须用 `curl` 测试 HTTP 层
- **Match 兜底规则是最后的防线**：所有未匹配的流量都走 Match 规则。如果 Match 指向代理，国内网站也会走代理
- **没有 GEOSITE/GEOIP 规则的 Clash 配置不完整**：缺少 GEOSITE,CN 和 GEOIP,CN 规则会导致国内网站无法直连
- **Clash 的 `allow-lan: true` + 透明代理模式**：局域网内 PC 的流量会被自动路由到 Clash，即使 PC 没有手动设置代理
- **curl --noproxy '*' 用于测试直连**：绕过所有代理设置，直接访问目标。这是排查代理问题的必备参数
