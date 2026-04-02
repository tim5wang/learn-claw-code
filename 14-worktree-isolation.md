# 14 - Worktree + 任务隔离机制深度分析

> **claw-code 任务隔离与上下文管理源码剖析** - 架构设计、源码位置、实现细节  
> **更新时间:** 2026-04-02

---

## 📋 概述

claw-code 采用 **Worktree + 会话隔离** 的双重机制来实现任务隔离：

1. **Worktree 隔离** - 基于 Git Worktree 的工作区隔离
2. **会话隔离** - 基于 Session 的上下文隔离
3. **沙箱隔离** - 基于文件系统/网络的运行时隔离

---

## 🏗️ 架构设计

### 三层隔离模型

```
┌─────────────────────────────────────────────────────────┐
│                    用户请求                              │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│              Layer 1: Worktree 隔离                      │
│                                                          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │  Worktree A  │  │  Worktree B  │  │  Worktree C  │  │
│  │  (任务 1)     │  │  (任务 2)     │  │  (任务 3)     │  │
│  │  /tmp/wt-a   │  │  /tmp/wt-b   │  │  /tmp/wt-c   │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  │
│                                                          │
│  目的：物理隔离不同任务的工作目录                          │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│              Layer 2: Session 隔离                       │
│                                                          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │  Session A   │  │  Session B   │  │  Session C   │  │
│  │  messages[]  │  │  messages[]  │  │  messages[]  │  │
│  │  tokens: 10K │  │  tokens: 15K │  │  tokens: 8K  │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  │
│                                                          │
│  目的：逻辑隔离不同会话的上下文历史                        │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│              Layer 3: Sandbox 隔离                       │
│                                                          │
│  ┌──────────────────────────────────────────────────┐  │
│  │  SandboxConfig                                    │  │
│  │  ├── filesystem_mode: workspace-only             │  │
│  │  ├── network_isolation: false                    │  │
│  │  ├── namespace_restrictions: true                │  │
│  │  └── allowed_mounts: ["/workspace", "/tmp"]      │  │
│  └──────────────────────────────────────────────────┘  │
│                                                          │
│  目的：限制工具执行的文件/网络访问范围                     │
└─────────────────────────────────────────────────────────┘
```

---

## 📂 源码位置索引

### Python 版本 (`src/`)

| 文件 | 功能 | 行数 | 关键内容 |
|------|------|------|---------|
| `src/session_store.py` | 会话存储 | ~30 | `StoredSession`, `save_session`, `load_session` |
| `src/context.py` | 上下文构建 | ~40 | `PortContext`, `build_port_context` |
| `src/runtime.py` | 运行时管理 | - | 主运行时逻辑 |
| `src/remote_runtime.py` | 远程运行时 | - | 远程执行支持 |

### Rust 版本 (`rust/crates/runtime/src/`)

| 文件 | 功能 | 行数 | 关键内容 |
|------|------|------|---------|
| `session.rs` | 会话管理 | ~280 | `Session`, `ConversationMessage`, `ContentBlock` |
| `sandbox.rs` | 沙箱配置 | ~200 | `SandboxConfig`, `SandboxRequest`, `FilesystemIsolationMode` |
| `compact.rs` | 上下文压缩 | - | 会话压缩逻辑 |
| `conversation.rs` | 对话管理 | - | 对话历史管理 |
| `config.rs` | 配置管理 | - | 全局配置加载 |
| `permissions.rs` | 权限控制 | - | 工具执行权限 |

---

## 🔍 核心机制详解

### 1. Session 会话隔离

#### Rust 实现 (`rust/crates/runtime/src/session.rs`)

**数据结构:**

```rust
// 会话结构
pub struct Session {
    pub version: u32,
    pub messages: Vec<ConversationMessage>,
}

// 消息结构
pub struct ConversationMessage {
    pub role: MessageRole,      // system/user/assistant/tool
    pub blocks: Vec<ContentBlock>,
    pub usage: Option<TokenUsage>,
}

// 内容块 (支持文本、工具调用、工具结果)
pub enum ContentBlock {
    Text { text: String },
    ToolUse {
        id: String,
        name: String,
        input: String,
    },
    ToolResult {
        tool_use_id: String,
        tool_name: String,
        output: String,
        is_error: bool,
    },
}
```

