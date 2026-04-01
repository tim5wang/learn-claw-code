# 09 - 上下文构造机制

> **LLM 请求时的记忆与上下文组装流程** - 短期记忆、长期记忆、压缩策略  
> **更新时间:** 2026-04-01

---

## 🧠 记忆系统架构

### 三层记忆模型

```
┌─────────────────────────────────────────────────────────┐
│                   工作记忆 (Working Memory)              │
│              当前会话的活跃上下文 (Token 限制内)            │
│                  容量：~100K tokens                      │
└─────────────────────────────────────────────────────────┘
                          ↑
                          │ 压缩/提取
┌─────────────────────────────────────────────────────────┐
│                   短期记忆 (Short-term Memory)           │
│           最近 N 轮对话 + 当前项目文件索引                 │
│                  容量：~10 轮对话                         │
└─────────────────────────────────────────────────────────┘
                          ↑
                          │ 摘要/索引
┌─────────────────────────────────────────────────────────┐
│                   长期记忆 (Long-term Memory)            │
│     历史会话摘要 + 项目决策记录 + 用户偏好 + 技能学习       │
│                  容量：无限 (磁盘存储)                     │
└─────────────────────────────────────────────────────────┘
```

---

## 📂 记忆存储结构

### 文件系统布局

```
memdir/
├── sessions/                     # 会话记忆
│   └── {session_id}/
│       ├── messages.json         # 完整消息历史
│       ├── context.json          # 上下文状态
│       ├── state.json            # 会话状态
│       └── compaction/           # 压缩记录
│           ├── summary_001.md    # 第 1 次压缩摘要
│           ├── summary_002.md    # 第 2 次压缩摘要
│           └── ...
│
├── projects/                     # 项目记忆
│   └── {project_id}/
│       ├── files.json            # 文件索引
│       ├── decisions.json        # 关键决策记录 ⭐
│       ├── todos.json            # 任务列表
│       └── learnings.json        # 项目经验 ⭐
│
└── global/                       # 全局记忆
    ├── preferences.json          # 用户偏好 ⭐
    ├── skills/                   # 技能学习
    │   ├── coding.json
    │   ├── debugging.json
    │   └── ...
    └── entities/                 # 实体识别
        ├── people.json
        └── concepts.json
```

---

## 🔄 上下文构造流程

### 完整流程图

```
┌──────────────┐
│  用户发送消息 │
└──────┬───────┘
       ↓
┌──────────────────────────────────────────────────────────┐
│  Step 1: 加载会话状态                                     │
│  - 从 session_store 加载当前会话                          │
│  - 读取最近 N 轮消息 (默认 10 轮)                            │
│  - 检查是否需要压缩                                       │
└──────┬───────────────────────────────────────────────────┘
       ↓
┌──────────────────────────────────────────────────────────┐
│  Step 2: 加载项目上下文                                    │
│  - 从 projects/{project_id}/files.json 读取文件索引        │
│  - 加载项目决策记录 (decisions.json)                      │
│  - 加载待办事项 (todos.json)                              │
└──────┬───────────────────────────────────────────────────┘
       ↓
┌──────────────────────────────────────────────────────────┐
│  Step 3: 加载全局记忆                                      │
│  - 用户偏好 (preferences.json)                           │
│  - 相关技能 (skills/)                                     │
│  - 实体信息 (entities/)                                   │
└──────┬───────────────────────────────────────────────────┘
       ↓
┌──────────────────────────────────────────────────────────┐
│  Step 4: 相关性评分                                        │
│  - 计算消息与历史上下文的相关性                            │
│  - 选择 Top-K 最相关的历史消息                              │
│  - 过滤低优先级内容                                       │
└──────┬───────────────────────────────────────────────────┘
       ↓
┌──────────────────────────────────────────────────────────┐
│  Step 5: Token 预算分配                                     │
│  ┌────────────────────────────────────────────────────┐  │
│  │  总预算：100,000 tokens                             │  │
│  │  ├─ 系统提示：2,000 tokens                          │  │
│  │  ├─ 项目上下文：10,000 tokens                       │  │
│  │  ├─ 历史消息：50,000 tokens                         │  │
│  │  ├─ 长期记忆摘要：10,000 tokens                     │  │
│  │  └─ 预留空间：28,000 tokens                         │  │
│  └────────────────────────────────────────────────────┘  │
└──────┬───────────────────────────────────────────────────┘
       ↓
┌──────────────────────────────────────────────────────────┐
│  Step 6: 构建最终 Prompt                                  │
│  [System Prompt]                                         │
│  [Project Context]                                       │
│  [Long-term Memory Summary]                              │
│  [Recent Conversation History]                           │
│  [Current User Message]                                  │
└──────┬───────────────────────────────────────────────────┘
       ↓
┌──────────────┐
│  调用 LLM API │
└──────────────┘
```

