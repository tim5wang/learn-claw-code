# 10 - Agent 循环机制

> **单次 Agent 循环的完整流程分析** - 消息处理、工具调用、状态转换  
> **更新时间:** 2026-04-01

---

## 🔄 Agent 循环概述

### 什么是 Agent 循环？

Agent 循环 (Agent Loop) 是 claw-code 的核心执行引擎，它负责：
1. 接收用户输入
2. 构造上下文
3. 调用 LLM
4. 解析响应
5. 执行工具调用
6. 更新状态
7. 决定继续或结束

这是一个**持续循环**的过程，直到任务完成或用户中断。

---

## 📊 完整流程图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         Agent 循环 (Agent Loop)                          │
│                                                                          │
│  ┌────────────┐                                                         │
│  │  空闲状态   │                                                         │
│  └─────┬──────┘                                                         │
│        │ 用户发送消息                                                    │
│        ↓                                                                  │
│  ┌────────────────────────────────────────────────────────────────────┐ │
│  │  Phase 1: 输入处理                                                  │ │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐             │ │
│  │  │  接收消息     │→ │  消息验证     │→ │  命令解析     │             │ │
│  │  └──────────────┘  └──────────────┘  └──────────────┘             │ │
│  └────────────────────────────────────────────────────────────────────┘ │
│        ↓                                                                  │
│  ┌────────────────────────────────────────────────────────────────────┐ │
│  │  Phase 2: 上下文构造                                                │ │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐             │ │
│  │  │  加载会话历史 │→ │  检索长期记忆 │→ │  Token 预算分配 │             │ │
│  │  └──────────────┘  └──────────────┘  └──────────────┘             │ │
│  │                              ↓                                       │ │
│  │                    ┌──────────────────┐                             │ │
│  │                    │   构建最终 Prompt │                             │ │
│  │                    └──────────────────┘                             │ │
│  └────────────────────────────────────────────────────────────────────┘ │
│        ↓                                                                  │
│  ┌────────────────────────────────────────────────────────────────────┐ │
│  │  Phase 3: LLM 调用                                                   │ │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐             │ │
│  │  │  发送请求     │→ │  流式接收     │→ │  响应解析     │             │ │
│  │  └──────────────┘  └──────────────┘  └──────────────┘             │ │
│  └────────────────────────────────────────────────────────────────────┘ │
│        ↓                                                                  │
│  ┌────────────────────────────────────────────────────────────────────┐ │
│  │  Phase 4: 响应处理                                                  │ │
│  │                                                                      │ │
│  │         ┌──────────────────────────────────────────────────┐        │ │
│  │         ↓                                                  │        │ │
│  │  ┌─────────────────┐                                       │        │ │
│  │  │ 有工具调用？     │── Yes ────────────────────────────────┤        │ │
│  │  └────────┬────────┘                                       │        │ │
│  │           │ No                                             │        │ │
│  │           ↓                                                │        │ │
│  │  ┌─────────────────┐                                       │        │ │
│  │  │ 直接文本回复     │                                       │        │ │
│  │  └─────────────────┘                                       │        │ │
│  │                                                              ↓        │ │
│  └────────────────────────────────────────────────────────────────────┘ │
│        ↓                                                                  │
│  ┌────────────────────────────────────────────────────────────────────┐ │
│  │  Phase 5: 工具执行 (如有)                                           │ │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐             │ │
│  │  │  权限验证     │→ │  执行工具     │→ │  收集结果     │             │ │
│  │  └──────────────┘  └──────────────┘  └──────────────┘             │ │
│  └────────────────────────────────────────────────────────────────────┘ │
│        ↓                                                                  │
│  ┌────────────────────────────────────────────────────────────────────┐ │
│  │  Phase 6: 状态更新                                                  │ │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐             │ │
│  │  │  记录消息     │→ │  更新上下文   │→ │  检查压缩     │             │ │
│  │  └──────────────┘  └──────────────┘  └──────────────┘             │ │
│  └────────────────────────────────────────────────────────────────────┘ │
│        ↓                                                                  │
│  ┌────────────────────────────────────────────────────────────────────┐ │
│  │  Phase 7: 循环决策                                                  │ │
│  │                                                                      │ │
│  │  ┌─────────────────┐                                               │ │
│  │  │ 需要继续执行？   │── Yes ──────────────────────────────────┐    │ │
│  │  └────────┬────────┘                                       │    │ │
│  │           │ No                                             │    │ │
│  │           ↓                                                │    │ │
│  │  ┌─────────────────┐                                       │    │ │
│  │  │ 结束/等待用户    │                                       │    │ │
│  │  └─────────────────┘                                       │    │ │
│  │           ↑                                                │    │ │
│  │           └────────────────────────────────────────────────┘    │ │
│  └────────────────────────────────────────────────────────────────────┘ │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 🔍 详细阶段分析

