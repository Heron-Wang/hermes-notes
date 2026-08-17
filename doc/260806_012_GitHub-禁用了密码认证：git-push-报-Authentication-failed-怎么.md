# GitHub 禁用了密码认证：git push 报 Authentication failed 怎么办？

> **日期**: 2026-08-06  
> **分类**: 经验总结  
> **标签**: 无  
> **来源**: hermes

---

## 背景/问题

用 `git push` 推送代码时报错 `Authentication failed`，明明用户名和密码是对的。或者 `git clone` 一个私有仓库返回 404。这是因为 GitHub 在 2021 年 8 月后**彻底禁用了密码认证**，必须用 Personal Access Token（PAT）替代密码。

## 原因分析

GitHub 的认证方式变化：

| 方式 | 状态 | 说明 |
|------|------|------|
| 账号密码 | 已废弃 | 2021年8月后不再支持 HTTPS 密码认证 |
| Personal Access Token | 推荐 | 替代密码，可控制权限范围和有效期 |
| SSH Key | 推荐 | 适合长期使用，一次配置永久有效 |
| gh CLI | 最方便 | 自动管理认证，但需要安装 gh |

## "Repository not found" 不一定是仓库不存在

```bash
# 这个报错有两种可能：
git ls-remote https://github.com/xxx/yyy.git
remote: Repository not found.
fatal: repository not found

# 可能1：仓库真的不存在
# 可能2：仓库是私有的，你没认证 -> GitHub 不会告诉你"无权限"，而是假装不存在
```

**辨别方法**：加了 Token 认证后还是 404，才是真的不存在。加了 Token 就能访问，说明之前是认证问题。

## 解决方案

### 第一步：创建 Personal Access Token

1. 打开 https://github.com/settings/tokens
2. 点击 Generate new token (classic)
3. 填写信息：
   - Note：给 token 起个名字，如 my-server
   - Expiration：建议 90 天
   - Scopes 勾选：repo（完整仓库访问）
4. 点 Generate token
5. **立即复制** token（格式类似 ghp_xxx）—— 只显示一次！

### 第二步：配置 git 记住凭证

```bash
# 设置 credential helper 为 store
git config --global credential.helper store

# 设置 git 身份信息
git config --global user.name "你的名字"
git config --global user.email "你的邮箱"

# 手动写入凭证文件
printf 'https://你的用户名:***REDACTED***@github.com\n' > ~/.git-credentials
chmod 600 ~/.git-credentials  # 保护凭证文件权限

# 验证
git ls-remote https://github.com/你的用户名/任意仓库.git
# 如果不报 Authentication failed，就配置好了
```

### 第三步：用 Token 访问 GitHub API

```bash
# 验证 Token
curl -s -H "Authorization=***REDACTED*** 你的Token" https://api.github.com/user

# 查看仓库列表
curl -s -H "Authorization=***REDACTED*** 你的Token" \
  "https://api.github.com/users/你的用户名/repos?per_page=10"

# 检查某仓库是否可访问（含私有仓库）
curl -s -H "Authorization=***REDACTED*** 你的Token" \
  https://api.github.com/repos/owner/repo
```

## 避坑提示

- **Token 只显示一次**——生成后如果没复制，只能重新生成
- **credential.helper store 是明文存储**——凭证存在 ~/.git-credentials，务必 chmod 600。更高安全需求可用 cache 模式（内存缓存，有过期时间）
- **Token 有有效期**——过期后需要重新生成
- **git config user.name 和 Token 用户名不是一回事**——user.name 是 commit 作者名，Token 认证用的是 GitHub 账号用户名
- **Token 权限最小化**——日常开发只需要 repo scope
- **多账户场景**：用 SSH Key 加 ~/.ssh/config 配置不同 host 别名