---

## 📊 核心代码分析

### 1. Context Manager (`context.py`)

```python
class ContextManager:
    """上下文管理器 - 负责加载和组装上下文"""
    
    def __init__(self, session_id: str, project_id: Optional[str] = None):
        self.session_id = session_id
        self.project_id = project_id
        self.session_store = SessionStore()
        self.memory_store = MemoryStore()
    
    async def build_context(
        self,
        user_message: str,
        max_tokens: int = 100000
    ) -> LLMContext:
        """构建 LLM 上下文"""
        
        # Step 1: 加载会话历史
        session = await self.session_store.load(self.session_id)
        recent_messages = session.get_recent_messages(limit=10)
        
        # Step 2: 加载项目上下文
        project_context = {}
        if self.project_id:
            project_context = await self._load_project_context(
                self.project_id
            )
        
        # Step 3: 加载长期记忆
        longterm_summary = await self._retrieve_relevant_memories(
            query=user_message,
            limit=5
        )
        
        # Step 4: 计算相关性评分
        scored_messages = self._score_messages(
            recent_messages,
            user_message
        )
        top_messages = scored_messages[:10]  # Top-10
        
        # Step 5: Token 预算分配
        budget = TokenBudget(max_tokens)
        budget.allocate("system", 2000)
        budget.allocate("project", 10000)
        budget.allocate("longterm", 10000)
        budget.allocate("history", 50000)
        budget.allocate("reserve", 28000)
        
        # Step 6: 构建最终上下文
        context = LLMContext()
        context.system_prompt = self._build_system_prompt()
        context.project_info = self._truncate(project_context, budget.get("project"))
        context.memory_summary = self._truncate(longterm_summary, budget.get("longterm"))
        context.history = self._build_history(top_messages, budget.get("history"))
        context.user_message = user_message
        
        return context
    
    def _score_messages(
        self,
        messages: List[Message],
        query: str
    ) -> List[ScoredMessage]:
        """计算消息相关性评分"""
        scored = []
        for msg in messages:
            score = self._compute_relevance(msg, query)
            scored.append(ScoredMessage(message=msg, score=score))
        return sorted(scored, key=lambda x: x.score, reverse=True)
    
    def _compute_relevance(
        self,
        message: Message,
        query: str
    ) -> float:
        """计算单条消息的相关性"""
        # 1. 词法相似度
        lexical_score = cosine_similarity(
            self._embed(message.content),
            self._embed(query)
        )
        
        # 2. 时间衰减 (越近越相关)
        time_decay = exp(-0.1 * message.age_hours)
        
        # 3. 重要性标记 (决策/TODO 权重更高)
        importance = 1.0
        if message.has_decision:
            importance = 1.5
        if message.has_todo:
            importance = 1.3
        
        return lexical_score * time_decay * importance
    
    async def _retrieve_relevant_memories(
        self,
        query: str,
        limit: int = 5
    ) -> List[Memory]:
        """从长期记忆中检索相关内容"""
        # 1. 语义搜索
        memories = await self.memory_store.search(
            query=query,
            limit=limit * 2  # 先多取一些
        )
        
        # 2. 重新排序
        scored = []
        for mem in memories:
            score = self._score_memory(mem, query)
            scored.append((mem, score))
        
        # 3. 返回 Top-K
        return [m for m, _ in sorted(scored, key=lambda x: x[1], reverse=True)[:limit]]
```

---

### 2. Session Store (`session_store.py`)

```python
class SessionStore:
    """会话存储 - 管理会话状态和历史"""
    
    def __init__(self, storage_path: str = "memdir/sessions"):
        self.storage_path = storage_path
    
    async def load(self, session_id: str) -> Session:
        """加载会话"""
        path = f"{self.storage_path}/{session_id}"
        
        # 加载消息历史
        messages = await self._load_messages(path)
        
        # 加载上下文状态
        context = await self._load_context(path)
        
        # 加载压缩记录
        compactions = await self._load_compactions(path)
        
        return Session(
            id=session_id,
            messages=messages,
            context=context,
            compactions=compactions
        )
    
    async def save(self, session: Session):
        """保存会话"""
        path = f"{self.storage_path}/{session.id}"
        
        # 检查是否需要压缩
        if session.should_compact():
            await self._compact(session)
        
        # 保存消息
        await self._save_messages(path, session.messages)
        
        # 保存状态
        await self._save_context(path, session.context)
    
    async def _compact(self, session: Session):
        """压缩会话历史"""
        # 1. 使用 LLM 生成摘要
        summary = await self._generate_summary(session.messages)
        
        # 2. 保存压缩记录
        await self._save_compaction(session.id, summary)
        
        # 3. 保留最近 N 轮，删除旧消息
        session.messages = session.messages[-10:]  # 保留最近 10 轮
        
        # 4. 将摘要作为第一条消息
        session.messages.insert(0, Message(
            role="system",
            content=f"[历史摘要] {summary}"
        ))
```

