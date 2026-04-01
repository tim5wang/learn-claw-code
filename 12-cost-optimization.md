# 08 - 成本优化策略

> **降低 LLM 使用成本的完整指南** - Token 优化、缓存策略、预算管理  
> **更新时间:** 2026-04-01

---

## 💰 成本结构分析

### claw-code 的成本构成

```
┌─────────────────────────────────────────────────────────┐
│              claw-code 成本构成                          │
│                                                          │
│  ┌────────────────────────────────────────────────────┐ │
│  │  LLM API 调用 (70-90%)                              │ │
│  │  - Token 消耗 (输入 + 输出)                          │ │
│  │  - 模型选择 (GPT-4 vs GPT-3.5 vs Claude)           │ │
│  │  - 调用频率                                         │ │
│  └────────────────────────────────────────────────────┘ │
│                                                          │
│  ┌────────────────────────────────────────────────────┐ │
│  │  计算资源 (5-15%)                                   │ │
│  │  - CPU/内存                                         │ │
│  │  - 存储                                             │ │
│  │  - 网络                                             │ │
│  └────────────────────────────────────────────────────┘ │
│                                                          │
│  ┌────────────────────────────────────────────────────┐ │
│  │  其他 (5-15%)                                       │ │
│  │  - 工具调用 (网络请求、文件操作)                     │ │
│  │  - 日志存储                                         │ │
│  │  - 监控告警                                         │ │
│  └────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────┘
```

### Token 消耗分布

基于实际使用数据分析：

| 场景 | Token 占比 | 说明 |
|------|-----------|------|
| **上下文输入** | 50-60% | 历史消息、项目文件、记忆 |
| **LLM 输出** | 25-35% | Agent 回复、代码生成 |
| **工具调用** | 10-15% | 工具定义、参数、结果 |
| **系统提示** | 3-5% | 系统指令、配置 |

---

## 📊 成本基准测试

### 典型会话成本分析

**场景:** 完成一个中等复杂度的编程任务 (重构一个模块)

```
┌─────────────────────────────────────────────────────────┐
│  会话统计                                                │
│                                                          │
│  迭代次数：8 轮                                           │
│  总 Token: 125,000                                       │
│  耗时：25 分钟                                            │
│                                                          │
│  Token 分布：                                            │
│  ├─ 上下文输入：72,000 (57.6%)                          │
│  ├─ LLM 输出：38,000 (30.4%)                             │
│  ├─ 工具调用：12,000 (9.6%)                             │
│  └─ 系统提示：3,000 (2.4%)                              │
│                                                          │
│  成本估算 (以 GPT-4 Turbo 为例):                          │
│  ├─ 输入：72K × $0.01/1K = $0.72                        │
│  ├─ 输出：38K × $0.03/1K = $1.14                        │
│  └─ 总计：$1.86 / 会话                                   │
└─────────────────────────────────────────────────────────┘
```

### 不同模型成本对比

| 模型 | 输入价格 | 输出价格 | 单次会话成本 | 月成本 (100 次/天) |
|------|---------|---------|-------------|------------------|
| **GPT-4 Turbo** | $0.01/1K | $0.03/1K | $1.86 | $5,580 |
| **GPT-4** | $0.03/1K | $0.06/1K | $4.68 | $14,040 |
| **GPT-3.5 Turbo** | $0.0005/1K | $0.0015/1K | $0.09 | $270 |
| **Claude 3 Opus** | $0.015/1K | $0.075/1K | $4.32 | $12,960 |
| **Claude 3 Sonnet** | $0.003/1K | $0.015/1K | $0.90 | $2,700 |
| **Qwen-Max** | $0.004/1K | $0.012/1K | $0.72 | $2,160 |

---

## 🔧 优化策略详解

### 策略 1: 上下文压缩 ⭐⭐⭐⭐⭐

**原理:** 减少输入 Token 数量

**优化前 vs 优化后:**

```
优化前 (完整历史):
├─ 最近 20 轮对话：40,000 tokens
├─ 项目文件：20,000 tokens
├─ 长期记忆：10,000 tokens
└─ 总计：70,000 tokens

优化后 (压缩 + 摘要):
├─ 最近 5 轮对话：10,000 tokens
├─ 历史摘要：5,000 tokens
├─ 项目文件 (精选): 8,000 tokens
├─ 长期记忆 (相关): 3,000 tokens
└─ 总计：26,000 tokens

节省：63% Token
```

