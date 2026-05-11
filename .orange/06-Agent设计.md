# 06 Agent 设计

## 6.1 新版 create_agent（核心 API）

```python
from langchain.agents import create_agent
from langchain_anthropic import ChatAnthropic

agent = create_agent(
    model=ChatAnthropic(model="claude-sonnet-4-5"),
    tools=[search_tool, code_tool],
    middleware=[
        model_call_limit(max_calls=100),
        model_retry(max_retries=3),
        tool_retry(max_retries=2),
        summarization(model=llm, max_tokens=4096),
    ],
    response_format="auto",
    checkpointer=InMemorySaver(),  # 持久化
)

# 执行
result = await agent.ainvoke(
    {"messages": [HumanMessage("Search for Python news")]},
    config={"configurable": {"thread_id": "session-1"}}
)
```

内部构建的 LangGraph 图:
```
START → agent_node → [tools_node → agent_node]* → END
```

---

## 6.2 AgentMiddleware 8 个钩子

```python
class AgentMiddleware(Protocol):
    # 整个 Agent 调用的前后
    async def before_agent(self, state: AgentState) -> AgentState: ...
    async def after_agent(self, state: AgentState) -> AgentState: ...

    # 每次 LLM 调用的前后
    async def before_model(self, request: ModelRequest) -> ModelRequest: ...
    def wrap_model_call(self, call: ModelCallFn) -> ModelCallFn: ...
    async def after_model(self, state: AgentState) -> AgentState: ...

    # 每次工具调用的前后
    def wrap_tool_call(self, call: ToolCallFn) -> ToolCallFn: ...
    async def before_tool(self, request: ToolCallRequest) -> ToolCallRequest: ...
    async def after_tool(self, state: AgentState) -> AgentState: ...
```

**各钩子用途:**

| 钩子 | 数据类型 | 典型用途 |
|------|---------|---------|
| `before_agent` | AgentState | 初始化追踪、注入系统 prompt |
| `after_agent` | AgentState | 保存结果、清理资源 |
| `before_model` | ModelRequest | 修改 messages、替换工具列表 |
| `wrap_model_call` | ModelCallFn | 包装整个 LLM 调用（retry、timeout） |
| `after_model` | AgentState | 分析 LLM 响应、记录 token 用量 |
| `wrap_tool_call` | ToolCallFn | 包装整个工具调用 |
| `before_tool` | ToolCallRequest | 验证参数、人类确认 |
| `after_tool` | AgentState | 处理工具结果、记录副作用 |

---

## 6.3 内置中间件详解

### model_call_limit - 限制 LLM 调用次数

```python
from langchain.agents.middleware import model_call_limit

middleware = model_call_limit(max_calls=100)

# 内部实现：计数 LLM 调用次数
class ModelCallLimitMiddleware:
    def __init__(self, max_calls: int):
        self.max_calls = max_calls
        self.call_count = 0

    async def before_model(self, request: ModelRequest) -> ModelRequest:
        if self.call_count >= self.max_calls:
            raise ModelCallLimitExceeded(f"Exceeded {self.max_calls} LLM calls")
        self.call_count += 1
        return request
```

### model_retry - LLM 调用失败重试

```python
from langchain.agents.middleware import model_retry

middleware = model_retry(
    max_retries=3,
    retry_on=[RateLimitError, APITimeoutError],
    backoff=exponential_backoff,
)

# 实现 wrap_model_call 钩子
class ModelRetryMiddleware:
    def wrap_model_call(self, call: ModelCallFn) -> ModelCallFn:
        async def wrapped(*args, **kwargs):
            for attempt in range(self.max_retries + 1):
                try:
                    return await call(*args, **kwargs)
                except tuple(self.retry_on) as e:
                    if attempt == self.max_retries:
                        raise
                    await asyncio.sleep(self.backoff(attempt))
        return wrapped
```

### tool_retry - 工具调用失败重试

```python
from langchain.agents.middleware import tool_retry

middleware = tool_retry(max_retries=2)
# 实现 wrap_tool_call 钩子，工具返回错误时自动重试
```

### summarization - 上下文压缩

