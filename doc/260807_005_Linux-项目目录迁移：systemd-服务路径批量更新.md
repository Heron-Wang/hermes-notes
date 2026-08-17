# Linux 项目目录迁移：systemd 服务路径批量更新

> **日期**: 2026-08-07  
> **分类**: 经验总结  
> **标签**: 无  
> **来源**: hermes

---

## 背景/问题

服务器上有多个项目分散在不同目录，想统一到 `/home/heron/workspace/` 下方便管理。但这些项目都通过 **systemd 服务** 在后台运行，直接 `mv` 移动目录后，服务会找不到程序文件而崩溃。

## 原因分析

systemd 服务通过 `.service` 文件中的两个字段定位程序：

```ini
[Service]
WorkingDirectory=/home/heron/projects/airplane    # 工作目录
ExecStart=/usr/bin/python3 /home/heron/projects/airplane/server.py  # 启动命令
```

移动目录后，这些路径就指向了不存在的位置。而且 service 文件有**两份**：

1. 项目目录内的 `.service` 文件（源文件）
2. `/etc/systemd/system/` 下的安装副本（实际生效的）

两份都要改，漏一个都不行。

## 解决方案

### 第 1 步：创建新目录并移动项目

```bash
mkdir -p /home/heron/workspace

# 移动主站
mv /home/heron/web-service /home/heron/workspace/

# 移动所有项目（按需排除已废弃的）
mv /home/heron/projects/airplane /home/heron/projects/benchmark \
   /home/heron/projects/collabdocs /home/heron/projects/finance \
   /home/heron/projects/snake-rs /home/heron/projects/webshell \
   /home/heron/workspace/
```

### 第 2 步：找出所有引用旧路径的文件

```bash
# 搜索所有配置文件中的旧路径引用
grep -rl '/home/heron/web-service\|/home/heron/projects/' \
  /home/heron/workspace/ \
  --include='*.py' --include='*.sh' --include='*.service' \
  --include='*.toml' --include='*.json' --include='*.yaml' \
  --include='*.yml' --include='*.conf' --include='*.rs' \
  --include='*.go' 2>/dev/null
```

重点关注 `.service` 文件和启动脚本。

### 第 3 步：批量更新路径

```bash
# 更新项目目录内的 service 文件
cd /home/heron/workspace

# 主站（路径不同，单独处理）
sed -i 's|/home/heron/web-service|/home/heron/workspace/web-service|g' \
  web-service/web-service.service web-service/add-project.sh

# 其他项目（统一替换 projects → workspace）
for proj in airplane benchmark collabdocs finance snake-rs webshell; do
  sed -i 's|/home/heron/projects/|/home/heron/workspace/|g' \
    "$proj/$proj.service"
done

# 更新 systemd 安装的 service 文件（需要 sudo）
sudo sed -i 's|/home/heron/web-service|/home/heron/workspace/web-service|g' \
  /etc/systemd/system/web-service.service
sudo sed -i 's|/home/heron/projects/|/home/heron/workspace/|g' \
  /etc/systemd/system/{airplane,benchmark,collabdocs,finance,snake-rs,webshell}.service

# 重新加载 systemd 配置
sudo systemctl daemon-reload
```

### 第 4 步：重启所有服务并验证

```bash
# 重启所有服务
for svc in web-service airplane benchmark collabdocs finance snake-rs webshell; do
  sudo systemctl restart $svc
done

# 验证每个服务是否正常
for port in 8080 8086 8083 8085 8081 8082 8084; do
  echo -n "Port $port: "
  curl -s -o /dev/null -w "%{http_code}" http://localhost:$port/health 2>/dev/null \
    || echo "no health endpoint"
  echo ""
done
```

## 避坑提示

- **systemd 有两份 service 文件**：项目目录内一份（源文件），`/etc/systemd/system/` 下一份（实际运行的）。两份都要改，不然下次从项目目录重新安装 service 时又会覆盖回旧路径。
- **改完路径必须 `systemctl daemon-reload`**。systemd 会缓存 service 文件内容，不 reload 的话改了也不生效。
- **用 `|` 做 sed 分隔符**而不是 `/`，因为路径里有 `/`，用 `/` 做分隔符要转义很麻烦：`sed 's|old|new|g'` 比 `sed 's/\/home\/heron\/old/\/home\/heron\/new/g'` 清爽多了。
- **检查启动脚本里的路径**：除了 .service 文件，项目里的 `.sh` 脚本可能也硬编码了旧路径（比如 `add-project.sh` 里通过 `sys.path.insert` 引用主站路径），别忘了搜一下。
- **虚拟环境(.venv)里的路径**：Python venv 里的 shebang 和 direct_url.json 也包含旧路径，但一般不需要改——venv 的路径是绝对路径，移动后 venv 会失效，需要重建或用 `uv` 等工具修复。如果服务起不来，检查 journalctl 日志看是不是 venv 的问题。
- **移动前先停服务**：虽然 systemd 会自动重启，但移动正在运行的程序的目录可能导致不可预测的行为。安全做法是先 `systemctl stop` 再移动。
