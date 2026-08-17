# CANNBot 模型配置原理与 OpenCode 迁移指南

> 本文档详细说明 CANNBot 在 OpenCode 中的模型配置原理、认证机制，以及如何将完整配置打包并迁移到另一台电脑。

---

## 1. 整体架构概述

CANNBot 并非一个独立的 API 端点，而是通过 **OpenCode 插件机制** 实现的一套定制化 Agent 配置。其核心组件包括：

```
┌─────────────────────────────────────────────────────────────┐
│                      OpenCode 主程序                         │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │            opencode.json (主配置文件)                │   │
│  │  { "plugin": ["file://.../cannbot-auth.js"] }       │   │
│  └──────────────────────┬──────────────────────────────┘   │
│                         │ 加载                              │
│                         ▼                                   │
│  ┌─────────────────────────────────────────────────────┐   │
│  │          cannbot-auth.js (认证插件)                  │   │
│  │                                                     │   │
│  │  ┌─────────────┐  ┌──────────────┐  ┌────────────┐ │   │
│  │  │ config hook │  │  auth hook   │  │ chat.headers│ │   │
│  │  │ 注册 Provider│  │ 定义认证方式  │  │ 注入鉴权头  │ │   │
│  │  └─────────────┘  └──────────────┘  └────────────┘ │   │
│  └──────────┬──────────────┬───────────────────────────┘   │
│             │              │                                │
│             ▼              ▼                                │
│  ┌────────────────┐  ┌────────────────┐                    │
│  │  auth.json     │  │  session.json  │                    │
│  │  虚拟密钥 VK    │  │  JWT Token     │                    │
│  └────────────────┘  └────────────────┘                    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. 核心文件清单

| # | 文件路径 | 用途 | 敏感性 |
|---|----------|------|--------|
| 1 | `~/.config/opencode/opencode.json` | 主配置文件，加载 cannbot-auth 插件 | 否 |
| 2 | `~/.config/opencode/plugins/cannbot-auth.js` | 认证插件，注册 Provider 并注入鉴权头 | 否 |
| 3 | `~/.local/share/opencode/auth.json` | 存储虚拟密钥（VK） | **敏感** |
| 4 | `~/.cannbot/session.json` | 存储 JWT 会话令牌 | **敏感** |
| 5 | `~/.config/opencode/skills/` | CANNBot 专属技能库（22 个技能目录） | 否 |
| 6 | `~/.config/opencode/built-in-skills-state.json` | 技能状态配置 | 否 |

> **注意**：文件 3 和 4 包含身份凭证，切勿上传至公开仓库。

---

## 3. 模型配置原理

### 3.1 API 网关

CANNBot 使用统一的 OpenAI 兼容接口网关：

```
https://cannbot.hicann.cn/gateway/compatible-mode/v1
```

该网关兼容 OpenAI API 格式，opencode 通过 `@ai-sdk/openai-compatible` SDK 进行调用。

### 3.2 模型列表获取

模型列表通过两种方式获取：

1. **动态拉取（优先）**：从 `https://cannbot.hicann.cn/cannbot/api/models/list` 实时获取可用模型列表
2. **硬编码备用**：当动态拉取失败时，回退到插件内置的模型列表

内置备用模型包括：

| 模型 ID | 名称 | 系列 |
|---------|------|------|
| `qwen-plus` | Qwen Plus | qwen |
| `qwen-max` | Qwen Max | qwen |
| `qwen-turbo` | Qwen Turbo | qwen |
| `qwen-plus-latest` | Qwen Plus Latest | qwen |
| `qwen-max-latest` | Qwen Max Latest | qwen |
| `qwen-turbo-latest` | Qwen Turbo Latest | qwen |
| `deepseek-v3` | DeepSeek V3 | deepseek |
| `deepseek-r1` | DeepSeek R1 | deepseek |

### 3.3 模型能力配置

所有模型统一配置如下：

```json
{
  "capabilities": {
    "temperature": true,
    "reasoning": true,
    "attachment": true,
    "toolcall": true,
    "input": { "text": true, "audio": false, "image": true, "video": false, "pdf": false },
    "output": { "text": true, "audio": false, "image": false, "video": false, "pdf": false }
  },
  "limit": {
    "context": 131072,
    "output": 8192
  },
  "cost": {
    "input": 0,
    "output": 0,
    "cache": { "read": 0, "write": 0 }
  }
}
```

