# NAS 机械硬盘降噪方案：用 ionice 降低 I/O 优先级让硬盘安静下来

> **日期**: 2026-08-08  
> **分类**: 经验总结  
> **标签**: Linux, NAS, 硬件, ionice  
> **来源**: hermes

---

## 问题

NAS 用的是机械硬盘（HDD），晚上运行时磁头频繁寻道发出咔嗒声，严重影响睡眠。

## 原因

HDD 噪音的根源是磁头物理寻道。NAS 上有几个后台服务在持续读写硬盘：
- search_serv（文件搜索索引）
- index_serv（文件索引）
- thumb_core（缩略图生成）
- media_serv（媒体服务）
- systemd-journald（系统日志）
- rsyslogd（系统日志）

这些服务不断触发随机 I/O，导致磁头在盘片上频繁移动，产生机械噪音。

## 解决方案

核心思路：用 ionice -c 3（idle 策略）把进程的 I/O 优先级设为最低。idle 级别的 I/O 请求只在磁盘空闲时才执行，不会和正常工作争抢磁头时间。

### 1. 可以直接停掉的服务

systemctl stop search_serv index_serv thumb_core media_serv

这些服务停了不影响核心功能，需要时再启动。

### 2. 不能停但可以降优先级的服务

journald 和 rsyslogd 是系统关键服务，不能停，但可以降 I/O 优先级：

ionice -c 3 -p $(pgrep -f systemd-journald)
ionice -c 3 -p $(pgrep -f rsyslogd)

### 3. 封装成脚本

把开关逻辑封装到 hdd-quiet.sh：
- hdd-quiet.sh on：停索引服务 + ionice idle
- hdd-quiet.sh off：启索引服务 + ionice normal
- hdd-quiet.sh status：查看当前状态

### 4. 定时自动降噪

用 cron 或 Hermes 定时任务：
- 每天 23:00 自动开启降噪
- 每天 07:00 自动恢复正常

## 踩坑记录

| 坑 | 说明 |
|---|---|
| pgrep -x 匹配不到进程 | journald 进程名不精确匹配，改用 pgrep -f 模糊匹配 |
| cron 任务无法直接执行 sudo | 需要配置 SUDO_ASKPASS 或用 systemd timer 代替 |
| 降噪影响项目服务吗 | 不影响。项目服务 I/O 量很小，且 ionice idle 只在磁盘空闲时生效 |

## 适用场景

- 家用 NAS 夜间降噪
- 任何使用机械硬盘且需要降低噪音的场景
- SSD 不需要此方案（没有机械部件，不会产生寻道噪音）