---

### 3. Memory Store (`memdir/`)

```python
class MemoryStore:
    """记忆存储 - 管理长期记忆"""
    
    def __init__(self, storage_path: str = "memdir"):
        self.storage_path = storage_path
        self.index = MemoryIndex()
    
    async def add(self, memory: Memory):
        """添加记忆"""
        # 1. 分类存储
        category = memory.category  # session/project/global
        
        # 2. 序列化
        data = memory.to_dict()
        
        # 3. 写入文件
        path = f"{self.storage_path}/{category}/{memory.id}.json"
        await self._write(path, data)
        
        # 4. 更新索引
        await self.index.add(memory)
    
    async def search(
        self,
        query: str,
        limit: int = 10,
        filters: Optional[Dict] = None
    ) -> List[Memory]:
        """搜索记忆"""
        # 1. 语义搜索
        query_embedding = self._embed(query)
        
        # 2. 向量相似度检索
        candidates = await self.index.similarity_search(
            query_embedding,
            limit=limit * 3
        )
        
        # 3. 应用过滤器
        if filters:
            candidates = [
                m for m in candidates
                if self._matches(m, filters)
            ]
        
        # 4. 重新排序 (结合时间衰减)
        scored = []
        for mem in candidates:
            score = self._score(mem, query)
            scored.append((mem, score))
        
        return [m for m, _ in sorted(scored, key=lambda x: x[1], reverse=True)[:limit]]
    
    async def get_project_memories(
        self,
        project_id: str,
        types: List[str] = None
    ) -> ProjectMemories:
        """获取项目记忆"""
        path = f"{self.storage_path}/projects/{project_id}"
        
        memories = ProjectMemories()
        
        if not types or "decisions" in types:
            memories.decisions = await self._load_json(
                f"{path}/decisions.json"
            )
        
        if not types or "todos" in types:
            memories.todos = await self._load_json(
                f"{path}/todos.json"
            )
        
        if not types or "learnings" in types:
            memories.learnings = await self._load_json(
                f"{path}/learnings.json"
            )
        
        return memories
```

---

### 4. Compaction Strategy (`compaction.py`)

```python
class CompactionStrategy:
    """压缩策略 - 决定何时以及如何压缩"""
    
    def __init__(self):
        self.token_counter = TokenCounter()
        self.llm = LLMClient()
    
    def should_compact(self, session: Session) -> bool:
        """判断是否需要压缩"""
        # 策略 1: Token 数超过阈值
        total_tokens = self.token_counter.count(session.messages)
        if total_tokens > 80000:  # 阈值的 80%
            return True
        
        # 策略 2: 消息轮数超过限制
        if len(session.messages) > 50:
            return True
        
        # 策略 3: 会话时长超过 2 小时
        if session.duration_hours > 2:
            return True
        
        return False
    
    async def compact(self, session: Session) -> CompactionResult:
        """执行压缩"""
        # 1. 选择要压缩的消息范围
        messages_to_compact = session.messages[:-10]  # 保留最近 10 轮
        
        # 2. 构建压缩请求
        prompt = self._build_compaction_prompt(messages_to_compact)
        
        # 3. 调用 LLM 生成摘要
        response = await self.llm.chat(
            messages=[{"role": "user", "content": prompt}],
            max_tokens=2000
        )
        
        # 4. 解析摘要
        summary = self._parse_summary(response.content)
        
        # 5. 提取关键信息
        decisions = self._extract_decisions(messages_to_compact)
        todos = self._extract_todos(messages_to_compact)
        
        return CompactionResult(
            summary=summary,
            decisions=decisions,
            todos=todos,
            preserved_messages=session.messages[-10:]
        )
    
    def _build_compaction_prompt(
        self,
        messages: List[Message]
    ) -> str:
        """构建压缩提示"""
        return f"""请总结以下对话历史，提取关键信息：

要求：
1. 用 300-500 字概括主要讨论内容
2. 列出所有重要决策 (格式：【决策】xxx)
3. 列出所有待办事项 (格式：【TODO】xxx)
4. 记录技术要点和解决方案
5. 保持关键代码片段和文件名

对话历史：
{self._format_messages(messages)}
"""
```

---

## 🎯 Token 预算分配策略

### 动态分配算法

