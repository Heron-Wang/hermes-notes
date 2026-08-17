# 用 Cloudflare Tunnel 给每个子项目分配独立子域名

> **日期**: 2026-08-06  
> **分类**: 经验总结  
> **标签**: 无  
> **来源**: hermes

---

## 背景：多个 Web 项目怎么组织？

个人站上有多个独立项目：贪吃蛇游戏、Web 终端、性能测评平台。每个都是独立服务。怎么组织最合理？

## 推荐方案：子域名 + 注册表

每个项目独立运行在不同端口，通过子域名访问：

```
heronwang.cn          → 主站 :8080
snake.heronwang.cn    → 贪吃蛇 :8081
shell.heronwang.cn    → 终端 :8082
bench.heronwang.cn    → 测评 :8083
```

主站作品集页面只是一个目录索引，点「访问」跳转到对应子域名。

## 怎么加一个新项目？4 步

### 第 1 步：项目部署到本地端口

```bash
cd /home/heron/projects/myapp
PORT=8084 python3 server.py
```

### 第 2 步：创建子域名 DNS

```bash
~/bin/cloudflared tunnel route dns heron-web myapp.heronwang.cn
```

> 不用手动去 CF 面板加 DNS！这条命令直接搞定。

### 第 3 步：更新 Tunnel 配置

```yaml
# ~/.cloudflared/config.yml
ingress:
  - hostname: heronwang.cn
    service: http://localhost:8080
  - hostname: myapp.heronwang.cn    # 新增
    service: http://localhost:8084  # 新增
  - service: http_status:404
```

重启：`sudo systemctl restart cloudflared`

### 第 4 步：注册到主站作品集

```bash
curl -X POST https://heronwang.cn/api/portfolio \
  -H "X-API-Token=***REDACTED*** \
  -d '{"title":"我的应用","url":"https://myapp.heronwang.cn"}'
```

## 避坑提示

1. **config.yml 最后必须有兜底规则** `http_status:404`
2. **YAML 缩进要一致**，用空格不用 Tab
3. **每个项目独立 systemd service**，一个重启不影响其他
