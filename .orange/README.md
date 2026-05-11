# LangChain 项目分析文档索引

本目录包含 LangChain 框架的详细分析文档。

## 文档目录

| 文件 | 内容 |
|------|------|
| [01-产品介绍.md](./01-产品介绍.md) | 项目背景、架构演进（Classic → v1）、定位 |
| [02-功能结构.md](./02-功能结构.md) | AgentExecutor（旧）vs create_agent（新）、中间件 |
| [03-技术架构.md](./03-技术架构.md) | langchain-core、langchain-classic、langchain_v1 三层架构 |
| [04-代码结构.md](./04-代码结构.md) | 目录组织、核心文件、关键类说明 |
| [05-数据流.md](./05-数据流.md) | 旧版 AgentExecutor 流 vs 新版 create_agent 流 |
| [06-Agent设计.md](./06-Agent设计.md) | AgentMiddleware 8 钩子、ResponseFormat、工具系统 |
| [07-部署运行.md](./07-部署运行.md) | 安装、迁移路径、常见配置 |
| [08-记忆架构.md](./08-记忆架构.md) | 旧版 Memory 系统（废弃）vs LangGraph BaseStore |
| [09-多轮会话管理.md](./09-多轮会话管理.md) | 旧版 ConversationBufferMemory vs 新版 Checkpointer |
| [10-Lead-Sub-Agent.md](./10-Lead-Sub-Agent.md) | 旧版 Agent Chain vs 新版 LangGraph 子图 |
| [11-AgentLoop.md](./11-AgentLoop.md) | AgentExecutor while 循环 vs create_agent LangGraph Loop |
| [12-Thinking模式.md](./12-Thinking模式.md) | Extended Thinking 在 LangChain 中的支持 |
| [13-SKILL管理.md](./13-SKILL管理.md) | StructuredTool 工具系统 vs SKILL.md |
| [14-如何评测.md](./14-如何评测.md) | pytest 测试体系、迁移测试、性能基准 |

## 项目一句话

**LangChain** 是 AI 应用开发领域历史最悠久的框架，经历了从 "Chain+AgentExecutor" 到 "langchain_v1+create_agent+AgentMiddleware" 的重大架构重写，新版本完全基于 LangGraph 构建，提供统一的中间件系统和工具抽象，是连接 LLM API 与 Agent 框架的"标准库"。
