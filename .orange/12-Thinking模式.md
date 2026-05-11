# 12 Thinking 模式

## 12.1 LangChain 中的 Extended Thinking

LangChain 通过 `langchain-anthropic` 集成支持 Claude 的 Extended Thinking：

```python
from langchain_anthropic import ChatAnthropic

# 启用 Extended Thinking
llm = ChatAnthropic(
    model="claude-sonnet-4-5",
    thinking={"type": "enabled", "budget_tokens": 10000},
)

# 直接调用
response = await llm.ainvoke("What is 17 * 38?")
# response.content 包含 thinking 块和文本块
for block in response.content:
    if block["type"] == "thinking":
        print(f"[Thinking]: {block['thinking']}")
    elif block["type"] == "text":
        print(f"[Answer]: {block['text']}")
```

---

## 12.2 在 create_agent 中使用 Thinking

```python
from langchain_anthropic import ChatAnthropic
from langchain.agents import create_agent

# 创建带 Thinking 的 LLM
thinking_model = ChatAnthropic(
    model="claude-sonnet-4-5",
    thinking={"type": "enabled", "budget_tokens": 8000},
)

agent = create_agent(
    model=thinking_model,
    tools=[search_tool, code_tool],
)

result = await agent.ainvoke(
    {"messages": [HumanMessage("Solve this complex math problem: ...")]}
)

# 消息中包含 thinking 内容
for msg in result["messages"]:
    if hasattr(msg, "content") and isinstance(msg.content, list):
        for block in msg.content:
            if isinstance(block, dict) and block.get("type") == "thinking":
                print(f"Thinking: {block['thinking'][:200]}...")
```

---

## 12.3 流式 Thinking 输出

```python
async for chunk in agent.astream(
    {"messages": [HumanMessage("Complex reasoning task")]},
    config=config,
    stream_mode="messages"
):
    message, metadata = chunk
    if hasattr(message, "content"):
        for block in (message.content if isinstance(message.content, list) else []):
            if isinstance(block, dict):
                if block.get("type") == "thinking_delta":
                    print(f"💭 {block['thinking']}", end="")
                elif block.get("type") == "text_delta":
                    print(block["text"], end="")
```

---

## 12.4 通过 with_config 动态切换 Thinking

```python
# 基础模型（无 Thinking）
base_model = ChatAnthropic(model="claude-haiku-4-5")
agent = create_agent(model=base_model, tools=tools)

# 在特定调用中启用 Thinking（覆盖模型配置）
config_with_thinking = {
    "configurable": {
        "thread_id": "session-1",
        "model_kwargs": {
            "thinking": {"type": "enabled", "budget_tokens": 5000}
        }
    }
}
result = await agent.ainvoke(
    {"messages": [HumanMessage("Hard problem")]},
    config=config_with_thinking
)
```

---

## 12.5 Thinking 与 Middleware 集成

```python
class ThinkingLogMiddleware(AgentMiddleware):
    """记录 Thinking 内容供调试使用"""

    async def after_model(self, state: AgentState) -> AgentState:
        last_msg = state.messages[-1]
        if hasattr(last_msg, "content") and isinstance(last_msg.content, list):
            for block in last_msg.content:
                if isinstance(block, dict) and block.get("type") == "thinking":
                    thinking_text = block.get("thinking", "")
                    token_count = len(thinking_text.split())
                    print(f"[Thinking tokens: ~{token_count}]")
        return state

agent = create_agent(
    model=ChatAnthropic(
        model="claude-sonnet-4-5",
        thinking={"type": "enabled", "budget_tokens": 8000}
    ),
    tools=tools,
    middleware=[ThinkingLogMiddleware()],
)
```

---

## 12.6 与其他项目 Thinking 模式对比

| 特性 | LangChain | LangGraph | deer-flow | hermes-agent | nanobot | gstack |
|------|----------|-----------|-----------|-------------|---------|--------|
| **Thinking 方式** | LLM 参数传入 | LLM 参数传入 | LLM 参数传入 | 无 | 无 | 文本指令注入 |
| **配置位置** | ChatAnthropic 构造 | ChatAnthropic 构造 | ChatAnthropic 构造 | N/A | N/A | SKILL.md 文本 |
| **流式 Thinking** | ✅ astream_events | ✅ astream_events | ✅ | ❌ | ❌ | ❌ |
| **动态切换** | with_config | with_config | 无 | N/A | N/A | 模型覆盖 |
| **Thinking 可见** | 在消息块中 | 在消息块中 | 在消息块中 | N/A | N/A | 隐式（通过输出体现） |
| **budget_tokens** | 可配置 | 可配置 | 可配置 | N/A | N/A | N/A |
| **Middleware 集成** | ✅ after_model 钩子 | ❌ 无中间件 | ❌ | N/A | N/A | N/A |

**关键洞察：**
- LangChain/LangGraph/deer-flow 都通过 `langchain-anthropic` 的 `ChatAnthropic` 参数启用 Thinking
- LangChain 独有优势：可通过 `after_model` 中间件钩子拦截和处理 Thinking 内容
- gstack 用文本指令触发隐式思考，不使用 API 级参数
- `budget_tokens` 参数控制 Thinking 深度，越大推理越深但越慢