### Phase 1: 输入处理

```python
async def process_input(self, user_message: str) -> ProcessedInput:
    """处理用户输入"""
    
    # Step 1: 接收消息
    message = Message(
        role="user",
        content=user_message,
        timestamp=datetime.utcnow(),
        session_id=self.session_id
    )
    
    # Step 2: 消息验证
    if not self._validate_message(message):
        raise ValidationError("无效消息")
    
    # Step 3: 命令解析
    if user_message.startswith("/"):
        command = self._parse_command(user_message)
        return ProcessedInput(
            type=InputType.COMMAND,
            command=command,
            message=message
        )
    
    # Step 4: 特殊标记检测
    attachments = self._extract_attachments(user_message)
    mentions = self._extract_mentions(user_message)
    
    return ProcessedInput(
        type=InputType.MESSAGE,
        message=message,
        attachments=attachments,
        mentions=mentions
    )
```

**可能执行的操作:**
- ✅ 接收文本消息
- ✅ 解析 Slash 命令 (`/help`, `/status`, `/tools`)
- ✅ 提取附件 (文件路径、URL)
- ✅ 识别 @提及
- ✅ 消息验证 (长度、内容安全)

---

### Phase 2: 上下文构造

```python
async def build_context(
    self,
    processed_input: ProcessedInput
) -> LLMContext:
    """构建 LLM 上下文"""
    
    # Step 1: 加载会话历史
    session = await self.session_store.load(self.session_id)
    recent_messages = session.get_recent_messages(limit=10)
    
    # Step 2: 检索长期记忆
    memories = await self.memory_store.search(
        query=processed_input.message.content,
        limit=5
    )
    
    # Step 3: 加载项目上下文
    project_context = {}
    if self.project_id:
        project_context = await self._load_project_context()
    
    # Step 4: Token 预算分配
    budget = TokenBudget.smart_allocate(
        total=self.model_config.max_tokens,
        history_length=len(recent_messages),
        has_project=bool(self.project_id),
        has_longterm=len(memories) > 0
    )
    
    # Step 5: 构建最终 Prompt
    context = LLMContext()
    context.system_prompt = self._build_system_prompt()
    context.project_info = self._truncate(project_context, budget["project"])
    context.memory_summary = self._truncate(memories, budget["longterm"])
    context.history = self._build_history(recent_messages, budget["history"])
    context.user_message = processed_input.message.content
    
    return context
```

**可能执行的操作:**
- ✅ 加载最近 N 轮对话
- ✅ 语义搜索长期记忆
- ✅ 加载项目文件索引
- ✅ 加载待办事项
- ✅ Token 预算分配
- ✅ 构建结构化 Prompt

---

### Phase 3: LLM 调用

```python
async def call_llm(self, context: LLMContext) -> LLMResponse:
    """调用 LLM API"""
    
    # Step 1: 构建请求
    request = ChatCompletionRequest(
        model=self.model_config.model,
        messages=[
            {"role": "system", "content": context.system_prompt},
            {"role": "user", "content": context.build_full_prompt()}
        ],
        max_tokens=context.remaining_tokens,
        temperature=self.model_config.temperature,
        stream=True,  # 流式响应
        tools=self.tool_pool.get_tool_specs()  # 工具定义
    )
    
    # Step 2: 发送请求
    response_stream = await self.llm_client.chat(request)
    
    # Step 3: 流式接收
    full_response = ""
    tool_calls = []
    
    async for chunk in response_stream:
        if chunk.choices[0].delta.content:
            full_response += chunk.choices[0].delta.content
            # 实时输出到 UI
            await self.ui.display_token(chunk.choices[0].delta.content)
        
        if chunk.choices[0].delta.tool_calls:
            tool_calls.extend(chunk.choices[0].delta.tool_calls)
    
    # Step 4: 响应解析
    return LLMResponse(
        content=full_response,
        tool_calls=tool_calls,
        usage=chunk.usage,
        finish_reason=chunk.choices[0].finish_reason
    )
```