**会话持久化:**

```rust
// 源码位置：rust/crates/runtime/src/session.rs:75-85
impl Session {
    pub fn save_to_path(&self, path: impl AsRef<Path>) -> Result<(), SessionError> {
        fs::write(path, self.to_json().render())?;
        Ok(())
    }

    pub fn load_from_path(path: impl AsRef<Path>) -> Result<Self, SessionError> {
        let contents = fs::read_to_string(path)?;
        Self::from_json(&JsonValue::parse(&contents)?)
    }
}
```

**Python 版本 (`src/session_store.py`):**

```python
# 源码位置：src/session_store.py:10-25
@dataclass(frozen=True)
class StoredSession:
    session_id: str
    messages: tuple[str, ...]
    input_tokens: int
    output_tokens: int


DEFAULT_SESSION_DIR = Path('.port_sessions')


def save_session(session: StoredSession, directory: Path | None = None) -> Path:
    target_dir = directory or DEFAULT_SESSION_DIR
    target_dir.mkdir(parents=True, exist_ok=True)
    path = target_dir / f'{session.session_id}.json'
    path.write_text(json.dumps(asdict(session), indent=2))
    return path


def load_session(session_id: str, directory: Path | None = None) -> StoredSession:
    target_dir = directory or DEFAULT_SESSION_DIR
    data = json.loads((target_dir / f'{session_id}.json').read_text())
    return StoredSession(
        session_id=data['session_id'],
        messages=tuple(data['messages']),
        input_tokens=data['input_tokens'],
        output_tokens=data['output_tokens'],
    )
```

**隔离机制:**
- 每个会话独立存储为 JSON 文件
- 会话 ID 作为唯一标识符
- 消息历史完全隔离，不共享

---

### 2. Sandbox 沙箱隔离

#### Rust 实现 (`rust/crates/runtime/src/sandbox.rs`)

**沙箱配置:**

```rust
// 源码位置：rust/crates/runtime/src/sandbox.rs:10-25
#[derive(Debug, Clone, Serialize, Deserialize, PartialEq, Eq, Default)]
pub struct SandboxConfig {
    pub enabled: Option<bool>,
    pub namespace_restrictions: Option<bool>,
    pub network_isolation: Option<bool>,
    pub filesystem_mode: Option<FilesystemIsolationMode>,
    pub allowed_mounts: Vec<String>,
}

// 文件系统隔离模式
#[derive(Debug, Clone, Copy, Serialize, Deserialize, PartialEq, Eq, Default)]
#[serde(rename_all = "kebab-case")]
pub enum FilesystemIsolationMode {
    Off,                    // 无隔离
    #[default]
    WorkspaceOnly,          // 仅工作区
    AllowList,              // 白名单模式
}
```

**沙箱请求解析:**

```rust
// 源码位置：rust/crates/runtime/src/sandbox.rs:27-40
#[derive(Debug, Clone, Serialize, Deserialize, PartialEq, Eq, Default)]
pub struct SandboxRequest {
    pub enabled: bool,
    pub namespace_restrictions: bool,
    pub network_isolation: bool,
    pub filesystem_mode: FilesystemIsolationMode,
    pub allowed_mounts: Vec<String>,
}

impl SandboxConfig {
    pub fn resolve_request(
        &self,
        enabled_override: Option<bool>,
        namespace_override: Option<bool>,
        network_override: Option<bool>,
        filesystem_mode_override: Option<FilesystemIsolationMode>,
        allowed_mounts_override: Option<Vec<String>>,
    ) -> SandboxRequest {
        SandboxRequest {
            enabled: enabled_override.unwrap_or(self.enabled.unwrap_or(true)),
            namespace_restrictions: namespace_override
                .unwrap_or(self.namespace_restrictions.unwrap_or(true)),
            network_isolation: network_override.unwrap_or(self.network_isolation.unwrap_or(false)),
            filesystem_mode: filesystem_mode_override
                .or(self.filesystem_mode)
                .unwrap_or_default(),
            allowed_mounts: allowed_mounts_override.unwrap_or_else(|| self.allowed_mounts.clone()),
        }
    }
}
```

