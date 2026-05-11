# 11 Agent Loop

## 11.1 旧版 AgentExecutor While 循环（已废弃）

```python
# langchain_classic/agents/agent.py
class AgentExecutor(Chain):
    max_iterations: int = 15
    max_execution_time: Optional[float] = None

    def _call(self, inputs: dict, ...) -> dict:
        intermediate_steps = []
        iterations = 0
        time_elapsed = 0.0
        start_time = time.time()

        while self._should_continue(iterations, time_elapsed):
            next_step_output = self._iter_next_step(
                self.tools, ..., intermediate_steps, inputs
            )
            # AgentFinish → 终止
            if isinstance(next_step_output, AgentFinish):
                return self._return(next_step_output, intermediate_steps)
            # 继续循环
            intermediate_steps.extend(next_step_output)
            iterations += 1
            time_elapsed = time.time() - start_time

        # 超限强制停止
        return self._return(self._get_forced_stop(intermediate_steps), ...)

    def _should_continue(self, iterations: int, time_elapsed: float) -> bool:
        if self.max_iterations is not None and iterations >= self.max_iterations:
            return False
        if self.max_execution_time is not None and time_elapsed >= self.max_execution_time:
            return False
        return True
```

---

## 11.2 新版 create_agent：LangGraph Pregel 超步

```python
# langchain_v1 内部构建的图
def create_agent(model, tools, middleware, ...):
    builder = StateGraph(AgentState)
    builder.add_node("agent", _build_agent_node(model, middleware))
    builder.add_node("tools", ToolNode(tools))
    builder.add_conditional_edges("agent", _route)
    builder.add_edge("tools", "agent")
    return builder.compile(checkpointer=checkpointer)
```

**LangGraph Pregel 超步循环：**

```
超步 1: START → agent_node
    middleware.before_agent(state)
    middleware.before_model(request)
    middleware.wrap_model_call(call) → LLM 调用
    middleware.after_model(state)
    写入 messages channel: AIMessage(tool_calls=[...])

超步 2: agent_node → tools_node（如果有 tool_calls）
    middleware.wrap_tool_call(call)
    middleware.before_tool(request)
    ToolNode 并行执行所有工具
    middleware.after_tool(state)
    写入 messages channel: [ToolMessage × N]

超步 3: tools_node → agent_node
    ... 同超步 1 ...

超步 N: agent_node → END（LLM 无 tool_calls）
    middleware.after_agent(state)
    返回最终状态
```

---

## 11.3 终止条件

**旧版 AgentExecutor:**
- `AgentFinish` 对象返回 → `Final Answer:` 文本
- `max_iterations` 超限（默认 15）
- `max_execution_time` 超时
- 异常（raise 或 early_stopping_method="force" 强制停止）

**新版 create_agent / LangGraph:**
- `_route` 函数返回 `END` → LLM 输出无 `tool_calls`
- `ModelCallLimitExceeded` → `model_call_limit` 中间件触发
- `interrupt_before/after` → Human-in-the-Loop 中断
- `max_turns` → 配置参数限制最大轮次
- 异常（middleware 或工具抛出，不被 retry 捕获）

---

## 11.4 中间件对 Loop 的影响

```python
# 中间件可以修改 Loop 行为
class EarlyStopMiddleware(AgentMiddleware):
    """检测到特定条件时提前终止"""

    async def after_model(self, state: AgentState) -> AgentState:
        last_msg = state.messages[-1]
        # 检测 LLM 说 "I cannot help"
        if "cannot help" in last_msg.content.lower():
            # 修改状态触发终止（通过移除 tool_calls）
            state.messages[-1] = AIMessage(
                content=last_msg.content,
                tool_calls=[]  # 清空 tool_calls → route 会返回 END
            )
        return state
```

---

## 11.5 异步 vs 同步执行

```python
# ✅ 推荐：异步执行
result = await agent.ainvoke({"messages": [HumanMessage("...")]}, config)

# ⚠️ 同步（在异步事件循环中会阻塞）
result = agent.invoke({"messages": [HumanMessage("...")]}, config)

# 流式执行（最低延迟）
async for chunk in agent.astream({"messages": [...]}, config, stream_mode="values"):
    pass

# 事件流（最细粒度）
async for event in agent.astream_events({"messages": [...]}, config, version="v2"):
    if event["event"] == "on_chat_model_stream":
        print(event["data"]["chunk"].content, end="")
```

---

## 11.6 与其他项目 Agent Loop 对比

| 特性 | LangChain (旧版) | LangChain (新版) | LangGraph | deer-flow | nanobot | gstack |
|------|----------------|----------------|-----------|-----------|---------|--------|
| **循环机制** | while 循环 | LangGraph Pregel | Pregel BSP | Pregel BSP | while 循环 | claude 原生 |
| **终止条件** | AgentFinish/max_iter | route→END | route→END | route→END | AgentFinish | 无固定 |
| **中断恢复** | ❌ | ✅ 中间件 | ✅ interrupt API | ✅ | ❌ | ❌ |
| **并行工具** | ❌ 串行 | ✅ ToolNode | ✅ ToolNode | ✅ | ❌ | N/A (SKILL) |
| **中间件** | ❌ | ✅ 8 钩子 | ❌ | ❌ | ❌ | ❌ |
| **持久化** | ❌ | ✅ Checkpointer | ✅ Checkpointer | ✅ | ✅ 文件 | ⚠️ git |
| **最大轮次** | max_iterations=15 | max_turns 参数 | 无内置限制 | 无内置限制 | 无 | claude --max-turns |
| **异步** | ❌ | ✅ ainvoke | ✅ ainvoke | ✅ | ❌ | ✅ (claude 进程) |
