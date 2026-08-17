# SpiderFoot OSINT 工具：从邮箱出发的自动化情报收集

> **日期**: 2026-08-18  
> **分类**: 技术知识  
> **标签**: OSINT, SpiderFoot, 安全研究, 信息收集, Hunter.io  
> **来源**: hermes

---

## 背景/问题

想从一个邮箱地址（如 QQ 邮箱）出发，尽可能多地收集关联信息——QQ 号、社交媒体账号、数据泄露记录、关联域名等。手动逐个平台搜索效率太低，需要自动化工具。

## SpiderFoot 是什么

SpiderFoot 是开源的 OSINT（开源情报）自动化收集工具，Python 编写，通过 200+ 个模块从公开数据源并行收集目标信息。

| 特性 | 说明 |
|------|------|
| 200+ 模块 | 域名、IP、邮箱、用户名、漏洞等 |
| Web GUI + CLI | 浏览器界面和命令行两种方式 |
| 可视化关联 | 图谱展示数据关联关系 |
| API 集成 | Shodan、Censys、HaveIBeenPwned、VirusTotal、Hunter.io 等 |
| 报告导出 | JSON/CSV/HTML/GEXF |

GitHub: https://github.com/smicallef/spiderfoot

## 安装与使用

```bash
# 源码安装
git clone https://github.com/smicallef/spiderfoot.git
cd spiderfoot
pip3 install -r requirements.txt

# CLI 扫描（passive 模式，不接触目标）
python3 sf.py -s "target@example.com" -u passive -o json -q

# Web UI 启动
python3 sf.py -l 127.0.0.1:5001
# 浏览器访问 http://127.0.0.1:5001
```

## 实战案例：QQ 邮箱 OSINT 调查

以 `xxx@qq.com` 为目标，调查链路如下：

```
邮箱地址
  → 提取 QQ 号（邮箱 @ 前缀）
    → QQ 头像下载（https://q.qlogo.cn/g?b=qq&nk={QQ号}&s=640）
      → 头像大小判断活跃度（<3KB = 默认头像，>5KB = 自定义头像）
    → QQ 空间存在性检查（curl user.qzone.qq.com/{QQ号}）
  → GitHub Commits 搜索（api.github.com/search/commits?q={email}）
    → 从 commit author email 关联 GitHub 用户
    → 提取多仓库 commit 中的作者名变体
  → Hunter.io SMTP 验证（实际连接 MX 服务器确认邮箱存在）
  → 搜索引擎聚合（Bing/DuckDuckGo site: 搜索）
  → 中文平台（Gitee、知乎、微博、贴吧、B站、CSDN、掘金）
  → 国际平台（Reddit、Telegram、TikTok、Instagram、Keybase）
  → 数据泄露检查（HaveIBeenPwned）
```

### 关键发现技巧

1. **QQ 头像判断活跃度**：`curl https://q.qlogo.cn/g?b=qq&nk={QQ号}&s=640`，文件 >5KB 是自定义头像（活跃用户），<3KB 是默认头像
2. **GitHub commit email 关联**：GitHub 用户可能在 commit 中暴露真实邮箱（非 noreply.github.com）。搜索 `api.github.com/search/commits?q={email}` 可以跨仓库关联
3. **Hunter.io SMTP 验证**：不仅检查格式，还实际连接腾讯 SMTP 服务器确认邮箱存在且可收信。Free 计划 50 次/月
4. **作者名变体**：同一 GitHub 用户可能在不同仓库用不同 author name（如 leolulu-x15、surface@pro.3），这是关联匿名身份的关键线索

## 配置 API Key

SpiderFoot 的 API Key 配置在 SQLite 数据库中：

```bash
# 配置 Hunter.io API Key
cd ~/workspace/spiderfoot && source venv/bin/activate
python3 -c "
import sqlite3
conn = sqlite3.connect('spiderfoot.db')
c = conn.cursor()
c.execute('INSERT OR REPLACE INTO tbl_config (scope, opt, val) VALUES (?, ?, ?)',
          ('sfp_hunter', 'api_key', 'YOUR_HUNTER_API_KEY'))
conn.commit()
conn.close()
print('Hunter API Key configured')
"

# 验证模块工作
python3 sf.py -s "spotify.com" -m sfp_hunter -o json -q
# 应返回企业域名下的员工邮箱列表
```

## 免费可注册的 OSINT API 清单

| API | 免费额度 | 主要用途 |
|-----|----------|----------|
| HaveIBeenPwned | $3.95/月 | 邮箱泄露检查 |
| EmailRep | 50次/天 | 邮箱信誉评分 |
| Hunter | 50次/月 | 邮箱验证+关联 |
| Shodan | 100次/月 | 网络资产搜索 |
| VirusTotal | 500次/天 | 恶意软件检测 |
| Censys | 250次/月 | 主机搜索 |
| SecurityTrails | 50次/月 | DNS 历史记录 |
| AbuseIPDB | 1000次/天 | IP 黑名单 |

## 避坑提示

- **SpiderFoot 对中国平台覆盖有限**：716 个社交平台模块主要覆盖国际平台，QQ/微信/QQ空间需要手动 curl 检查
- **Hunter.io 只索引企业域名邮箱**：对 QQ 邮箱（个人邮箱）的 domain search 返回 0 条，但 email verifier 仍可验证 SMTP
- **GitHub API 搜索需 User-Agent header**：不加会被拒绝。`-H "User-Agent: Mozilla/5.0"`
- **GitHub commit search 需要 cloak-preview Accept header**：`-H "Accept: application/vnd.github.cloak-preview+json"`
- **被动扫描（passive）vs 主动扫描（active）**：passive 不接触目标服务器，只查公开数据源；active 会直接连接目标，可能被发现。调查个人邮箱用 passive 即可
- **QQ 头像 URL 格式**：`https://q.qlogo.cn/g?b=qq&nk={QQ号}&s={尺寸}`，尺寸可选 40/100/140/640

## 相关知识

### OSINT 调查的合法边界

OSINT 工具收集的都是公开数据源的信息。但要注意：
- 仅用于安全研究、渗透测试授权范围内
- 不得用于跟踪、骚扰个人
- 收集到的数据不得公开传播（特别是个人隐私信息）
- GDPR/个人信息保护法适用时需合规