| 参数 | 值 | 说明 |
|------|-----|------|
| 上下文窗口 | 131,072 tokens | 约 128K 上下文 |
| 最大输出 | 8,192 tokens | 约 8K 输出 |
| 费用 | 全部为 0 | 内部服务，不计费 |
| 图片输入 | 支持 | 可上传图片 |
| 工具调用 | 支持 | 支持 function calling |

### 3.4 Provider 注册流程

插件通过 `config` hook 向 opencode 注册 CANNBOT Provider：

```
opencode 启动
  → 加载 cannbot-auth.js 插件
    → 执行 config hook
      → 注册 provider "cannbot"
        → npm: "@ai-sdk/openai-compatible"
        → baseURL: "https://cannbot.hicann.cn/gateway/compatible-mode/v1"
        → models: 动态拉取 or 硬编码备用
```

---

## 4. 认证机制

### 4.1 双重认证

CANNBot 采用**双重认证**机制，每次 API 请求需同时携带两个认证头：

| 请求头 | 值 | 来源 | 有效期 |
|--------|-----|------|--------|
| `x-api-vkey` | `vk-xxxxxxxxxxxxx` | `~/.local/share/opencode/auth.json` | 长期有效 |
| `Authorization` | `Bearer <JWT>` | `~/.cannbot/session.json` | **24 小时** |

### 4.2 认证注入流程

```
用户发送消息
  → OpenCode 准备 API 请求
    → 触发 chat.headers hook
      → 从 auth.json 读取 VK → 注入 x-api-vkey 头
      → 从 session.json 读取 JWT → 注入 Authorization 头
    → 发送请求到 CANNBot 网关
```

### 4.3 Token 有效期与续期

| Token 类型 | 有效期 | 续期方式 |
|------------|--------|----------|
| JWT accessToken | 24 小时 | 通过 CANNBot VS Code 扩展自动续期 |
| 虚拟密钥 VK | 长期有效 | 无需续期，除非被手动撤销 |

**JWT 续期步骤**：

1. 打开 VS Code，确保 CANNBot 扩展已安装并登录
2. 扩展会自动检测 Token 过期并刷新 `~/.cannbot/session.json`
3. 如果未自动刷新，在扩展中退出登录并重新登录
4. 续期完成后重启 opencode

**验证 Token 状态**：

```powershell
node -e "const t = JSON.parse(require('fs').readFileSync(require('path').join(require('os').homedir(), '.cannbot', 'session.json'), 'utf-8')).accessToken; const p = JSON.parse(Buffer.from(t.split('.')[1], 'base64url').toString()); const exp = new Date(p.exp * 1000); console.log('过期时间:', exp.toLocaleString('zh-CN')); console.log('状态:', new Date() < exp ? '未过期' : '已过期')"
```

---

## 5. 插件工作原理详解

### 5.1 插件入口

`cannbot-auth.js` 使用 OpenCode 的 legacy 插件格式（直接导出异步函数），提供三个 Hook：

### 5.2 config Hook — Provider 注册

```javascript
config: async function (cfg) {
  cfg.provider["cannbot"] = {
    name: "CANNBOT",
    npm: "@ai-sdk/openai-compatible",
    options: { baseURL: "https://cannbot.hicann.cn/gateway/compatible-mode/v1" },
    models: await buildModelsDynamic(),  // 优先动态拉取，失败则回退
  };
}
```

### 5.3 auth Hook — 认证定义

```javascript
auth: {
  provider: "cannbot",
  methods: [{
    type: "api",
    label: "CANNBOT Virtual Key (VK)",
    async authorize(inputs) {
      const session = readSession();
      if (!session?.accessToken) return { type: "failed" };
      return { type: "success", key: inputs?.key ?? "" };
    },
  }],
  async loader(getAuth) {
    return {};  // 不返回 apiKey，由 chat.headers 手动注入
  },
}
```

### 5.4 chat.headers Hook — 鉴权头注入

```javascript
"chat.headers": function (input, output) {
  if (input.model.providerID !== "cannbot") return;

  // 注入虚拟密钥
  const auth = readAuthJson();
  const vk = auth["cannbot"]?.key;
  if (vk) output.headers["x-api-vkey"] = vk;

  // 注入 JWT
  const session = readSession();
  if (session?.accessToken) {
    output.headers["Authorization"] = `Bearer ${session.accessToken}`;
  }
}
```

---

## 6. Skills 技能库

CANNBot 自带 22 个专业技能，位于 `~/.config/opencode/skills/` 目录下：

