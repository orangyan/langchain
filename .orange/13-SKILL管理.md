# 13 SKILL 管理

## 13.1 LangChain 的工具系统

LangChain 没有"SKILL"的概念，而是使用 **Tool（工具）** 系统：

```
langchain-core (BaseTool/StructuredTool)
    ↑
langchain-community (第三方工具集成)
    ↑
langchain_v1 create_agent (工具注册到 Agent)
```

---

## 13.2 @tool 装饰器（最简单）

```python
from langchain_core.tools import tool

@tool
def search_web(query: str) -> str:
    """Search the web for information about a topic.

    Args:
        query: The search query string.

    Returns:
        Search results as a formatted string.
    """
    # 实现搜索逻辑
    return f"Results for: {query}"

# @tool 等价于：
# StructuredTool.from_function(func=search_web, name="search_web", ...)

print(search_web.name)         # "search_web"
print(search_web.description)  # 从 docstring 提取
print(search_web.args)         # {"query": {"type": "string"}}
```

---

## 13.3 StructuredTool（带 Pydantic Schema）

```python
from langchain_core.tools import StructuredTool
from pydantic import BaseModel, Field

class SearchInput(BaseModel):
    query: str = Field(description="The search query")
    max_results: int = Field(default=5, description="Maximum number of results")
    language: str = Field(default="en", description="Language code")

def search_web(query: str, max_results: int = 5, language: str = "en") -> str:
    """Search the web with advanced options."""
    return f"Searching '{query}' ({max_results} results, lang={language})"

search_tool = StructuredTool.from_function(
    func=search_web,
    name="search_web",
    description="Search the web for information",
    args_schema=SearchInput,
    return_direct=False,  # True = 直接返回给用户，跳过后续 LLM
)
```

---

## 13.4 BaseTool 自定义实现

```python
from langchain_core.tools import BaseTool
from typing import Type, Union

class DatabaseQueryTool(BaseTool):
    name: str = "database_query"
    description: str = "Query the company database"
    args_schema: Type[BaseModel] = DatabaseQueryInput

    # vendor-specific 扩展（如 Anthropic 缓存）
    extras: dict = {"cache_control": {"type": "ephemeral"}}

    def _run(self, query: str, table: str) -> str:
        """同步实现"""
        return db.execute(query, table)

    async def _arun(self, query: str, table: str) -> str:
        """异步实现（可选，不实现则自动 run_in_executor）"""
        return await db.aexecute(query, table)
```

---

## 13.5 response_format 控制工具输出

```python
# "content"（默认）：工具返回值直接作为 ToolMessage 内容
@tool(response_format="content")
def get_text() -> str:
    return "Some text"

# "content_and_artifact"：返回 (content, artifact) 元组
# content 发给 LLM，artifact 不发（用于存储大文件、图片等）
@tool(response_format="content_and_artifact")
def get_image() -> tuple[str, bytes]:
    image_data = load_image()
    return "Image loaded successfully", image_data

# 在工具内手动声明
from langchain_core.tools import tool, ToolArtifact

@tool(response_format="content_and_artifact")
def analyze_file(path: str) -> tuple[str, dict]:
    data = parse_file(path)
    summary = f"Parsed {len(data)} records"
    return summary, data  # LLM 看到 summary，data 存到 ToolMessage.artifact
```

---

## 13.6 langchain-community 预置工具

```bash
pip install langchain-community
```

```python
from langchain_community.tools import (
    DuckDuckGoSearchRun,     # 网络搜索
    WikipediaQueryRun,       # Wikipedia 查询
    PythonREPLTool,          # Python 代码执行
    ShellTool,               # 系统命令执行
    RequestsGetTool,         # HTTP GET 请求
)

# 直接使用
search = DuckDuckGoSearchRun()
wiki = WikipediaQueryRun(api_wrapper=WikipediaAPIWrapper())
python_repl = PythonREPLTool()

agent = create_agent(
    model=model,
    tools=[search, wiki, python_repl],
)
```

---

## 13.7 工具注入（依赖注入模式）

```python
from langchain_core.tools import InjectedToolArg
from typing import Annotated

@tool
def query_db(
    sql: str,
    db_connection: Annotated[Connection, InjectedToolArg]  # 注入，不暴露给 LLM
) -> str:
    """Execute a SQL query."""
    return db_connection.execute(sql)

# 传入注入的依赖
agent = create_agent(
    model=model,
    tools=[query_db],
    tool_kwargs={"db_connection": get_db_connection()}
)
```

---

## 13.8 与其他项目工具系统对比

| 特性 | LangChain | LangGraph | deer-flow | hermes-agent | nanobot | gstack |
|------|----------|-----------|-----------|-------------|---------|--------|
| **工具定义** | @tool / BaseTool | @tool / BaseTool | @tool / BaseTool | 无 | @tool (langchain) | SKILL.md 文档 |
| **Schema 验证** | Pydantic | Pydantic | Pydantic | N/A | Pydantic | 无（自然语言） |
| **并行执行** | ToolNode | ToolNode | ToolNode | N/A | 串行 | SKILL 独立进程 |
| **返回格式** | content / content_and_artifact | content / content_and_artifact | content | N/A | content | 文本/文件 |
| **依赖注入** | InjectedToolArg | InjectedToolArg | 无 | N/A | 无 | 无 |
| **缓存控制** | extras.cache_control | extras.cache_control | 无 | N/A | 无 | 无 |
| **第三方集成** | langchain-community (100+) | 需手动定义 | 有限 | N/A | 有限 | SKILL 生态 |
| **工具发现** | 代码注册 | 代码注册 | 代码注册 | N/A | 代码注册 | 文件系统搜索 |

**关键洞察：**
- LangChain/LangGraph 共享相同的 `BaseTool`/`@tool` 接口（来自 langchain-core）
- LangChain 独有：`langchain-community` 提供 100+ 预置工具集成
- gstack 的 SKILL 系统是独特的"文档驱动"方式，Claude 读 SKILL.md 决定如何操作
- `InjectedToolArg` 和 `extras` 是 LangChain 的高级特性