**实现方法:**

```python
class ContextCompressor:
    """上下文压缩器"""
    
    def __init__(self, llm_client, max_tokens=30000):
        self.llm = llm_client
        self.max_tokens = max_tokens
    
    async def compress(self, messages: List[Message]) -> List[Message]:
        """压缩消息历史"""
        
        # 1. 保留最近 N 轮
        recent_count = 5
        recent_messages = messages[-recent_count * 2:]  # 每轮 2 条消息
        
        # 2. 压缩旧消息
        old_messages = messages[:-recent_count * 2]
        if old_messages:
            summary = await self._generate_summary(old_messages)
            summary_message = Message(
                role="system",
                content=f"[历史摘要] {summary}"
            )
            return [summary_message] + recent_messages
        
        return recent_messages
    
    async def _generate_summary(self, messages: List[Message]) -> str:
        """生成历史摘要"""
        prompt = f"""请总结以下对话，提取关键信息：
- 主要讨论内容
- 重要决策
- 待办事项
- 技术要点

对话：
{self._format_messages(messages)}

摘要 (300 字以内):"""
        
        response = await self.llm.chat(
            messages=[{"role": "user", "content": prompt}],
            max_tokens=500
        )
        return response.content
```

**压缩触发条件:**

```python
COMPACTION_CONFIG = {
    "token_threshold": 50000,      # Token 超过 50K 触发
    "message_count": 30,           # 消息超过 30 轮触发
    "duration_minutes": 60,        # 会话超过 60 分钟触发
    "compression_ratio": 0.4,      # 目标压缩率 40%
}
```

---

### 策略 2: 智能缓存 ⭐⭐⭐⭐⭐

**原理:** 避免重复计算和 API 调用

**缓存类型:**

```
┌─────────────────────────────────────────────────────────┐
│                    缓存层次结构                          │
│                                                          │
│  L1: 响应缓存 (最快)                                     │
│  ├── 相同请求直接返回                                    │
│  ├── 命中率：30-50%                                     │
│  └── 延迟：<1ms                                         │
│                                                          │
│  L2: 语义缓存 (较快)                                     │
│  ├── 相似请求返回缓存                                    │
│  ├── 命中率：50-70%                                     │
│  └── 延迟：10-50ms                                      │
│                                                          │
│  L3: 工具结果缓存 (中等)                                 │
│  ├── 工具执行结果缓存                                    │
│  ├── 命中率：40-60%                                     │
│  └── 延迟：5-10ms                                       │
│                                                          │
│  L4: 文件内容缓存 (较慢)                                 │
│  ├── 文件变更检测                                        │
│  ├── 命中率：60-80%                                     │
│  └── 延迟：1-5ms                                        │
└─────────────────────────────────────────────────────────┘
```

**响应缓存实现:**

```python
class ResponseCache:
    """LLM 响应缓存"""
    
    def __init__(self, ttl_seconds=3600):
        self.cache = {}  # key -> (response, timestamp)
        self.ttl = ttl_seconds
    
    def _make_key(self, messages: List[dict], **kwargs) -> str:
        """生成缓存键"""
        # 简单哈希
        content = json.dumps({
            "messages": messages,
            "kwargs": kwargs
        }, sort_keys=True)
        return hashlib.sha256(content.encode()).hexdigest()
    
    async def get(self, key: str) -> Optional[LLMResponse]:
        """获取缓存"""
        if key in self.cache:
            response, timestamp = self.cache[key]
            if time.time() - timestamp < self.ttl:
                return response
            else:
                del self.cache[key]
        return None
    
    def set(self, key: str, response: LLMResponse):
        """设置缓存"""
        self.cache[key] = (response, time.time())
    
    async def get_or_compute(
        self,
        messages: List[dict],
        compute_fn: callable,
        **kwargs
    ) -> LLMResponse:
        """获取或计算"""
        key = self._make_key(messages, **kwargs)
        
        # 尝试缓存
        cached = await self.get(key)
        if cached:
            return cached
        
        # 计算新响应
        response = await compute_fn(messages, **kwargs)
        
        # 存入缓存
        self.set(key, response)
        
        return response

# 使用示例
cache = ResponseCache(ttl_seconds=3600)

async def chat_with_cache(messages, **kwargs):
    return await cache.get_or_compute(
        messages,
        llm_client.chat,
        **kwargs
    )
```

