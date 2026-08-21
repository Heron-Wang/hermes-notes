# CLI-Anything 平台调研：让所有软件变成 AI Agent 可调用的原生接口

> **日期**: 2026-08-21  
> **分类**: 技术知识  
> **标签**: CLI-Anything, AI-Agent, HKUDS, CLI, 自动化  
> **来源**: hermes

---

## 背景/问题

在研究 AI Agent 生态时发现了 CLI-Anything 项目，这是一个将传统软件包装成 AI Agent 可调用接口的平台。需要了解其架构、生态和与 Hermes Agent 的关系。

## CLI-Anything 是什么

香港大学数据科学实验室（HKUDS）开发的开源项目，核心目标：**让所有软件变成 AI Agent 可调用的原生 CLI 接口**。

| 指标 | 数据 |
|------|------|
| 已生成 CLI 数量 | 100+（覆盖 35 个专业领域） |
| 总调用量 | 75 万+ |
| Agent vs 人类调用量 | Agent 是人类的 10 倍 |
| 测试通过率 | 2,461 项测试 100% 通过 |
| GitHub | https://github.com/HKUDS/CLI-Anything |
| 技术报告 | https://arxiv.org/abs/2606.03854 |
| CLI-Hub | https://clianything.cc |

## 支持的软件（83 个目录，按领域分）

| 领域 | 软件 |
|------|------|
| **3D/建模** | Blender, FreeCAD, 3MF, CloudCompare |
| **图像/设计** | GIMP, Inkscape, Krita, Sketch, DrawIO, Live2D |
| **视频/媒体** | OBS Studio, Kdenlive, Shotcut, Audacity, MuseScore, VideoCaptioner |
| **游戏引擎** | Godot, Slay the Spire II |
| **知识管理** | Zotero, Obsidian, Joplin, SiYuan, Calibre |
| **开发工具** | LLDB, RenderDoc, NSight, PM2, WireMock |
| **AI/ML** | Ollama, ComfyUI, ChromaDB, MiniMax |
| **浏览器** | Browser, Safari, SeaClip |
| **办公** | LibreOffice, Mermaid, OpenRefine |

## 标准 CLI 结构

```
软件名/
├── agent-harness/
│   ├── setup.py              # pip 安装
│   ├── cli_anything/软件名/
│   │   ├── __main__.py       # CLI 入口
│   │   ├── core/             # 核心功能模块
│   │   ├── utils/            # 工具函数
│   │   ├── tests/            # 单元测试 + E2E 测试
│   │   └── skills/SKILL.md   # Agent 技能文档
```

## 安装与使用

```bash
# 通过 CLI-Hub 安装
pip install cli-anything-hub
cli-hub install blender

# 或从源码安装
git clone https://github.com/HKUDS/CLI-Anything.git
cd CLI-Anything/blender/agent-harness
pip install .
```

## 核心理念

> 今天的软件是给人类用的 GUI，明天的"用户"是 AI Agent。CLI-Anything 把各种软件包装成标准 CLI，让 Agent 能像调用 API 一样操作 Blender、GIMP、OBS 等 100+ 软件。

下一阶段目标：Generate → Use → Test → Diagnose → Repair → Verify → Publish 自动化闭环。

## 与 Hermes Agent 的关系

CLI-Anything 的理念和 Hermes Agent 生态互补：
- Hermes 的 skills 系统类似 CLI-Anything 的 SKILL.md
- 可以通过 `cli-hub install` 为 Hermes 安装已有 CLI
- 也可以参考其 harness 结构，为自己的项目创建标准 CLI 接口

## 避坑提示

- 83 个目录中有部分是元数据/工具目录（如 `cli-hub`、`skills`、`docs`），实际可用 CLI 约 70+ 个
- README 中的 News 时间线显示项目更新非常频繁（几乎每天都有新 CLI 合并）
- Hermes skill 已被提出（#320 PR），说明社区正在推进与 Hermes 的集成
