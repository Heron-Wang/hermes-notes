# DeepSeek Harness (dsh)：国产开源 AI 编码 Agent 介绍

> **日期**: 2026-08-21  
> **分类**: 技术知识  
> **标签**: DeepSeek, dsh, AI-Agent, 编码助手, 国产开源  
> **来源**: hermes

---

## 背景/问题

了解 CLI-Anything 生态时发现 DeepSeek Harness（dsh），需要搞清楚它是什么、与 Claude Code/Hermes Agent 的定位区别。

## DeepSeek Harness 是什么

DeepSeek Harness（简称 `dsh`）是 DeepSeek（深度求索）推出的 **AI Agent 终端工具**，与 Claude Code、OpenAI Codex、Hermes Agent 属于同一类别——运行在终端中的 AI 编码助手。

| 项目 | 说明 |
|------|------|
| **名称** | DeepSeek Harness（dsh） |
| **开发商** | DeepSeek（深度求索） |
| **类型** | 终端 AI Agent（CLI 编码助手） |
| **模型** | DeepSeek-V3 / DeepSeek-R1 |
| **定位** | 国产开源 AI 编码 Agent，对标 Claude Code |

## 主要功能

| 功能 | 说明 |
|------|------|
| **终端对话** | 在终端中与 AI 对话，执行编码任务 |
| **代码生成** | 自动生成、修改、重构代码 |
| **文件操作** | 读写文件、搜索代码、批量修改 |
| **命令执行** | 运行 shell 命令、构建、测试 |
| **插件系统** | 支持社区插件扩展功能 |
| **桌面客户端** | Windows/Linux 桌面版（bundled Node.js） |
| **宠物系统** | PetDex 动画宠物（与 Hermes/Claude Code 共享生态） |

## 生态组件

| 组件 | 说明 |
|------|------|
| **dsh CLI** | 核心命令行工具 |
| **awesome-dsh-plugin** | 社区插件列表（959⭐） |
| **dsh_desktop** | Windows 桌面客户端（498⭐） |
| **PetDex** | 动画宠物画廊（3926⭐，跨平台共享） |
| **open-design** | 设计插件（89454⭐） |

## 与同类工具对比

| 工具 | 模型 | 费用 | 开源 | 特点 |
|------|------|------|------|------|
| **Claude Code** | Claude 3.5/4 | 付费 | 否 | Anthropic 官方，功能最全 |
| **OpenAI Codex** | GPT-4/o3 | 付费 | 否 | OpenAI 官方 |
| **DeepSeek Harness** | DeepSeek-V3/R1 | **极低价** | 是 | 国产，性价比最高 |
| **Hermes Agent** | 任意模型 | 看模型 | 是 | 多平台，多模型支持 |

## 关键特点

1. **极低成本**：DeepSeek API 价格远低于 Claude/GPT，适合高频编码场景
2. **国产合规**：DeepSeek 是国内公司，数据不出境
3. **PetDex 共享**：宠物系统与 Hermes/Claude Code 跨平台共享
4. **插件生态活跃**：社区已有 959⭐ 的插件列表

## 对个人项目的价值

当前使用 Hermes Agent + CANNBot（cannbot.hicann.cn）作为 LLM 后端。dsh 的价值在于：
- 如果切换到 DeepSeek 模型，dsh 是官方工具，原生支持
- 插件生态可与 Hermes 共享（PetDex 已共享）
- 可作为 Hermes 的备选方案