**沙箱状态:**

```rust
// 源码位置：rust/crates/runtime/src/sandbox.rs:50-65
#[derive(Debug, Clone, PartialEq, Eq)]
pub struct SandboxStatus {
    pub enabled: bool,
    pub requested: SandboxRequest,
    pub supported: bool,
    pub active: bool,
    pub namespace_supported: bool,
    pub namespace_active: bool,
    pub network_supported: bool,
    pub network_active: bool,
    pub filesystem_mode: FilesystemIsolationMode,
    pub filesystem_active: bool,
    pub allowed_mounts: Vec<String>,
    pub in_container: bool,
    pub container_markers: Vec<String>,
    pub fallback_reason: Option<String>,
}
```

**容器环境检测:**

```rust
// 源码位置：rust/crates/runtime/src/sandbox.rs:100-130
#[must_use]
pub fn detect_container_environment() -> ContainerEnvironment {
    let proc_1_cgroup = fs::read_to_string("/proc/1/cgroup").ok();
    detect_container_environment_from(SandboxDetectionInputs {
        env_pairs: env::vars().collect(),
        dockerenv_exists: Path::new("/.dockerenv").exists(),
        containerenv_exists: Path::new("/run/.containerenv").exists(),
        proc_1_cgroup: proc_1_cgroup.as_deref(),
    })
}

#[must_use]
pub fn detect_container_environment_from(
    inputs: SandboxDetectionInputs<'_>,
) -> ContainerEnvironment {
    let mut markers = Vec::new();
    
    // 检测 Docker
    if inputs.dockerenv_exists {
        markers.push("/.dockerenv".to_string());
    }
    
    // 检测 containerd
    if inputs.containerenv_exists {
        markers.push("/run/.containerenv".to_string());
    }
    
    // 检测环境变量
    for (key, value) in inputs.env_pairs {
        let normalized = key.to_ascii_lowercase();
        if matches!(
            normalized.as_str(),
            "container" | "docker" | "podman" | "kubernetes_service_host"
        ) && !value.is_empty()
        {
            markers.push(format!("env:{key}={value}"));
        }
    }
    
    // 检测 cgroup
    if let Some(cgroup) = inputs.proc_1_cgroup {
        for needle in ["docker", "containerd", "kubepods", "podman", "libpod"] {
            if cgroup.contains(needle) {
                markers.push(format!("/proc/1/cgroup:{needle}"));
            }
        }
    }
    
    ContainerEnvironment {
        in_container: !markers.is_empty(),
        markers,
    }
}
```

---

### 3. Context 上下文管理

#### Python 实现 (`src/context.py`)

```python
# 源码位置：src/context.py:10-30
@dataclass(frozen=True)
class PortContext:
    source_root: Path
    tests_root: Path
    assets_root: Path
    archive_root: Path
    python_file_count: int
    test_file_count: int
    asset_file_count: int
    archive_available: bool


def build_port_context(base: Path | None = None) -> PortContext:
    root = base or Path(__file__).resolve().parent.parent
    source_root = root / 'src'
    tests_root = root / 'tests'
    assets_root = root / 'assets'
    archive_root = root / 'archive' / 'claw_code_ts_snapshot' / 'src'
    return PortContext(
        source_root=source_root,
        tests_root=tests_root,
        assets_root=assets_root,
        archive_root=archive_root,
        python_file_count=sum(1 for path in source_root.rglob('*.py') if path.is_file()),
        test_file_count=sum(1 for path in tests_root.rglob('*.py') if path.is_file()),
        asset_file_count=sum(1 for path in assets_root.rglob('*') if path.is_file()),
        archive_available=archive_root.exists(),
    )
```

---

## 🔄 工作流程

### 任务执行流程