**可能执行的操作:**
- ✅ 发送 Chat Completion 请求
- ✅ 流式接收 Token
- ✅ 实时显示到 UI
- ✅ 收集工具调用
- ✅ 记录 Token 使用量

---

### Phase 4: 响应处理

```python
async def process_response(self, response: LLMResponse) -> ActionResult:
    """处理 LLM 响应"""
    
    # 判断响应类型
    if response.tool_calls:
        # 有工具调用
        return await self._handle_tool_calls(response.tool_calls)
    elif response.content:
        # 纯文本回复
        return ActionResult(
            type=ActionType.REPLY,
            content=response.content
        )
    else:
        # 异常情况
        return ActionResult(
            type=ActionType.ERROR,
            error="Empty response"
        )
```

**分支决策:**
```
                    ┌─────────────────┐
                    │  LLM 响应到达     │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │ 有 tool_calls?   │
                    └────────┬────────┘
                             │
            ┌────────────────┼────────────────┐
            │ Yes            │            No  │
            ↓                │                ↓
    ┌───────────────┐        │        ┌───────────────┐
    │ 工具调用处理   │        │        │ 直接文本回复   │
    └───────────────┘        │        └───────────────┘
                             │
                    ┌────────▼────────┐
                    │ 有 finish_reason?│
                    └────────┬────────┘
                             │
            ┌────────────────┼────────────────┐
            │ stop           │ length        │
            ↓                │                ↓
    ┌───────────────┐        │        ┌───────────────┐
    │ 正常结束       │        │        │ 截断/继续      │
    └───────────────┘        │        └───────────────┘
```

---

### Phase 5: 工具执行

```python
async def execute_tool_call(
    self,
    tool_call: ToolCall
) -> ToolResult:
    """执行单个工具调用"""
    
    # Step 1: 解析工具信息
    tool_name = tool_call.function.name
    tool_args = json.loads(tool_call.function.arguments)
    
    # Step 2: 权限验证
    if not await self._check_permission(tool_name, tool_args):
        return ToolResult(
            success=False,
            error="Permission denied",
            requires_approval=True
        )
    
    # Step 3: 执行工具
    try:
        tool = self.tool_pool.get_tool(tool_name)
        result = await tool.execute(**tool_args)
        
        # Step 4: 收集结果
        return ToolResult(
            success=True,
            result=result,
            output=self._format_result(result)
        )
    
    except Exception as e:
        return ToolResult(
            success=False,
            error=str(e),
            output=None
        )
```

**可能执行的工具:**

| 工具类别 | 工具示例 | 说明 |
|---------|---------|------|
| **文件操作** | `read_file` | 读取文件内容 |
| | `write_file` | 写入文件 |
| | `edit_file` | 编辑文件 (diff) |
| **命令执行** | `run_command` | 执行 Shell 命令 |
| | `bash` | Bash 脚本执行 |
| **网络请求** | `web_search` | 搜索网络 |
| | `web_fetch` | 抓取网页 |
| **代码分析** | `search_code` | 代码搜索 |
| | `grep` | 文本搜索 |
| **Git 操作** | `git_status` | Git 状态 |
| | `git_commit` | Git 提交 |
| **系统工具** | `list_dir` | 列出目录 |
| | `file_info` | 文件信息 |

**工具执行流程:**
```
┌─────────────┐
│ 工具调用请求 │
└──────┬──────┘
       ↓
┌─────────────┐
│ 查找工具定义 │
└──────┬──────┘
       ↓
┌─────────────┐
│ 参数验证     │
└──────┬──────┘
       ↓
┌─────────────┐
│ 权限检查     │──→ 需要批准？→ 等待用户确认
└──────┬──────┘
       ↓
┌─────────────┐
│ 执行工具     │
└──────┬──────┘
       ↓
┌─────────────┐
│ 捕获输出     │
└──────┬──────┘
       ↓
┌─────────────┐
│ 返回结果     │
└─────────────┘
```