**语义缓存实现:**

```python
class SemanticCache:
    """语义缓存 - 基于向量相似度"""
    
    def __init__(self, embedding_model, similarity_threshold=0.9):
        self.embedding = embedding_model
        self.threshold = similarity_threshold
        self.cache = []  # [(embedding, response, timestamp)]
    
    def _embed(self, text: str) -> np.ndarray:
        """生成向量"""
        return self.embedding.encode(text)
    
    def _similarity(self, a: np.ndarray, b: np.ndarray) -> float:
        """计算余弦相似度"""
        return cosine_similarity(a.reshape(1, -1), b.reshape(1, -1))[0][0]
    
    async def get(self, query: str) -> Optional[LLMResponse]:
        """获取相似缓存"""
        query_emb = self._embed(query)
        
        for emb, response, ts in self.cache:
            if time.time() - ts > 3600:  # 1 小时过期
                continue
            
            sim = self._similarity(query_emb, emb)
            if sim >= self.threshold:
                return response
        
        return None
    
    def set(self, query: str, response: LLMResponse):
        """设置缓存"""
        emb = self._embed(query)
        self.cache.append((emb, response, time.time()))
```

---

### 策略 3: 模型路由 ⭐⭐⭐⭐

**原理:** 根据任务复杂度选择合适的模型

**路由策略:**

```python
class ModelRouter:
    """模型路由器"""
    
    def __init__(self):
        self.models = {
            "simple": "gpt-3.5-turbo",      # 简单任务
            "medium": "gpt-4-turbo",        # 中等任务
            "complex": "gpt-4",             # 复杂任务
            "code": "gpt-4-turbo",          # 代码任务
            "reasoning": "gpt-4",           # 推理任务
        }
    
    def select_model(self, task: Task) -> str:
        """选择模型"""
        
        # 1. 基于任务类型
        if task.type == "code_review":
            return self.models["code"]
        
        if task.type == "data_analysis":
            return self.models["reasoning"]
        
        # 2. 基于复杂度评分
        complexity = self._estimate_complexity(task)
        
        if complexity < 0.3:
            return self.models["simple"]
        elif complexity < 0.7:
            return self.models["medium"]
        else:
            return self.models["complex"]
        
        # 3. 基于成本预算
        if task.budget < 0.50:
            return self.models["simple"]
        
        return self.models["medium"]
    
    def _estimate_complexity(self, task: Task) -> float:
        """估计任务复杂度 (0-1)"""
        score = 0.0
        
        # 文件数量
        score += min(len(task.files) / 10, 0.3)
        
        # 代码行数
        total_lines = sum(f.lines for f in task.files)
        score += min(total_lines / 1000, 0.3)
        
        # 依赖关系
        score += min(task.dependencies / 5, 0.2)
        
        # 历史迭代次数
        score += min(task.iterations / 10, 0.2)
        
        return min(score, 1.0)
```

**成本对比:**

| 任务类型 | 推荐模型 | 成本/次 | 替代方案 | 节省 |
|---------|---------|--------|---------|------|
| 简单问答 | GPT-3.5 | $0.01 | GPT-4 | 97% |
| 代码审查 | GPT-4 Turbo | $0.50 | GPT-4 | 60% |
| 复杂重构 | GPT-4 | $2.00 | - | - |
| 数据分析 | GPT-4 | $1.50 | - | - |

---

### 策略 4: 请求合并 ⭐⭐⭐

**原理:** 合并多个小请求为一个大请求

**实现:**

```python
class RequestBatcher:
    """请求批处理器"""
    
    def __init__(self, max_batch_size=5, max_wait_ms=100):
        self.batch_size = max_batch_size
        self.max_wait = max_wait_ms / 1000
        self.pending = []
        self.lock = asyncio.Lock()
    
    async def add(self, request: dict) -> LLMResponse:
        """添加请求"""
        future = asyncio.Future()
        
        async with self.lock:
            self.pending.append((request, future))
            
            # 达到批次大小或超时
            if len(self.pending) >= self.batch_size:
                asyncio.create_task(self._process_batch())
            elif len(self.pending) == 1:
                # 第一个请求，启动定时器
                asyncio.create_task(self._wait_and_process())
        
        return await future
    
    async def _wait_and_process(self):
        """等待并处理"""
        await asyncio.sleep(self.max_wait)
        async with self.lock:
            if self.pending:
                await self._process_batch()
    
    async def _process_batch(self):
        """处理批次"""
        batch = self.pending[:self.batch_size]
        self.pending = self.pending[self.batch_size:]
        
        # 合并请求
        combined_messages = []
        for req, _ in batch:
            combined_messages.extend(req["messages"])
        
        # 批量调用
        response = await llm_client.chat(messages=combined_messages)
        
        # 分发结果
        for _, future in batch:
            future.set_result(response)
```

