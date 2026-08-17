# Linux 服务用 systemd 管理实现开机自启和崩溃自动重启

> **日期**: 2026-08-06  
> **分类**: 经验总结  
> **标签**: 无  
> **来源**: hermes

---

## 背景：手动启动的服务重启后就没了

用 python3 server.py & 手动启动的服务，机器重启后全没了，进程崩溃了也没人管。

## 什么是 systemd？

Linux 的「服务管家」，帮你管：开机自启、崩溃重启、统一日志。

## 怎么把服务交给 systemd 管？

### 第 1 步：写 service 文件

```ini
# /etc/systemd/system/myapp.service
[Unit]
Description=我的Web服务
After=network.target

[Service]
Type=simple
User=heron
WorkingDirectory=/home/heron/myapp
ExecStart=/usr/bin/python3 /home/heron/myapp/server.py
Restart=always        # 关键！崩溃后总是重启
RestartSec=5          # 崩溃后等5秒再重启
Environment=PORT=8080

[Install]
WantedBy=multi-user.target
```

> **关键参数：**
> - `Restart=always`：不管崩溃还是被 kill 都自动重启
> - `RestartSec=5`：重启间隔别太快，不然一直崩溃重启会卡死
> - `After=network.target`：等网络好了再启动

### 第 2 步：安装并启动

```bash
sudo systemctl daemon-reload    # 改了service文件都要执行
sudo systemctl enable myapp     # 开机自启
sudo systemctl start myapp      # 现在启动
```

### 第 3 步：日常管理

```bash
sudo systemctl status myapp     # 查状态
sudo systemctl restart myapp    # 重启
sudo journalctl -u myapp -f     # 实时看日志
```

## 踩过的坑

### 坑1：sudo 需要密码无法自动化
**解决：** 用 SUDO_ASKPASS 指定密码脚本，配合 sudo -A。

### 坑2：改了 service 文件不生效
**原因：** systemd 缓存了旧配置。
**解决：** 每次修改后必须 daemon-reload。

### 坑3：PATH 不对找不到命令
**原因：** systemd 的 PATH 跟终端不一样。
**解决：** service 文件里显式指定 Environment=PATH=...