---

### Phase 6: 状态更新

```python
async def update_state(
    self,
    response: LLMResponse,
    tool_results: List[ToolResult]
):
    """更新会话状态"""
    
    # Step 1: 记录 LLM 响应
    assistant_message = Message(
        role="assistant",
        content=response.content,
        tool_calls=response.tool_calls,
        timestamp=datetime.utcnow()
    )
    self.session.messages.append(assistant_message)
    
    # Step 2: 记录工具结果
    for result in tool_results:
        tool_message = Message(
            role="tool",
            content=result.output,
            tool_call_id=result.tool_call_id,
            timestamp=datetime.utcnow()
        )
        self.session.messages.append(tool_message)
    
    # Step 3: 更新上下文
    self.context = await self.build_context(
        ProcessedInput(message=self.session.messages[-1])
    )
    
    # Step 4: 检查是否需要压缩
    if self.session.should_compact():
        await self.compact_session()
    
    # Step 5: 持久化
    await self.session_store.save(self.session)
```

**可能执行的操作:**
- ✅ 记录 Assistant 消息
- ✅ 记录 Tool 消息
- ✅ 更新 Token 计数
- ✅ 检查压缩条件
- ✅ 持久化到磁盘
- ✅ 更新 UI 状态

---

### Phase 7: 循环决策

```python
async def should_continue(self) -> bool:
    """判断是否继续循环"""
    
    # 条件 1: 达到最大迭代次数
    if self.iteration_count >= self.config.max_iterations:
        return False
    
    # 条件 2: LLM 明确表示完成
    if self.last_response.finish_reason == "stop":
        return False
    
    # 条件 3: 有待执行的工具调用
    if self.pending_tool_calls:
        return True
    
    # 条件 4: 用户中断
    if self.ui.user_requested_stop():
        return False
    
    # 条件 5: 错误状态
    if self.last_response.has_error:
        return False
    
    # 默认继续
    return True
```

**决策树:**
```
                         ┌─────────────────┐
                         │ 循环决策点       │
                         └────────┬────────┘
                                  │
         ┌────────────┬───────────┼───────────┬────────────┐
         │            │           │           │            │
         ↓            ↓           ↓           ↓            ↓
  ┌───────────┐ ┌───────────┐ ┌───────────┐ ┌───────────┐ ┌───────────┐
  │ 达到最大   │ │ LLM 说完成  │ │ 有待执行   │ │ 用户中断   │ │ 发生错误   │
  │ 迭代次数   │ │           │ │ 工具调用   │ │           │ │           │
  └─────┬─────┘ └─────┬─────┘ └─────┬─────┘ └─────┬─────┘ └─────┬─────┘
        │             │             │             │             │
        ↓             ↓             ↓             ↓             ↓
     NO 继续        NO 继续      YES 继续      NO 继续      NO 继续
```

---

## 📋 完整循环示例

### 场景：用户请求"帮我检查项目状态并提交更改"