---

### 策略 5: 流式处理优化 ⭐⭐⭐

**原理:** 提前终止不需要的响应

**实现:**

```python
class StreamingOptimizer:
    """流式处理优化器"""
    
    def __init__(self):
        self.stop_patterns = [
            r"\n\n$",  # 双换行
            r"```$",   # 代码块结束
            r"\n[1-9]\.",  # 列表开始
        ]
    
    async def stream_with_early_stop(
        self,
        messages: List[dict],
        stop_condition: callable = None
    ) -> AsyncGenerator[str, None]:
        """带提前终止的流式处理"""
        
        buffer = ""
        
        async for token in llm_client.chat_stream(messages):
            buffer += token
            yield token
            
            # 检查停止条件
            if stop_condition and stop_condition(buffer):
                break
            
            # 检查模式
            for pattern in self.stop_patterns:
                if re.search(pattern, buffer):
                    # 可能已经完成
                    if self._is_complete(buffer):
                        break
        
        # 记录优化效果
        self._log_optimization(buffer)
    
    def _is_complete(self, text: str) -> bool:
        """判断是否完成"""
        # 检查括号匹配
        if text.count("(") != text.count(")"):
            return False
        if text.count("[") != text.count("]"):
            return False
        if text.count("{") != text.count("}"):
            return False
        
        # 检查代码块
        if text.count("```") % 2 != 0:
            return False
        
        return True
```

---

## 💰 预算管理

### 预算配置

```python
class BudgetManager:
    """预算管理器"""
    
    def __init__(self, daily_budget: float = 100.0):
        self.daily_budget = daily_budget
        self.spent_today = 0.0
        self.alerts = []
    
    def check_budget(self, estimated_cost: float) -> bool:
        """检查预算"""
        if self.spent_today + estimated_cost > self.daily_budget:
            self._trigger_alert("budget_exceeded", estimated_cost)
            return False
        return True
    
    def record_usage(self, tokens: int, model: str):
        """记录使用量"""
        cost = self._calculate_cost(tokens, model)
        self.spent_today += cost
        
        # 检查阈值
        usage_ratio = self.spent_today / self.daily_budget
        
        if usage_ratio >= 0.9:
            self._trigger_alert("near_limit", usage_ratio)
        elif usage_ratio >= 0.75:
            self._trigger_alert("warning", usage_ratio)
        elif usage_ratio >= 0.5:
            self._trigger_alert("info", usage_ratio)
    
    def _calculate_cost(self, tokens: int, model: str) -> float:
        """计算成本"""
        prices = {
            "gpt-4": {"input": 0.03, "output": 0.06},
            "gpt-4-turbo": {"input": 0.01, "output": 0.03},
            "gpt-3.5-turbo": {"input": 0.0005, "output": 0.0015},
        }
        
        # 简化计算 (假设输入输出各半)
        price = prices.get(model, prices["gpt-4-turbo"])
        return (tokens / 2 / 1000 * price["input"] +
                tokens / 2 / 1000 * price["output"])
    
    def _trigger_alert(self, alert_type: str, value: float):
        """触发告警"""
        self.alerts.append({
            "type": alert_type,
            "value": value,
            "timestamp": time.time()
        })
```

### 自动限流

```python
class RateLimiter:
    """自动限流器"""
    
    def __init__(self, budget_manager: BudgetManager):
        self.budget = budget_manager
        self.current_rate = 1.0  # 100%
        self.min_rate = 0.1      # 最低 10%
    
    def adjust_rate(self):
        """根据预算调整速率"""
        usage_ratio = self.budget.spent_today / self.budget.daily_budget
        
        if usage_ratio > 0.9:
            # 接近超支，大幅降低
            self.current_rate = max(self.min_rate, 0.2)
        elif usage_ratio > 0.75:
            # 警告，降低速率
            self.current_rate = max(self.min_rate, 0.5)
        elif usage_ratio > 0.5:
            # 正常，保持
            self.current_rate = 0.8
        else:
            # 充足，全速
            self.current_rate = 1.0
    
    async def execute(self, task: callable):
        """带限流的执行"""
        self.adjust_rate()
        
        # 根据速率决定是否跳过非关键任务
        if self.current_rate < 0.5:
            if not task.is_critical:
                return None  # 跳过非关键任务
        
        return await task()