```python
class TokenBudget:
    """Token 预算管理"""
    
    def __init__(self, total: int = 100000):
        self.total = total
        self.allocated = {}
        self.used = {}
    
    def allocate(self, category: str, tokens: int):
        """分配预算"""
        self.allocated[category] = tokens
    
    def get(self, category: str) -> int:
        """获取剩余预算"""
        return self.allocated.get(category, 0) - self.used.get(category, 0)
    
    def adjust(self, category: str, delta: int):
        """动态调整"""
        self.allocated[category] += delta
    
    @staticmethod
    def smart_allocate(
        total: int,
        history_length: int,
        has_project: bool,
        has_longterm: bool
    ) -> Dict[str, int]:
        """智能分配"""
        budget = {
            "system": 2000,  # 固定
            "reserve": 20000  # 固定预留
        }
        
        remaining = total - budget["system"] - budget["reserve"]
        
        # 项目上下文优先
        if has_project:
            budget["project"] = min(15000, remaining * 0.2)
            remaining -= budget["project"]
        else:
            budget["project"] = 0
        
        # 长期记忆次之
        if has_longterm:
            budget["longterm"] = min(10000, remaining * 0.15)
            remaining -= budget["longterm"]
        else:
            budget["longterm"] = 0
        
        # 剩余给历史消息
        budget["history"] = remaining
        
        return budget
```

---

## 📈 上下文构造示例

### 实际 Prompt 结构

```markdown
# System Prompt (2,000 tokens)
你是一个专业的编程助手，擅长代码分析和开发建议。
用户偏好：使用中文回复，简洁风格，优先 Rust 方案。

# Project Context (10,000 tokens)
## 项目信息
- 名称：hiclaw-fs
- 类型：AI Agent 文件系统
- 主要语言：Python

## 关键决策
【决策 2026-04-01】使用 Caddy 而非 nginx 部署静态网站
【决策 2026-04-01】文档网站采用 Docsify 而非 VitePress

## 待办事项
[~] 完成 frpc 配置
[ ] 监控代理稳定性

# Long-term Memory Summary (10,000 tokens)
## 相关记忆
### 记忆 #001 - Docker 挂载问题
时间：2026-04-01 10:30
内容：Docker bind mount 在容器生命周期内不会更新文件内容，
      解决方案是重新创建容器或使用 tar 复制。

### 记忆 #002 - frpc 代理冲突
时间：2026-04-01 11:00
内容：frps 服务器端会话残留会导致"proxy already exists"错误，
      需要等待会话超时或使用唯一代理名称。

# Conversation History (50,000 tokens)
[历史摘要] 用户请求部署 claw-code 研究文档网站，
已完成 Docsify 部署和 frpc 配置...

User: 帮我检查一下 frpc 的配置
Assistant: 好的，让我检查 frpc 配置文件...
[...最近 10 轮对话...]

# Current Message
User: 他每次请求大语言模型时的上下文是怎么从长期短期记忆里面构造出来的？
```

---

## 🔑 关键技术点

### 1. 记忆检索策略

| 策略 | 说明 | 实现 |
|------|------|------|
| **语义搜索** | 向量相似度检索 | `memory_store.search()` |
| **时间衰减** | 越近的记忆权重越高 | `exp(-0.1 * age_hours)` |
| **重要性标记** | 决策/TODO 权重更高 | `importance_multiplier` |
| **关键词匹配** | 精确匹配关键术语 | `keyword_boost` |

### 2. 压缩触发条件

```python
COMPACTION_TRIGGERS = {
    "token_threshold": 80000,      # Token 数超过 80K
    "message_count": 50,           # 消息超过 50 轮
    "duration_hours": 2,           # 会话超过 2 小时
    "manual": True                 # 支持手动触发
}
```

### 3. 上下文优先级

```
1. 系统提示 (必须)
2. 当前用户消息 (必须)
3. 项目关键决策 (高优先级)
4. 最近 3 轮对话 (高优先级)
5. 相关长期记忆 (中优先级)
6. 历史对话摘要 (中优先级)
7. 较远历史消息 (低优先级)
```

---

## 📊 性能优化

### 1. 缓存策略

```python
class ContextCache:
    """上下文缓存"""
    
    def __init__(self):
        self.cache = LRUCache(max_size=100)
    
    def get(self, session_id: str, query: str) -> Optional[LLMContext]:
        """获取缓存的上下文"""
        key = f"{session_id}:{hash(query)}"
        return self.cache.get(key)
    
    def set(self, session_id: str, query: str, context: LLMContext):
        """缓存上下文"""
        key = f"{session_id}:{hash(query)}"
        self.cache.set(key, context, ttl=300)  # 5 分钟过期
```

### 2. 增量更新

```python
async def update_context(
    old_context: LLMContext,
    new_message: Message
) -> LLMContext:
    """增量更新上下文，避免重新构建"""
    # 只更新变化的部分
    old_context.history.append(new_message)
    old_context.token_count += count_tokens(new_message)
    
    # 检查是否需要重新压缩
    if old_context.token_count > threshold:
        await compact(old_context)
    
    return old_context
```

---

[← 返回上一章](/claw-code/08-development-guide) | [下一章 →](/claw-code/10-最佳实践)
