# 01 - Agent 循环机制

> **Agent 的心脏** - 理解 Agent 如何思考、决策、执行  
> **参考项目:** learn-claude-code (s01), hermes-agent (agent/)

---

## 📋 概述

Agent 循环 (Agent Loop) 是 Agent 系统的核心引擎，负责：

1. **接收输入** - 用户消息、工具结果
2. **构建上下文** - 历史消息、系统提示
3. **调用 LLM** - 发送请求、接收响应
4. **解析响应** - 判断工具调用或直接回复
5. **执行工具** - 调用工具、收集结果
6. **循环决策** - 继续循环或结束

```
┌─────────────────────────────────────────────────────────┐
│                    Agent Loop                           │
│                                                          │
│  ┌──────────┐                                           │
│  │  用户输入 │                                           │
│  └────┬─────┘                                           │
│       ↓                                                  │
│  ┌──────────────────────────────────────────────────┐  │
│  │  1. 构建上下文 (Build Context)                    │  │
│  │     - 系统提示                                    │  │
│  │     - 历史消息                                    │  │
│  │     - 工具定义                                    │  │
│  └──────────────────────────────────────────────────┘  │
│       ↓                                                  │
│  ┌──────────────────────────────────────────────────┐  │
│  │  2. 调用 LLM (Call LLM)                           │  │
│  │     - 发送请求                                    │  │
│  │     - 流式接收                                    │  │
│  └──────────────────────────────────────────────────┘  │
│       ↓                                                  │
│  ┌──────────────────────────────────────────────────┐  │
│  │  3. 解析响应 (Parse Response)                     │  │
│  │     - 判断：工具调用 or 直接回复？                 │  │
│  └──────────────────────────────────────────────────┘  │
│       ↓                                                  │
│       ├─────────────────┐                               │
│       ↓                 ↓                               │
│  ┌─────────┐    ┌─────────────┐                        │
│  │工具调用 │    │ 直接回复     │                        │
│  └────┬────┘    └──────┬──────┘                        │
│       ↓                ↓                                │
│  ┌─────────┐    ┌─────────────┐                        │
│  │执行工具 │    │ 返回给用户   │                        │
│  └────┬────┘    └─────────────┘                        │
│       ↓                                                  │
│  ┌─────────┐                                            │
│  │收集结果 │                                            │
│  └────┬────┘                                            │
│       ↓                                                  │
│  ┌─────────────────┐                                    │
│  │ 继续循环？       │──── No ────→ End                  │
│  └─────────────────┘                                    │
│       │ Yes                                              │
│       └──────────────────────────────────────────┘      │
└─────────────────────────────────────────────────────────┘
```

---

## 🔍 learn-claude-code 实现

### 源码位置

**文件:** `agents/s01_agent_loop.py`

**核心代码:**

```python
# agents/s01_agent_loop.py
import os
from anthropic import Anthropic
from dotenv import load_dotenv

load_dotenv()
client = Anthropic(api_key=os.getenv("ANTHROPIC_API_KEY"))

# 系统提示
SYSTEM_PROMPT = """You are a helpful AI assistant.
You can use tools to help the user.
Always think step by step."""

# Agent 循环
def agent_loop(user_message: str):
    messages = []
    
    # 1. 添加用户消息
    messages.append({
        "role": "user",
        "content": user_message
    })
    
    # 2. 循环执行
    while True:
        # 3. 调用 LLM
        response = client.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=1024,
            system=SYSTEM_PROMPT,
            messages=messages
        )
        
        # 4. 解析响应
        assistant_message = {
            "role": "assistant",
            "content": response.content
        }
        messages.append(assistant_message)
        
        # 5. 检查是否有工具调用
        tool_calls = [
            block for block in response.content 
            if block.type == "tool_use"
        ]
        
        if not tool_calls:
            # 没有工具调用，返回文本回复
            text_content = [
                block.text for block in response.content 
                if block.type == "text"
            ]
            return "".join(text_content)
        
        # 6. 执行工具
        for tool_call in tool_calls:
            tool_name = tool_call.name
            tool_input = tool_call.input
            tool_id = tool_call.id
            
            # 执行工具 (简化示例)
            tool_result = execute_tool(tool_name, tool_input)
            
            # 添加工具结果
            messages.append({
                "role": "user",
                "content": [{
                    "type": "tool_result",
                    "tool_use_id": tool_id,
                    "content": tool_result
                }]
            })
        
        # 7. 继续循环...

def execute_tool(name: str, input: dict):
    """执行工具的简化实现"""
    if name == "bash":
        import subprocess
        result = subprocess.run(
            input["command"], 
            shell=True, 
            capture_output=True, 
            text=True
        )
        return result.stdout
    return "Unknown tool"

# 使用示例
if __name__ == "__main__":
    response = agent_loop("列出当前目录的文件")
    print(response)
```