```
┌─────────────────────────────────────────────────────────────────────────┐
│  Iteration 1                                                            │
├─────────────────────────────────────────────────────────────────────────┤
│  User: 帮我检查项目状态并提交更改                                        │
│                                                                         │
│  → Phase 1: 输入处理                                                     │
│    - 接收消息                                                            │
│    - 无命令解析                                                          │
│                                                                         │
│  → Phase 2: 上下文构造                                                   │
│    - 加载最近 10 轮对话                                                    │
│    - 加载项目文件索引                                                    │
│    - Token 预算：100K                                                    │
│                                                                         │
│  → Phase 3: LLM 调用                                                      │
│    - 调用 hiclaw-gateway/qwen3.5-plus                                   │
│    - 流式接收 Token                                                      │
│                                                                         │
│  → Phase 4: 响应处理                                                     │
│    - 检测到工具调用：git_status()                                        │
│                                                                         │
│  → Phase 5: 工具执行                                                     │
│    - 执行 git status                                                     │
│    - 返回：3 个文件修改                                                    │
│                                                                         │
│  → Phase 6: 状态更新                                                     │
│    - 记录消息                                                            │
│    - Token 使用：+2,500                                                  │
│                                                                         │
│  → Phase 7: 循环决策                                                     │
│    - 有待处理结果 → 继续                                                  │
└─────────────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────────────┐
│  Iteration 2                                                            │
├─────────────────────────────────────────────────────────────────────────┤
│  → Phase 3: LLM 调用 (基于工具结果)                                        │
│    - 输入：git status 结果                                               │
│                                                                         │
│  → Phase 4: 响应处理                                                     │
│    - 检测到工具调用：git_diff(files=[...])                               │
│                                                                         │
│  → Phase 5: 工具执行                                                     │
│    - 执行 git diff                                                       │
│    - 返回：diff 内容                                                     │
│                                                                         │
│  → Phase 6: 状态更新                                                     │
│    - Token 使用：+3,200                                                  │
│                                                                         │
│  → Phase 7: 循环决策                                                     │
│    - 继续                                                                │
└─────────────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────────────┐
│  Iteration 3                                                            │
├─────────────────────────────────────────────────────────────────────────┤
│  → Phase 3: LLM 调用                                                      │
│    - 分析 diff 内容                                                      │
│                                                                         │
│  → Phase 4: 响应处理                                                     │
│    - 检测到工具调用：git_commit(message="...")                           │
│                                                                         │
│  → Phase 5: 工具执行                                                     │
│    - 执行 git commit                                                     │
│    - 返回：commit hash                                                   │
│                                                                         │
│  → Phase 6: 状态更新                                                     │
│    - Token 使用：+2,800                                                  │
│                                                                         │
│  → Phase 7: 循环决策                                                     │
│    - LLM 表示任务完成 → 结束                                               │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 🔢 性能指标

### 典型循环性能

| 指标 | 数值 | 说明 |
|------|------|------|
| **单次循环耗时** | 2-10 秒 | 取决于 LLM 响应速度 |
| **Token 消耗** | 2,000-10,000 | 每轮对话 |
| **工具调用延迟** | 50-500ms | 本地工具 |
| **最大迭代次数** | 10-20 | 防止无限循环 |
| **上下文窗口** | 100K-200K | 模型限制 |

### Token 使用分布

```
┌─────────────────────────────────────────────────────────┐
│  Token 使用分布 (典型循环)                                │
│                                                          │
│  输入上下文  ████████████████████████████████░░  75%     │
│  LLM 输出     ████████████░░░░░░░░░░░░░░░░░░░░  25%     │
│                                                          │
│  总计：~8,000 tokens/轮                                   │
└─────────────────────────────────────────────────────────┘
```

---

## ⚠️ 异常处理

### 常见异常及处理

| 异常类型 | 原因 | 处理策略 |
|---------|------|---------|
| **LLM Timeout** | API 响应超时 | 重试 (最多 3 次) |
| **Tool Error** | 工具执行失败 | 记录错误，通知用户 |
| **Permission Denied** | 权限不足 | 请求用户批准 |
| **Context Overflow** | Token 超限 | 触发压缩 |
| **Rate Limit** | API 限流 | 退避等待 |

### 异常处理流程

```
┌─────────────┐
│  异常捕获    │
└──────┬──────┘
       ↓
┌─────────────┐
│  分类异常    │
└──────┬──────┘
       │
       ├──→ Retryable? ──Yes──→ 重试 (指数退避)
       │
       ├──→ User Approval? ──Yes──→ 等待确认
       │
       ├──→ Context Overflow? ──Yes──→ 触发压缩
       │
       └──→ Fatal ──→ 记录日志，通知用户
```

---

## 🎯 优化策略

### 1. 并行工具调用

```python
# 串行执行 (慢)
result1 = await tool1.execute()
result2 = await tool2.execute()
result3 = await tool3.execute()

# 并行执行 (快)
results = await asyncio.gather(
    tool1.execute(),
    tool2.execute(),
    tool3.execute()
)
```

### 2. 增量上下文更新

```python
# 完全重建 (慢)
context = await build_context(messages)

# 增量更新 (快)
context.append_message(new_message)
context.update_token_count()
```

### 3. 流式处理

```python
# 等待完整响应 (慢)
response = await llm.chat()
display(response)

# 流式显示 (快)
async for token in llm.chat_stream():
    display(token)  # 实时显示
```

---

[← 返回上一章](/claw-code/09-context-construction) | [下一章 →](/claw-code/10-best-practices)