```
1. 用户发起请求
         │
         ▼
2. 创建/加载 Session
   (src/session_store.py 或
    rust/crates/runtime/src/session.rs)
         │
         ▼
3. 配置 Sandbox
   (rust/crates/runtime/src/sandbox.rs)
   ├── filesystem_mode: workspace-only
   ├── allowed_mounts: ["/workspace"]
   └── network_isolation: false
         │
         ▼
4. 构建 Context
   (src/context.py)
   ├── source_root
   ├── tests_root
   └── archive_root
         │
         ▼
5. 执行工具调用
   ├── 检查权限 (permissions.rs)
   ├── 验证路径 (sandbox.rs)
   └── 执行命令 (bash.rs)
         │
         ▼
6. 更新 Session
   ├── 添加消息
   ├── 记录 Token 使用
   └── 持久化存储
         │
         ▼
7. 返回结果给用户
```

---

## 🔐 安全机制

### 文件系统隔离

```rust
// 三种隔离模式
pub enum FilesystemIsolationMode {
    Off,            // 无限制，可访问任意文件
    WorkspaceOnly,  // 仅允许访问工作区
    AllowList,      // 仅允许白名单路径
}
```

**默认配置:**
```rust
SandboxConfig {
    enabled: Some(true),
    namespace_restrictions: Some(true),
    network_isolation: Some(false),
    filesystem_mode: Some(FilesystemIsolationMode::WorkspaceOnly),
    allowed_mounts: vec!["/workspace".to_string()],
}
```

### 权限检查流程

```rust
// 源码位置：rust/crates/runtime/src/permissions.rs
pub fn check_file_access(path: &Path, mode: FilesystemIsolationMode) -> bool {
    match mode {
        FilesystemIsolationMode::Off => true,
        FilesystemIsolationMode::WorkspaceOnly => {
            path.starts_with("/workspace")
        }
        FilesystemIsolationMode::AllowList => {
            ALLOWED_PATHS.iter().any(|p| path.starts_with(p))
        }
    }
}
```

---

## 📊 隔离效果对比

| 隔离层级 | 隔离对象 | 实现方式 | 性能开销 |
|---------|---------|---------|---------|
| **Worktree** | 工作目录 | Git Worktree | 低 |
| **Session** | 上下文历史 | JSON 文件存储 | 极低 |
| **Sandbox** | 文件/网络 | 路径检查 + 命名空间 | 中 |

---

## 🛠️ 配置示例

### Rust 版本配置

```rust
// rust/crates/runtime/src/config.rs
pub struct RuntimeConfig {
    pub session_dir: PathBuf,
    pub sandbox: SandboxConfig,
    pub max_context_tokens: u32,
    pub compaction_threshold: u32,
}

impl Default for RuntimeConfig {
    fn default() -> Self {
        Self {
            session_dir: PathBuf::from(".claw_sessions"),
            sandbox: SandboxConfig::default(),
            max_context_tokens: 100_000,
            compaction_threshold: 80_000,
        }
    }
}
```

### Python 版本配置

```python
# src/runtime.py (伪代码)
class RuntimeConfig:
    def __init__(self):
        self.session_dir = Path('.port_sessions')
        self.sandbox_enabled = True
        self.filesystem_mode = 'workspace-only'
        self.allowed_mounts = ['/workspace', '/tmp']
        self.max_context_tokens = 100000
```

---

## 📝 总结

### 关键设计原则

1. **分层隔离** - Worktree → Session → Sandbox 三层防护
2. **最小权限** - 默认仅允许访问工作区
3. **可配置性** - 支持多种隔离模式切换
4. **性能优先** - 轻量级 JSON 存储，低开销

### 源码位置汇总

| 功能 | Python | Rust |
|------|--------|------|
| **会话管理** | `src/session_store.py` | `rust/crates/runtime/src/session.rs` |
| **沙箱配置** | - | `rust/crates/runtime/src/sandbox.rs` |
| **上下文构建** | `src/context.py` | `rust/crates/runtime/src/conversation.rs` |
| **权限控制** | - | `rust/crates/runtime/src/permissions.rs` |
| **配置管理** | `src/runtime.py` | `rust/crates/runtime/src/config.rs` |

### 改进方向

1. **增强沙箱** - 考虑引入真正的容器隔离
2. **会话加密** - 敏感会话数据加密存储
3. **审计日志** - 记录所有文件访问操作
4. **动态权限** - 基于任务的动态权限分配

---

[← 返回上一章](/claw-code/13-agent-framework-comparison) | [下一章 →](/claw-code/15-mcp-protocol)
