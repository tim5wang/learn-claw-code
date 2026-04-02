# 🤖 Agent Harness 深度研究

> **从零到一构建 Agent Harness** - 基于 learn-claude-code 和 hermes-agent 的架构剖析

**最后更新:** 2026-04-02

---

## 📚 研究项目

本研究深入分析两个优秀的 Agent Harness 实现：

### 1. learn-claude-code ⭐⭐⭐⭐⭐

| 指标 | 值 |
|------|------|
| **GitHub** | [shareAI-lab/learn-claude-code](https://github.com/shareAI-lab/learn-claude-code) |
| **Stars** | 46,527+ |
| **Forks** | 7,348+ |
| **语言** | TypeScript |
| **定位** | Bash is all you need - 从零构建的 nano claude code |
| **特点** | 教学导向、12 个渐进式示例、完整文档 |

**核心内容:**
- ✅ Agent 循环 (s01)
- ✅ 工具使用 (s02)
- ✅ TODO 管理 (s03)
- ✅ 子 Agent (s04)
- ✅ Skill 加载 (s05)
- ✅ 上下文压缩 (s06)
- ✅ 任务系统 (s07)
- ✅ 后台任务 (s08)
- ✅ Agent 团队 (s09)
- ✅ 团队协议 (s10)
- ✅ 自主 Agent (s11)
- ✅ Worktree 隔离 (s12)

---

### 2. hermes-agent ⭐⭐⭐⭐⭐

| 指标 | 值 |
|------|------|
| **GitHub** | [NousResearch/hermes-agent](https://github.com/NousResearch/hermes-agent) |
| **Stars** | 21,697+ |
| **Forks** | 2,688+ |
| **语言** | Python |
| **定位** | The agent that grows with you |
| **特点** | 生产级、Acp 协议、上下文压缩、多模型支持 |

**核心内容:**
- ✅ ACP (Agent Communication Protocol)
- ✅ 上下文压缩器
- ✅ 提示词构建器
- ✅ 凭据池管理
- ✅ 多模型适配
- ✅ 流式输出
- ✅ Docker 部署
- ✅ 权限系统

---

## 📖 文档导航

### 🌟 核心章节

| 章节 | 主题 | learn-claude-code | hermes-agent |
|------|------|-------------------|--------------|
| **01** | [Agent 循环机制](/agent-harness/01-agent-loop) | s01_agent_loop.py | agent/prompt_builder.py |
| **02** | [工具系统](/agent-harness/02-tool-system) | s02_tool_use.py | acp_adapter/tools.py |
| **03** | [任务管理](/agent-harness/03-task-management) | s07_task_system.py | .plans/ |
| **04** | [子 Agent 与团队](/agent-harness/04-subagent-teams) | s04_subagent.py, s09_agent_teams.py | agent/ |
| **05** | [上下文管理](/agent-harness/05-context-management) | s06_context_compact.py | agent/context_compressor.py |
| **06** | [隔离机制](/agent-harness/06-isolation) | s12_worktree_task_isolation.py | - |

### 🔧 进阶主题

| 主题 | 描述 |
|------|------|
| **Skill 系统** | 动态加载和执行技能 |
| **后台任务** | 异步任务管理和监控 |
| **团队协议** | 多 Agent 协作协议 |
| **自主 Agent** | 自主决策和执行 |
| **ACP 协议** | Agent 通信协议标准 |

---

## 🏗️ 架构对比

### 整体架构

```
┌─────────────────────────────────────────────────────────┐
│              learn-claude-code                          │
│                                                          │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐              │
│  │  Agent   │  │  Tool    │  │  Skill   │              │
│  │  Loop    │  │  System  │  │  Loader  │              │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘              │
│       │             │             │                     │
│       └─────────────┴─────────────┘                     │
│                         │                               │
│                  ┌──────▼──────┐                        │
│                  │  Context    │                        │
│                  │  Manager    │                        │
│                  └──────┬──────┘                        │
│                         │                               │
│       ┌─────────────────┼─────────────────┐            │
│       │                 │                 │             │
│  ┌────▼────┐     ┌─────▼─────┐    ┌──────▼──────┐     │
│  │  Todo   │     │ Subagent  │    │  Worktree   │     │
│  │ System  │     │  System   │    │  Isolation  │     │
│  └─────────┘     └───────────┘    └─────────────┘     │
└─────────────────────────────────────────────────────────┘

特点：教学导向、渐进式学习、12 个独立示例


┌─────────────────────────────────────────────────────────┐
│                  hermes-agent                           │
│                                                          │
│  ┌──────────────────────────────────────────────────┐  │
│  │              ACP Protocol Layer                   │  │
│  └──────────────────────────────────────────────────┘  │
│                         │                               │
│  ┌─────────────────────▼────────────────────────────┐  │
│  │              Agent Core                           │  │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐       │  │
│  │  │ Prompt   │  │ Context  │  │  Model   │       │  │
│  │  │ Builder  │  │Compressor│  │ Adapter  │       │  │
│  │  └──────────┘  └──────────┘  └──────────┘       │  │
│  └───────────────────────────────────────────────────┘  │
│                         │                               │
│       ┌─────────────────┼─────────────────┐            │
│       │                 │                 │             │
│  ┌────▼────┐     ┌─────▼─────┐    ┌──────▼──────┐     │
│  │Credential│     │Permission │    │  Streaming  │     │
│  │   Pool   │     │  System   │    │   Output    │     │
│  └─────────┘     └───────────┘    └─────────────┘     │
└─────────────────────────────────────────────────────────┘

特点：生产级、协议驱动、完整部署方案
```

---

## 📊 关键特性对比

| 特性 | learn-claude-code | hermes-agent |
|------|-------------------|--------------|
| **语言** | TypeScript | Python |
| **定位** | 教学/学习 | 生产/部署 |
| **Agent 循环** | ✅ 显式实现 | ✅ 内置 |
| **工具系统** | ✅ 简单直观 | ✅ ACP 协议 |
| **子 Agent** | ✅ 独立示例 | ✅ 内置支持 |
| **上下文压缩** | ✅ 独立示例 | ✅ 生产级实现 |
| **任务隔离** | ✅ Worktree | ⚠️ 会话级 |
| **团队协议** | ✅ 独立示例 | ⚠️ 基础支持 |
| **权限系统** | ⚠️ 基础 | ✅ 完整 |
| **凭据管理** | ⚠️ .env | ✅ 凭据池 |
| **部署方案** | ⚠️ 本地 | ✅ Docker |
| **文档质量** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ |

---

## 🎯 学习目标

通过本研究，你将掌握：

1. **Agent 循环核心机制** - 理解 Agent 如何思考、决策、执行
2. **工具系统设计** - 如何设计可扩展的工具系统
3. **上下文管理** - Token 优化、压缩策略
4. **多 Agent 协作** - 子 Agent、团队协议
5. **任务隔离** - Worktree、会话隔离
6. **生产级考虑** - 权限、凭据、部署

---

## 📁 项目结构

```
agent-harness/
├── index.html              # Docsify 入口
├── README.md               # 本文件
├── _sidebar.md             # 侧边栏导航
├── 01-agent-loop.md        # Agent 循环机制
├── 02-tool-system.md       # 工具系统
├── 03-task-management.md   # 任务管理
├── 04-subagent-teams.md    # 子 Agent 与团队
├── 05-context-management.md # 上下文管理
├── 06-isolation.md         # 隔离机制
├── en/                     # 英文文档 (可选)
└── zh/                     # 中文文档 (可选)
```

---

## 🚀 快速开始

### 1. 阅读顺序建议

```
入门 → 01-Agent 循环 → 02-工具系统 → 03-任务管理
      ↓
进阶 → 04-子 Agent → 05-上下文 → 06-隔离
      ↓
实战 → 对比分析 → 最佳实践 → 部署指南
```

### 2. 代码学习

**learn-claude-code:**
```bash
git clone https://github.com/shareAI-lab/learn-claude-code
cd learn-claude-code
# 从 s01 开始逐步学习
python agents/s01_agent_loop.py
```

**hermes-agent:**
```bash
git clone https://github.com/NousResearch/hermes-agent
cd hermes-agent
docker-compose up
```

---

## 📚 相关资源

- [learn-claude-code 官方文档](https://github.com/shareAI-lab/learn-claude-code)
- [hermes-agent 官方文档](https://github.com/NousResearch/hermes-agent)
- [ACP 协议规范](https://github.com/NousResearch/hermes-agent/tree/main/acp_registry)
- [claw-code 研究](/claw-code/) - 另一个优秀的 Agent 实现

---

[开始学习 →](/agent-harness/01-agent-loop)

---

*最后更新：2026-04-02 | 研究持续进行中...*