```python
from langchain.agents.middleware import summarization

middleware = summarization(
    model=ChatAnthropic(model="claude-haiku-4-5"),
    max_tokens=4096,    # 超过此 token 数时触发压缩
)

# 实现 before_model 钩子
class SummarizationMiddleware:
    async def before_model(self, request: ModelRequest) -> ModelRequest:
        if count_tokens(request.messages) > self.max_tokens:
            # 调用 LLM 压缩历史消息
            summary = await self.summarize(request.messages[:-5])
            request.messages = [SystemMessage(summary)] + request.messages[-5:]
        return request
```

### human_in_the_loop - 人类确认

```python
from langchain.agents.middleware import human_in_the_loop

middleware = human_in_the_loop(
    tools=["delete_file", "send_email"],  # 需要确认的工具
)

# 实现 before_tool 钩子
class HumanInTheLoopMiddleware:
    async def before_tool(self, request: ToolCallRequest) -> ToolCallRequest:
        if request.tool_name in self.tools:
            confirmed = await self.ask_human(
                f"Execute {request.tool_name}({request.tool_args})?"
            )
            if not confirmed:
                raise ToolExecutionDenied(f"User denied {request.tool_name}")
        return request
```

### pii - PII 数据脱敏

```python
from langchain.agents.middleware import pii

middleware = pii(
    entities=["PERSON", "EMAIL", "PHONE", "SSN"],
    action="mask",  # "mask" | "remove" | "hash"
)

# 实现 before_model (脱敏) 和 after_model (还原) 钩子
```

---

## 6.4 ResponseFormat 策略

```python
# AutoStrategy（默认）- 自动检测模型能力
response_format = "auto"
# 优先级：ProviderStrategy > ToolStrategy > 文本解析

# ToolStrategy - 用工具调用返回结构化输出
response_format = "tool"
# 添加一个特殊的 "final_response" 工具

# ProviderStrategy - 用 Provider 的结构化输出 API
response_format = "provider"
# 使用 model.with_structured_output(schema)
```

**带 schema 的结构化输出:**
```python
from pydantic import BaseModel

class SearchResult(BaseModel):
    answer: str
    sources: list[str]
    confidence: float

agent = create_agent(
    model=llm,
    tools=tools,
    response_format=SearchResult,  # 传入 Pydantic schema
)

result = await agent.ainvoke({"messages": [HumanMessage("...")]})
# result["structured_response"] 是 SearchResult 实例
```

---

## 6.5 自定义中间件示例

```python
from langchain.agents.middleware.types import (
    AgentMiddleware, ModelRequest, AgentState, ToolCallRequest
)
import time

class ObservabilityMiddleware(AgentMiddleware):
    """追踪 LLM 调用成本和延迟"""

    def __init__(self):
        self.total_tokens = 0
        self.total_calls = 0

    async def before_model(self, request: ModelRequest) -> ModelRequest:
        request._start_time = time.time()
        return request

    async def after_model(self, state: AgentState) -> AgentState:
        # 从最新消息提取 token 用量
        last_msg = state.messages[-1]
        if hasattr(last_msg, "usage_metadata"):
            self.total_tokens += last_msg.usage_metadata.total_tokens
        self.total_calls += 1
        return state

    async def before_tool(self, request: ToolCallRequest) -> ToolCallRequest:
        print(f"Calling tool: {request.tool_name}({request.tool_args})")
        return request

    async def after_agent(self, state: AgentState) -> AgentState:
        print(f"Total: {self.total_calls} LLM calls, {self.total_tokens} tokens")
        return state
```

---

## 6.6 与其他 Agent 框架的对比

| 特性 | LangChain create_agent | LangGraph create_react_agent | deer-flow ReAct | nanobot AgentLoop |
|------|----------------------|----------------------------|-----------------|-------------------|
| 执行引擎 | LangGraph Pregel | LangGraph Pregel | LangGraph | while 循环 |
| 中间件 | AgentMiddleware 8钩子 | 无原生中间件 | 无 | 无 |
| 持久化 | Checkpointer | Checkpointer | Checkpointer | 无 |
| 工具并行 | ToolNode 并行 | ToolNode 并行 | 并行 | 串行 |
| 人类干预 | human_in_the_loop 中间件 | interrupt_before | 无 | 无 |
| 流式输出 | astream | astream | astream | 无 |
| 结构化输出 | ResponseFormat 策略 | with_structured_output | 无 | 无 |
