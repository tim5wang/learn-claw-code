# 06 - 核心架构设计

> **深度源码分析** - 安全、记忆、工具调用等核心模块  
> **更新时间:** 2026-04-01

---

## 📐 源码目录结构

### Python 版本 (main/src/)

```
src/
├── assistant/          # Agent 助手核心逻辑
├── bootstrap/          # 系统引导和初始化
├── bridge/            # 与上游编辑器的桥接层
├── buddy/             # Buddy 模式实现
├── cli/               # 命令行交互界面
├── components/        # UI 组件
├── constants/         # 常量定义
├── coordinator/       # 任务协调器
├── entrypoints/       # 入口点管理
├── hooks/             # 钩子系统
├── keybindings/       # 键盘绑定
├── memdir/            # 记忆目录管理 ⭐
├── migrations/        # 数据迁移
├── moreright/         # 权限/合规检查
├── native_ts/         # TypeScript 原生绑定
├── outputStyles/      # 输出样式
├── plugins/           # 插件系统 ⭐
├── reference_data/    # 参考数据
├── remote/            # 远程运行时
├── schemas/           # 数据模式定义
├── screens/           # 屏幕管理
├── server/            # HTTP/SSE 服务器
├── services/          # 服务层
├── skills/            # 技能系统 ⭐
├── state/             # 状态管理 ⭐
├── types/             # 类型定义
├── upstreamproxy/     # 上游代理
├── utils/             # 工具函数
├── vim/               # Vim 集成
├── voice/             # 语音支持
└── [核心模块]
    ├── main.py              # 主入口
    ├── runtime.py           # 运行时核心 ⭐
    ├── tools.py             # 工具定义 ⭐
    ├── tool_pool.py         # 工具池管理 ⭐
    ├── task.py              # 任务管理
    ├── context.py           # 上下文管理 ⭐
    ├── history.py           # 会话历史 ⭐
    ├── session_store.py     # 会话存储 ⭐
    ├── permissions.py       # 权限系统 ⭐
    ├── cost_tracker.py      # 成本追踪
    ├── query_engine.py      # 查询引擎
    └── transcript.py        # 转录记录
```

---

## 🔒 安全架构设计

### 1. 权限系统 (`permissions.py`)

```python
# 权限检查流程
class PermissionSystem:
    - 文件访问权限验证
    - 命令执行白名单
    - 网络请求限制
    - 环境变量隔离
```

**核心安全机制:**

| 机制 | 说明 | 实现位置 |
|------|------|----------|
| **沙箱执行** | 限制命令执行范围 | `moreright/` |
| **文件访问控制** | 限制可访问的文件路径 | `permissions.py` |
| **成本限制** | Token 使用量监控 | `cost_tracker.py` |
| **会话隔离** | 不同项目会话独立 | `session_store.py` |
| **钩子审计** | 所有操作记录日志 | `hooks/` |

### 2. 合规检查 (`moreright/`)

```
moreright/
├── compliance.py      # 合规性检查
├── license_check.py   # 许可证验证
└── audit_log.py       # 审计日志
```

**设计原则:**
- ✅ Clean-room 实现，不复制专有源码
- ✅ 所有操作可追溯
- ✅ 符合开源许可证要求

---

## 🧠 记忆系统设计

### 1. 记忆目录 (`memdir/`)

```python
# 记忆存储结构
memdir/
├── sessions/          # 会话记忆
│   └── {session_id}/
│       ├── messages.json
│       ├── context.json
│       └── state.json
├── projects/          # 项目记忆
│   └── {project_id}/
│       ├── files.json
│       ├── decisions.json
│       └── todos.json
└── global/            # 全局记忆
    ├── preferences.json
    └── learnings.json
```

### 2. 会话存储 (`session_store.py`)

**核心功能:**
- 会话状态持久化
- 上下文压缩 (compaction)
- 历史消息检索
- 跨会话记忆共享

```python
class SessionStore:
    def save_session(session_id, state):
        # 保存会话状态
        
    def load_session(session_id):
        # 加载会话状态
        
    def compact_history(session_id):
        # 压缩历史记录，保留关键信息
        
    def search_memory(query, scope):
        # 跨会话搜索记忆
```

### 3. 上下文管理 (`context.py`)

**上下文层级:**
```
┌─────────────────────────────────┐
│     Global Context (全局)        │
│   - 用户偏好                     │
│   - 系统配置                     │
├─────────────────────────────────┤
│   Project Context (项目级)       │
│   - 项目结构                     │
│   - 文件索引                     │
├─────────────────────────────────┤
│    Session Context (会话级)      │
│   - 当前任务                     │
│   - 对话历史                     │
├─────────────────────────────────┤
│     Turn Context (单次交互)      │
│   - 当前消息                     │
│   - 工具调用结果                 │
└─────────────────────────────────┘
```

---

## 🔧 工具调用架构

### 1. 工具定义 (`tools.py`)