### 关键设计

**1. 消息格式:**
```python
messages = [
    {"role": "user", "content": "用户消息"},
    {"role": "assistant", "content": "助手回复"},
    {"role": "user", "content": [{"type": "tool_result", ...}]},
]
```

**2. 响应解析:**
```python
# 检查工具调用
tool_calls = [
    block for block in response.content 
    if block.type == "tool_use"
]

# 提取文本回复
text_content = [
    block.text for block in response.content 
    if block.type == "text"
]
```

**3. 循环终止条件:**
- 没有工具调用 → 返回文本
- 达到最大迭代次数 → 强制终止
- 错误发生 → 异常处理

---

## 🔍 hermes-agent 实现

### 源码位置

**文件:** 
- `agent/prompt_builder.py` - 提示词构建
- `agent/anthropic_adapter.py` - LLM 适配
- `acp_adapter/server.py` - ACP 服务器

**核心流程:**

```python
# agent/prompt_builder.py (简化)
class PromptBuilder:
    def __init__(self):
        self.system_prompt = ""
        self.messages = []
        self.tools = []
    
    def add_system(self, prompt: str):
        """添加系统提示"""
        self.system_prompt = prompt
    
    def add_message(self, role: str, content: str):
        """添加消息"""
        self.messages.append({
            "role": role,
            "content": content
        })
    
    def add_tool(self, tool_def: dict):
        """添加工具定义"""
        self.tools.append(tool_def)
    
    def build(self) -> dict:
        """构建最终请求"""
        return {
            "system": self.system_prompt,
            "messages": self.messages,
            "tools": self.tools if self.tools else None
        }


# agent/anthropic_adapter.py (简化)
class AnthropicAdapter:
    def __init__(self, api_key: str):
        self.client = Anthropic(api_key=api_key)
    
    async def chat(self, request: dict) -> Response:
        """调用 LLM"""
        response = self.client.messages.create(
            model=request.get("model", "claude-sonnet-4-20250514"),
            max_tokens=request.get("max_tokens", 1024),
            system=request.get("system"),
            messages=request.get("messages"),
            tools=request.get("tools")
        )
        return Response(
            content=response.content,
            usage=response.usage
        )


# acp_adapter/server.py (简化)
class ACPServer:
    def __init__(self):
        self.builder = PromptBuilder()
        self.adapter = AnthropicAdapter(...)
        self.sessions = {}
    
    async def handle_message(self, session_id: str, message: str):
        """处理消息"""
        # 1. 加载或创建会话
        session = self.sessions.get(session_id)
        if not session:
            session = self.create_session(session_id)
        
        # 2. 添加用户消息
        session.add_message("user", message)
        
        # 3. Agent 循环
        while True:
            # 4. 构建请求
            request = self.builder.build(session)
            
            # 5. 调用 LLM
            response = await self.adapter.chat(request)
            
            # 6. 解析响应
            if response.has_tool_calls():
                # 执行工具
                results = await self.execute_tools(response.tool_calls)
                session.add_tool_results(results)
                # 继续循环
                continue
            else:
                # 返回文本回复
                session.add_message("assistant", response.text)
                return response.text
```

### 关键特性

**1. ACP 协议:**
```json
{
  "type": "message",
  "session_id": "xxx",
  "content": {
    "role": "user",
    "content": "消息内容"
  }
}
```

**2. 流式输出:**
```python
async def stream_response(self, response):
    async for chunk in response.stream():
        yield chunk
```

**3. 会话管理:**
```python
class Session:
    def __init__(self, session_id: str):
        self.session_id = session_id
        self.messages = []
        self.token_count = 0
    
    def add_message(self, role: str, content: str):
        self.messages.append({"role": role, "content": content})
        self.update_token_count()
```

---

## 📊 实现对比

| 特性 | learn-claude-code | hermes-agent |
|------|-------------------|--------------|
| **语言** | TypeScript/Python | Python |
| **代码行数** | ~150 (s01) | ~500+ (核心) |
| **复杂度** | 简单，教学导向 | 生产级 |
| **Agent 循环** | 显式 while 循环 | 内置在服务器 |
| **消息格式** | Anthropic 原生 | ACP 协议封装 |
| **工具调用** | 直接解析 | 通过 ACP 工具层 |
| **会话管理** | 简单列表 | 完整 Session 类 |
| **流式输出** | ⚠️ 基础 | ✅ 完整支持 |
| **错误处理** | ⚠️ 基础 | ✅ 完善 |

---

## 🎯 关键设计模式

### 1. 命令模式 (Command Pattern)

