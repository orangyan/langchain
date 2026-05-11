# 10 Lead-Sub Agent

## 10.1 架构：LangChain 作为 LangGraph 封装层

LangChain v1 的多 Agent 能力完全依赖 LangGraph 子图机制：

```
LangChain create_agent()
    ↓ 内部构建
LangGraph StateGraph
    START → agent_node → tools_node → agent_node → END
    ↑
    可以在 tools_node 中放置"子 Agent"工具
```

---

## 10.2 工具作为子 Agent（最简单模式）

```python
from langchain.agents import create_agent
from langchain_core.tools import tool

# 子 Agent：专门处理数学
math_agent = create_agent(
    model=ChatAnthropic(model="claude-haiku-4-5"),
    tools=[calculator_tool],
)

# 将子 Agent 包装成工具
@tool
async def call_math_agent(question: str) -> str:
    """Delegate math questions to the math specialist agent."""
    result = await math_agent.ainvoke(
        {"messages": [HumanMessage(question)]},
        config={"configurable": {"thread_id": f"math-{uuid4()}"}}
    )
    return result["messages"][-1].content

# 主 Agent：调度者
lead_agent = create_agent(
    model=ChatAnthropic(model="claude-sonnet-4-5"),
    tools=[call_math_agent, search_tool],
)
```

---

## 10.3 LangGraph 子图模式（更强大）

直接使用 LangGraph 构建 Lead-Sub 架构：

```python
from langgraph.graph import StateGraph, START, END
from langchain.agents import create_agent

# 创建专业子 Agent
research_agent = create_agent(model=llm, tools=[search, wikipedia])
writing_agent = create_agent(model=llm, tools=[draft_tool])

# 构建 Supervisor 图
class SupervisorState(TypedDict):
    messages: Annotated[list[BaseMessage], add_messages]
    next: str  # 下一个要执行的 Agent

def supervisor_node(state: SupervisorState) -> SupervisorState:
    """决定下一步调用哪个子 Agent"""
    decision = llm.invoke(state["messages"])
    return {"next": parse_decision(decision)}

def route(state: SupervisorState) -> str:
    return state["next"]

builder = StateGraph(SupervisorState)
builder.add_node("supervisor", supervisor_node)
builder.add_node("research", research_agent)
builder.add_node("writing", writing_agent)
builder.add_conditional_edges("supervisor", route, {
    "research": "research",
    "writing": "writing",
    "FINISH": END
})
builder.add_edge("research", "supervisor")
builder.add_edge("writing", "supervisor")
builder.add_edge(START, "supervisor")

graph = builder.compile(checkpointer=InMemorySaver())
```

---

## 10.4 并行子 Agent（Send API）

```python
from langgraph.types import Send
from langchain.agents import create_agent

worker_agent = create_agent(model=llm, tools=[search])

class MapReduceState(TypedDict):
    topics: list[str]
    results: Annotated[list[str], operator.add]

def dispatch_tasks(state: MapReduceState) -> list[Send]:
    """为每个 topic 派发一个子 Agent 任务（并行）"""
    return [
        Send("worker", {"messages": [HumanMessage(f"Research: {topic}")]})
        for topic in state["topics"]
    ]

def aggregate(state: MapReduceState) -> dict:
    """汇总所有子 Agent 的结果"""
    combined = "\n\n".join(state["results"])
    return {"final_report": combined}

builder = StateGraph(MapReduceState)
builder.add_node("worker", worker_agent)
builder.add_node("aggregate", aggregate)
builder.add_conditional_edges(START, dispatch_tasks)
builder.add_edge("worker", "aggregate")
builder.add_edge("aggregate", END)
```

---

## 10.5 与旧版 Chain 的对比

旧版 LangChain 的链式调用（Sequential Chain）是串行的，没有真正的 Lead-Sub Agent 概念：

```python
# ❌ 旧版 Sequential Chain（废弃）
from langchain.chains import SequentialChain

chain1 = LLMChain(llm=llm, prompt=prompt1, output_key="research")
chain2 = LLMChain(llm=llm, prompt=prompt2, output_key="writing")
overall = SequentialChain(chains=[chain1, chain2], ...)

# ✅ 新版 Supervisor Graph（真正的 Lead-Sub Agent）
graph = build_supervisor_graph(lead=supervisor, workers=[research_agent, writing_agent])
```

---

## 10.6 与其他项目 Lead-Sub Agent 对比

| 特性 | LangChain (新版) | LangGraph | deer-flow | hermes-agent | nanobot | gstack |
|------|----------------|-----------|-----------|-------------|---------|--------|
| **Lead-Sub 机制** | create_agent 作工具 | StateGraph 子图 | StateGraph 子图 | 无 | AgentTool | SKILL.md |
| **并行执行** | Send API | Send API | Send API | 无 | 有限 | Conductor 模式 |
| **通信方式** | 消息传递 | 消息传递 | 消息传递 | 无 | 工具返回值 | 文件/git |
| **状态共享** | Checkpointer | Checkpointer | Checkpointer | 无 | 无 | 无 |
| **专业化** | 工具集隔离 | 图节点隔离 | 图节点隔离 | 无 | 工具集隔离 | SKILL.md 专业化 |
| **动态派发** | Send API | Send API | Send API | 无 | 无 | Conductor 10-15个 |
| **原生支持** | ✅ (via LangGraph) | ✅ | ✅ | ❌ | ⚠️ 有限 | ✅ (SKILL 系统) |

**关键洞察：**
- LangChain v1 的 Lead-Sub Agent 能力完全依赖 LangGraph
- `create_agent` 返回的 `CompiledStateGraph` 可以直接作为另一个图的节点
- 最简单的方式是将子 Agent 包装成 `@tool`，让主 Agent 通过工具调用
- 复杂场景推荐直接用 LangGraph `StateGraph` + Supervisor 模式