```python
# 工具接口定义
class Tool:
    name: str           # 工具名称
    description: str    # 工具描述
    parameters: dict    # 参数 schema
    execute: callable   # 执行函数
```

**内置工具分类:**

| 类别 | 工具示例 | 说明 |
|------|----------|------|
| **文件操作** | `read`, `write`, `edit` | 文件读写改 |
| **命令执行** | `exec`, `bash` | Shell 命令 |
| **网络请求** | `web_search`, `web_fetch` | 网页访问 |
| **代码分析** | `grep`, `search_code` | 代码搜索 |
| **Git 操作** | `git_status`, `git_commit` | 版本控制 |
| **系统工具** | `list_dir`, `file_info` | 系统查询 |

### 2. 工具池管理 (`tool_pool.py`)

```python
class ToolPool:
    - 工具注册表
    - 工具发现机制
    - 权限验证
    - 执行沙箱
    - 结果缓存
```

**工具调用流程:**
```
1. Agent 请求工具调用
       ↓
2. ToolPool 验证权限
       ↓
3. 执行沙箱检查
       ↓
4. 调用工具 execute()
       ↓
5. 记录审计日志
       ↓
6. 返回结果给 Agent
```

### 3. 技能系统 (`skills/`)

```
skills/
├── skill_registry.py   # 技能注册表
├── skill_discovery.py  # 技能发现
└── [具体技能]
    ├── coding.py       # 编程技能
    ├── debugging.py    # 调试技能
    └── testing.py      # 测试技能
```

**技能 vs 工具:**
- **工具**: 原子操作 (如读文件)
- **技能**: 复合操作 (如"修复 bug" = 读 + 分析 + 改 + 测试)

---

## 🔄 运行时架构

### 1. 运行时核心 (`runtime.py`)

```python
class Runtime:
    - 会话生命周期管理
    - Agent 循环执行
    - 工具调用协调
    - 状态持久化
```

**Agent 执行循环:**
```
┌──────────────┐
│  接收用户输入 │
└──────┬───────┘
       ↓
┌──────────────┐
│  构建上下文   │
└──────┬───────┘
       ↓
┌──────────────┐
│  调用 LLM     │
└──────┬───────┘
       ↓
┌──────────────┐
│  解析响应     │
└──────┬───────┘
       ↓
  ┌────┴────┐
  ↓         ↓
┌─────┐  ┌──────┐
│工具  │  │直接  │
│调用  │  │回复  │
└──┬──┘  └──────┘
   ↓
┌────────┐
│执行结果│
└───┬────┘
    ↓
┌────────┐
│ 循环继续│
└────────┘
```

### 2. 任务管理 (`task.py`, `tasks.py`)

```python
class Task:
    id: str
    description: str
    status: Enum[TODO, IN_PROGRESS, DONE]
    assigned_to: str
    dependencies: List[str]
```

**任务状态机:**
```
TODO → IN_PROGRESS → DONE
              ↓
          BLOCKED
```

### 3. 历史管理 (`history.py`)

**历史记录内容:**
- 所有用户消息
- 所有 Agent 响应
- 工具调用记录
- 系统事件

**压缩策略:**
- 保留关键决策点
- 合并相似消息
- 提取摘要存储

---

## 📊 数据流图

```
┌─────────────┐     ┌──────────────┐     ┌─────────────┐
│   用户输入   │────→│   CLI/Server │────→│   Runtime   │
└─────────────┘     └──────────────┘     └──────┬──────┘
                                                 │
                    ┌────────────────────────────┼────────────────────────────┐
                    ↓                            ↓                            ↓
             ┌───────────┐               ┌───────────┐               ┌───────────┐
             │  Context  │               │   Tools   │               │  Memory   │
             │  Manager  │               │   Pool    │               │   Store   │
             └─────┬─────┘               └─────┬─────┘               └─────┬─────┘
                   │                           │                           │
                   ↓                           ↓                           ↓
             ┌───────────┐               ┌───────────┐               ┌───────────┐
             │   Files   │               │  External │               │  Session  │
             │   System  │               │   APIs    │               │   Store   │
             └───────────┘               └───────────┘               └───────────┘
```

---

## 🔐 安全边界

```
┌─────────────────────────────────────────────────────────┐
│                    用户空间                              │
│  ┌───────────────────────────────────────────────────┐  │
│  │              Agent 沙箱                            │  │
│  │  ┌─────────────────────────────────────────────┐  │  │
│  │  │           工具执行层                         │  │  │
│  │  │  ┌───────────────────────────────────────┐  │  │  │
│  │  │  │         权限检查层                     │  │  │  │
│  │  │  └───────────────────────────────────────┘  │  │  │
│  │  └─────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────┐
│                    系统空间                              │
│         文件系统 / 网络 / 进程                           │
└─────────────────────────────────────────────────────────┘
```

---

[← 返回上一章](/claw-code/05-法律伦理.md) | [下一章 →](/claw-code/07-rust-rewrite.md)