```

---

## 📈 监控与可视化

### 成本监控仪表板

```python
class CostDashboard:
    """成本监控仪表板"""
    
    def __init__(self):
        self.metrics = {
            "total_cost": 0.0,
            "total_tokens": 0,
            "sessions": 0,
            "avg_cost_per_session": 0.0,
        }
    
    def get_summary(self) -> dict:
        """获取摘要"""
        return {
            "today": {
                "cost": self.metrics["total_cost"],
                "tokens": self.metrics["total_tokens"],
                "sessions": self.metrics["sessions"],
            },
            "trend": self._calculate_trend(),
            "projection": self._project_monthly(),
        }
    
    def get_breakdown(self) -> dict:
        """获取明细"""
        return {
            "by_model": self._group_by_model(),
            "by_task": self._group_by_task(),
            "by_user": self._group_by_user(),
        }
    
    def get_optimization_suggestions(self) -> List[str]:
        """获取优化建议"""
        suggestions = []
        
        # 检查缓存命中率
        if self.cache_hit_rate < 0.3:
            suggestions.append("缓存命中率低，建议增加缓存")
        
        # 检查模型使用
        if self.gpt4_ratio > 0.5:
            suggestions.append("GPT-4 使用比例高，考虑使用 GPT-4 Turbo")
        
        # 检查上下文长度
        if self.avg_context_tokens > 50000:
            suggestions.append("上下文过长，建议启用压缩")
        
        return suggestions
```

---

## 📊 优化效果评估

### 优化前后对比

| 指标 | 优化前 | 优化后 | 改善 |
|------|--------|--------|------|
| **平均 Token/会话** | 125,000 | 45,000 | -64% |
| **平均成本/会话** | $1.86 | $0.67 | -64% |
| **缓存命中率** | 0% | 45% | +45% |
| **响应延迟** | 3.5s | 2.1s | -40% |
| **月成本 (100 次/天)** | $5,580 | $2,010 | -64% |

### 各策略贡献度

```
┌─────────────────────────────────────────────────────────┐
│              成本优化贡献度                              │
│                                                          │
│  上下文压缩  ████████████████████████████████░░  35%    │
│  智能缓存    ██████████████████████░░░░░░░░░░░░  25%    │
│  模型路由    ████████████████░░░░░░░░░░░░░░░░░░  20%    │
│  请求合并    ████████░░░░░░░░░░░░░░░░░░░░░░░░░░  10%    │
│  预算管理    ████████░░░░░░░░░░░░░░░░░░░░░░░░░░  10%    │
└─────────────────────────────────────────────────────────┘
```

---

## 🎯 实施路线图

### 第 1 周：快速见效
- [ ] 启用上下文压缩
- [ ] 配置预算告警
- [ ] 基础响应缓存

### 第 2 周：深度优化
- [ ] 语义缓存
- [ ] 模型路由
- [ ] 监控仪表板

### 第 3 周：自动化
- [ ] 自动限流
- [ ] 智能批处理
- [ ] 优化建议系统

---

## 📝 最佳实践清单

### ✅ 应该做的

1. **始终启用上下文压缩** - 减少 50-70% 输入 Token
2. **使用缓存** - 命中率可达 40-60%
3. **选择合适的模型** - 简单任务用 GPT-3.5
4. **设置预算告警** - 防止意外超支
5. **监控使用量** - 及时发现问题

### ❌ 避免做的

1. **不要每次都传完整历史** - 使用压缩/摘要
2. **不要过度使用 GPT-4** - 大部分任务 GPT-4 Turbo 足够
3. **不要忽略缓存** - 重复请求浪费钱
4. **不要无限制会话** - 定期压缩或重置
5. **不要忽略监控** - 及时发现问题

---

[← 返回上一章](/claw-code/11-future-research) | [下一章 →](/claw-code/12-cost-optimization-advanced)