| 技能目录 | 用途 |
|----------|------|
| `ascendc-api-best-practices` | AscendC API 最佳实践 |
| `ascendc-code-review` | AscendC 代码审查 |
| `ascendc-custom-op-to-kernel-launch` | 自定义算子转 Kernel Launch |
| `ascendc-direct-invoke-template` | Direct Invoke 模板 |
| `ascendc-docs-gen` | AscendC 文档生成 |
| `ascendc-docs-search` | AscendC 文档搜索 |
| `ascendc-env-check` | 环境检查 |
| `ascendc-kernel-develop-workflow` | Kernel 开发工作流 |
| `ascendc-npu-arch` | NPU 架构参考 |
| `ascendc-precision-debug` | 精度调试 |
| `ascendc-regbase-best-practice` | RegBase 最佳实践 |
| `ascendc-registry-invoke-to-direct-invoke` | Registry 转 Direct Invoke |
| `ascendc-runtime-debug` | 运行时调试 |
| `ascendc-st-design` | ST 设计 |
| `ascendc-task-focus` | 任务聚焦 |
| `ascendc-tiling-design` | Tiling 设计 |
| `ascendc-ut-develop` | UT 开发 |
| `ascendc-whitebox-design` | 白盒设计 |
| `cann-env-setup` | CANN 环境配置 |
| `ops-precision-standard` | 算子精度标准 |
| `ops-profiling` | 算子 Profiling |
| `ops-simulator` | 算子模拟器 |

---

## 7. 打包迁移方法

### 7.1 旧电脑打包

在源电脑上执行以下 PowerShell 命令：

```powershell
# ==============================
# CANNBot 配置打包脚本
# ==============================

$staging = "$env:TEMP\cannbot-migrate"
New-Item -ItemType Directory -Path $staging -Force

# 1. opencode 主配置
Copy-Item "$HOME/.config/opencode/opencode.json" "$staging/" -Force

# 2. cannbot-auth 插件
New-Item -ItemType Directory -Path "$staging/plugins" -Force
Copy-Item "$HOME/.config/opencode/plugins/cannbot-auth.js" "$staging/plugins/" -Force

# 3. auth.json（虚拟密钥）
$authDir = "$staging/auth"
New-Item -ItemType Directory -Path $authDir -Force
Copy-Item "$HOME/.local/share/opencode/auth.json" $authDir -Force

# 4. session.json（JWT Token）
$sessionDir = "$staging/cannbot"
New-Item -ItemType Directory -Path $sessionDir -Force
Copy-Item "$HOME/.cannbot/session.json" $sessionDir -Force

# 5. Skills 技能库
Copy-Item "$HOME/.config/opencode/skills" "$staging/skills" -Recurse -Force
Copy-Item "$HOME/.config/opencode/built-in-skills-state.json" "$staging/" -Force

# 打包
$zip = "$HOME\Desktop\cannbot-migrate.zip"
Compress-Archive -Path "$staging/*" -DestinationPath $zip -Force

# 清理临时目录
Remove-Item $staging -Recurse -Force

Write-Host "========================================="
Write-Host "打包完成: $zip"
Write-Host "========================================="
Write-Host ""
Write-Host "包含内容:"
Write-Host "  - opencode.json (主配置)"
Write-Host "  - plugins/cannbot-auth.js (认证插件)"
Write-Host "  - auth/auth.json (虚拟密钥 VK)"
Write-Host "  - cannbot/session.json (JWT Token)"
Write-Host "  - skills/ (22 个技能目录)"
Write-Host "  - built-in-skills-state.json (技能状态)"
Write-Host ""
Write-Host "请通过安全渠道传输此压缩包"
```

### 7.2 新电脑安装

确保新电脑已安装以下依赖：
- **Node.js**（opencode 插件运行依赖）
- **OpenCode**（`go install github.com/opencode-ai/opencode@latest` 或 `brew install opencode-ai/tap/opencode`）

然后执行以下命令：

