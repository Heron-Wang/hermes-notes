# Cloudflare Tunnel 自动健康检查：断连后 6 分钟自动恢复

> **日期**: 2026-08-08  
> **分类**: 运维实践  
> **标签**: Cloudflare, Tunnel, systemd, 运维  
> **来源**: hermes

---

## 问题

Cloudflare Tunnel 免费版偶发 QUIC 连接断连，导致外网访问返回 502/530 错误，需要手动 SSH 上去重启 cloudflared 才能恢复。

## 原因

CF Tunnel 底层用 QUIC 协议（UDP）维持长连接。网络抖动、运营商 UDP 限速、CF 边缘节点切换等都会导致 QUIC 连接断开。虽然 cloudflared 有自动重连机制，但有时重连失败后不会自动恢复，进程还在运行但隧道已失效。

## 解决方案

用 systemd timer 定时做外网健康检查，连续失败 3 次自动重启 cloudflared。

### 1. 健康检查服务

创建 /etc/systemd/system/cf-health.service，用 oneshot 类型执行 curl 检查 /health 端点。

### 2. 定时触发器

创建 /etc/systemd/system/cf-health.timer，OnCalendar=*:0/2 表示每 2 分钟触发一次。

### 3. 启用

systemctl daemon-reload & systemctl enable --now cf-health.timer

### 工作流程

每 2 分钟 curl /health 端点：成功则重置计数，失败则计数+1，连续 3 次失败（6 分钟）自动 restart cloudflared。

## 踩坑记录

| 坑 | 说明 |
|---|---|
| WatchdogSec 不生效 | cloudflared 不支持 sd_notify 信号 |
| cron 被 sudo 阻止 | 改用 systemd timer 更干净 |
| --noproxy 很重要 | 本地代理会干扰检测结果 |
| 用 /health 端点 | 比检测首页更高效 |

## 适用场景

- Cloudflare Tunnel 免费版用户
- 任何需要自动检测外网可达性并自动恢复的服务
