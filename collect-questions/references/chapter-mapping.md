# Chapter Mapping Reference

本文件提供 `questions/` 与 `solutions/` 的章节映射，供 `collect-questions` Skill 在归类题目时按需读取。

## Chapter List

### 01-fundamentals

- Agent 定义、核心组件、环境、自治性
- 感知-推理-行动循环
- Agentic AI vs. Generative AI
- LLM 作为 Agent 大脑
- Agent 与 Chatbot 区别
- Agent 局限性、涌现行为

### 02-frameworks

- LangChain、LangGraph、AutoGen、CrewAI、Dify、LlamaIndex
- OpenAI Assistants / Responses / Agents 相关框架式能力
- 框架对比、框架选型
- Observability、Tracing、框架集成体验

### 03-design-patterns

- ReAct
- Plan-and-Execute、Plan-and-Solve
- Reflexion、Reflection
- Chain-of-Thought、Tree-of-Thought、Graph-of-Thought
- LATS
- Router、Supervisor
- Re-planning、Self-correction、Human-in-the-Loop

### 04-system-design

- 生产级 Agent 架构
- 高并发、扩展性、容错
- 状态管理、中断恢复、任务队列
- 灰度发布、A/B 测试、成本控制、缓存
- 多区域部署、Agent Gateway

### 05-coding

- 手写 Agent、手写 Tool Loop
- Prompt 工程
- Streaming、结构化输出
- 单元测试、Mock、评估脚本
- 微调、数据集构建

### 06-multi-agent

- 多 Agent 通信模式
- 协调机制、角色分工、冲突解决
- Hub-and-Spoke、P2P、层级式拓扑
- 动态组队、社会模拟

### 07-memory-and-rag

- RAG 原理、RAG 流水线
- 短期记忆、长期记忆
- Embedding、向量数据库、Chunking
- Hybrid Search、Self-RAG、GraphRAG
- 知识图谱、上下文管理
- RAG 评估、RAG 幻觉
- Transformer 架构

### 08-tool-use

- Function Calling
- MCP
- Tool Registry、工具编排
- API 调用、工作流编排
- 代码执行、沙箱
- WebSocket、SSE
- 失败重试、降级与错误处理

### 09-evaluation-and-safety

- Agent 评估指标
- SWE-bench、WebArena、GAIA 等 benchmark
- Prompt Injection、防越权、权限控制
- Guardrails、审计日志、可解释性
- Alignment、可控性、安全架构

### 10-real-world-cases

- 客服 Agent、代码 Agent、数据分析 Agent
- 机器人、游戏、Computer Use
- Demo 到生产的迁移问题
- 延迟、成本、稳定性、失败恢复
- 行业案例、端侧 Agent

## Priority Rules

按以下顺序判断归类：

1. 用户显式指定的章节或主题。
2. 题目的主考点。
3. 与仓库当前章节结构最贴近的落点。

## Common Ambiguities

- 提到 LangGraph / AutoGen / CrewAI 的题，通常优先放 `02-frameworks`，即使里面也涉及多智能体。
- 提到 ReAct / Reflexion / Plan-and-Execute 的题，优先放 `03-design-patterns`，而不是 `05-coding`。
- 提到生产落地、成本、并发、状态恢复的题，优先放 `04-system-design`。
- 提到 RAG 时，如果重点是“检索、向量库、记忆、切块、召回、知识图谱”，优先放 `07-memory-and-rag`。
- 提到 MCP / Function Calling / Sandbox / Tool execution 的题，优先放 `08-tool-use`。
- 提到评测、安全、越权、注入攻击、审计的题，优先放 `09-evaluation-and-safety`。

## File Mapping

| Chapter | Questions File | Solutions File |
|---|---|---|
| 01-fundamentals | `questions/01-fundamentals/README.md` | `solutions/01-fundamentals.md` |
| 02-frameworks | `questions/02-frameworks/README.md` | `solutions/02-frameworks.md` |
| 03-design-patterns | `questions/03-design-patterns/README.md` | `solutions/03-design-patterns.md` |
| 04-system-design | `questions/04-system-design/README.md` | `solutions/04-system-design.md` |
| 05-coding | `questions/05-coding/README.md` | `solutions/05-coding.md` |
| 06-multi-agent | `questions/06-multi-agent/README.md` | `solutions/06-multi-agent.md` |
| 07-memory-and-rag | `questions/07-memory-and-rag/README.md` | `solutions/07-memory-and-rag.md` |
| 08-tool-use | `questions/08-tool-use/README.md` | `solutions/08-tool-use.md` |
| 09-evaluation-and-safety | `questions/09-evaluation-and-safety/README.md` | `solutions/09-evaluation-and-safety.md` |
| 10-real-world-cases | `questions/10-real-world-cases/README.md` | `solutions/10-real-world-cases.md` |