```powershell
# ==============================
# CANNBot 配置还原脚本
# ==============================

# 输入压缩包路径
$zip = Read-Host "请输入 cannbot-migrate.zip 的完整路径"

# 解压到临时目录
$staging = "$env:TEMP\cannbot-migrate"
Expand-Archive -Path $zip -DestinationPath $staging -Force

# 1. 还原 opencode 主配置
New-Item -ItemType Directory -Path "$HOME/.config/opencode" -Force
Copy-Item "$staging/opencode.json" "$HOME/.config/opencode/" -Force

# 2. 还原 cannbot-auth 插件
New-Item -ItemType Directory -Path "$HOME/.config/opencode/plugins" -Force
Copy-Item "$staging/plugins/cannbot-auth.js" "$HOME/.config/opencode/plugins/" -Force

# 3. 还原 auth.json（虚拟密钥）
$authTarget = "$HOME/.local/share/opencode"
New-Item -ItemType Directory -Path $authTarget -Force
Copy-Item "$staging/auth/auth.json" $authTarget -Force

# 4. 还原 session.json（JWT Token）
$cannbotTarget = "$HOME/.cannbot"
New-Item -ItemType Directory -Path $cannbotTarget -Force
Copy-Item "$staging/cannbot/session.json" $cannbotTarget -Force

# 5. 还原 Skills 技能库
Copy-Item "$staging/skills" "$HOME/.config/opencode/skills" -Recurse -Force
Copy-Item "$staging/built-in-skills-state.json" "$HOME/.config/opencode/" -Force

# 清理
Remove-Item $staging -Recurse -Force

# 验证
Write-Host ""
Write-Host "========================================="
Write-Host "还原完成，正在验证文件..."
Write-Host "========================================="

$files = @(
    "$HOME/.config/opencode/opencode.json",
    "$HOME/.config/opencode/plugins/cannbot-auth.js",
    "$HOME/.local/share/opencode/auth.json",
    "$HOME/.cannbot/session.json"
)
$allOk = $true
foreach ($f in $files) {
    if (Test-Path $f) {
        Write-Host "[OK] $f" -ForegroundColor Green
    } else {
        Write-Host "[MISSING] $f" -ForegroundColor Red
        $allOk = $false
    }
}

if ($allOk) {
    Write-Host ""
    Write-Host "所有文件就位！运行 opencode 即可使用。" -ForegroundColor Green
} else {
    Write-Host ""
    Write-Host "部分文件缺失，请检查上面的错误信息。" -ForegroundColor Yellow
}
```

### 7.3 验证安装

安装完成后，运行以下命令验证：

```powershell
# 启动 opencode
opencode

# 在 TUI 中按 Ctrl+O 查看
# 应该能看到 CANNBOT provider 下的所有模型
```

---

## 8. 注意事项

### 8.1 Token 过期处理

| 场景 | 解决方案 |
|------|----------|
| JWT 过期（24 小时后） | 打开 VS Code 中的 CANNBot 扩展，重新登录续期 |
| 纯命令行环境无法续期 | 需定期在有 VS Code 的环境中刷新 Token |
| VK 失效 | 联系管理员重新生成虚拟密钥 |

### 8.2 安全须知

- **VK 等同于密码**：虚拟密钥是长期凭证，传输时务必使用加密渠道
- **不要上传到公开仓库**：`auth.json` 和 `session.json` 包含敏感凭证
- **建议添加 `.gitignore`**：如果在项目目录中存放配置，确保排除敏感文件

### 8.3 跨平台兼容性

| 平台 | 配置目录路径 |
|------|-------------|
| Windows | `C:\Users\<用户名>\.config\opencode\` |
| Linux | `~/.config/opencode/` |
| macOS | `~/.config/opencode/` |

> Windows 上的 `~` 等同于 `C:\Users\<用户名>`，PowerShell 中用 `$HOME` 表示。

### 8.4 故障排查

| 问题 | 排查方法 |
|------|----------|
| 模型列表为空 | 检查 JWT 是否过期，网络是否可达 |
| 401 Unauthorized | 检查 `auth.json` 和 `session.json` 是否存在且有效 |
| 插件未加载 | 检查 `opencode.json` 中 `plugin` 路径是否正确 |
| Skills 不可用 | 检查 `~/.config/opencode/skills/` 目录是否完整 |

---

## 9. 快速参考

### 关键 URL

| 用途 | URL |
|------|-----|
| API 网关 | `https://cannbot.hicann.cn/gateway/compatible-mode/v1` |
| 模型列表 | `https://cannbot.hicann.cn/cannbot/api/models/list` |

### 关键文件路径速查

```
~/.config/opencode/opencode.json                              # 主配置
~/.config/opencode/plugins/cannbot-auth.js                    # 认证插件
~/.local/share/opencode/auth.json                             # 虚拟密钥 VK
~/.cannbot/session.json                                       # JWT Token
~/.config/opencode/skills/                                    # 技能库
~/.config/opencode/built-in-skills-state.json                 # 技能状态
```

---

> **文档版本**：v1.0  
> **最后更新**：2026-07-24  
> **适用环境**：Windows / OpenCode + CANNBot 插件