```python
class ToolCommand:
    def __init__(self, name: str, params: dict):
        self.name = name
        self.params = params
    
    def execute(self) -> str:
        """执行命令"""
        pass

class BashCommand(ToolCommand):
    def execute(self) -> str:
        import subprocess
        result = subprocess.run(
            self.params["command"],
            shell=True,
            capture_output=True,
            text=True
        )
        return result.stdout
```

### 2. 策略模式 (Strategy Pattern)

```python
class LLMStrategy:
    def call(self, messages: list) -> Response:
        pass

class AnthropicStrategy(LLMStrategy):
    def call(self, messages: list) -> Response:
        # Anthropic API 调用
        pass

class OpenAIStrategy(LLMStrategy):
    def call(self, messages: list) -> Response:
        # OpenAI API 调用
        pass

# 使用
class Agent:
    def __init__(self, strategy: LLMStrategy):
        self.strategy = strategy
    
    def run(self, message: str):
        response = self.strategy.call([...])
```

### 3. 观察者模式 (Observer Pattern)

```python
class AgentObserver:
    def on_tool_call(self, tool_name: str, params: dict):
        pass
    
    def on_tool_result(self, result: str):
        pass
    
    def on_response(self, text: str):
        pass

class Agent:
    def __init__(self):
        self.observers = []
    
    def add_observer(self, observer: AgentObserver):
        self.observers.append(observer)
    
    def run(self, message: str):
        # ... Agent 循环
        for observer in self.observers:
            observer.on_tool_call(name, params)
```

---

## 💡 最佳实践

### 1. 循环终止保护

```python
MAX_ITERATIONS = 10

def agent_loop(message: str):
    iterations = 0
    while iterations < MAX_ITERATIONS:
        # ... Agent 逻辑
        iterations += 1
    
    if iterations >= MAX_ITERATIONS:
        return "⚠️ 达到最大迭代次数，终止执行"
```

### 2. 错误处理

```python
def agent_loop(message: str):
    try:
        # ... Agent 逻辑
    except RateLimitError:
        return "⚠️ 请求频率超限，请稍后重试"
    except ContextLengthError:
        return "⚠️ 上下文过长，需要压缩"
    except Exception as e:
        return f"⚠️ 发生错误：{str(e)}"
```

### 3. 日志记录

```python
import logging

logger = logging.getLogger("agent")

def agent_loop(message: str):
    logger.info(f"开始处理消息：{message[:50]}...")
    
    while True:
        logger.debug(f"第 {iteration} 次迭代")
        # ... Agent 逻辑
        
        if tool_calls:
            logger.info(f"调用工具：{tool_names}")
```

### 4. Token 计数

```python
def count_tokens(messages: list) -> int:
    """估算 Token 数量"""
    total = 0
    for msg in messages:
        total += len(msg["content"]) // 4  # 粗略估算
    return total

def check_context_limit(messages: list, limit: int = 100000):
    """检查是否超过限制"""
    tokens = count_tokens(messages)
    if tokens > limit:
        raise ContextLengthError(f"Token 数 {tokens} 超过限制 {limit}")
```

---

## 📝 实现练习

### 练习 1: 实现一个简单的 Agent 循环

```python
# 任务：实现一个能调用 bash 工具的 Agent
# 要求:
# 1. 接收用户消息
# 2. 调用 LLM 判断是否需要工具
# 3. 执行 bash 命令
# 4. 返回结果

def simple_agent(message: str):
    # 你的实现
    pass
```

### 练习 2: 添加最大迭代次数保护

```python
# 在练习 1 的基础上添加:
# 1. 最大迭代次数限制 (默认 10)
# 2. 达到限制时返回友好提示

def safe_agent(message: str, max_iterations: int = 10):
    # 你的实现
    pass
```

### 练习 3: 添加日志记录

```python
# 在练习 2 的基础上添加:
# 1. 记录每次迭代
# 2. 记录工具调用
# 3. 记录错误

def logged_agent(message: str, max_iterations: int = 10):
    # 你的实现
    pass
```

---

## 🔗 参考资料

### learn-claude-code
- [s01_agent_loop.py](https://github.com/shareAI-lab/learn-claude-code/blob/main/agents/s01_agent_loop.py)
- [s01-the-agent-loop.md](https://github.com/shareAI-lab/learn-claude-code/blob/main/docs/en/s01-the-agent-loop.md)

### hermes-agent
- [prompt_builder.py](https://github.com/NousResearch/hermes-agent/blob/main/agent/prompt_builder.py)
- [anthropic_adapter.py](https://github.com/NousResearch/hermes-agent/blob/main/agent/anthropic_adapter.py)
- [acp_adapter/server.py](https://github.com/NousResearch/hermes-agent/blob/main/acp_adapter/server.py)

---

[上一章: 项目概览](/agent-harness/README.md) | [下一章: 工具系统 →](/agent-harness/02-tool-system.md)
