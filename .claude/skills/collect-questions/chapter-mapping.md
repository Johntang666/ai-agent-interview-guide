# 章节映射参考 / Chapter Mapping Reference

本文件提供题目关键词到章节的详细映射关系，供 skill 分类时参考。

## 详细映射表

### 01-fundamentals (基础概念)
- Agent 定义、组件、架构
- 感知-推理-行动循环
- 自主性、涌现行为
- Agentic AI vs Generative AI
- LLM 作为 Agent 大脑
- Agent 与 Chatbot 区别
- Agent 局限性

### 02-frameworks (主流框架)
- LangChain, LangGraph
- AutoGen, CrewAI
- Dify, LlamaIndex
- OpenAI Assistants API
- Claude Agent SDK
- 框架对比、选型
- 可观测性 (Observability)

### 03-design-patterns (设计模式)
- ReAct (Reasoning + Acting)
- Plan-and-Execute, Plan-and-Solve
- Reflexion, Reflection
- Chain-of-Thought (CoT)
- Tree-of-Thought (ToT)
- Graph-of-Thought (GoT)
- LATS (Language Agent Tree Search)
- Router, Supervisor 模式
- Self-correction, Re-planning
- Human-in-the-Loop

### 04-system-design (系统设计)
- 生产级架构设计
- 可扩展性、高并发
- 状态管理、中断恢复
- 限流、成本控制
- 灰度发布、A/B 测试
- 缓存策略
- 跨地域部署
- Agent 网关

### 05-coding (编码实战)
- 手写 Agent 实现
- Prompt 工程
- 流式输出 (Streaming)
- 单元测试、Mock
- 评估框架实现
- Agent 微调 (Fine-tuning)
- 数据集收集与构建

### 06-multi-agent (多智能体)
- 多 Agent 通信模式
- 协调机制、冲突解决
- 拓扑结构 (Hub-Spoke, P2P)
- 层级化架构
- 动态组队
- 社会模拟
- 多 Agent 优势与复杂性

### 07-memory-and-rag (记忆与 RAG)
- RAG 原理、流水线
- 短期记忆、长期记忆
- 向量数据库、Embedding
- 文本切块 (Chunking)
- 混合检索 (Hybrid Search)
- Self-RAG, GraphRAG
- 知识图谱
- RAG 评估
- 上下文窗口问题
- RAG 幻觉问题
- Transformer 架构
- RAG vs 微调

### 08-tool-use (工具使用)
- Function Calling
- MCP (Model Context Protocol)
- 工具编排、Tool Registry
- API 调用
- 代码执行、沙箱
- WebSocket, SSE
- 错误处理、降级策略

### 09-evaluation-and-safety (评估与安全)
- Agent 评估指标
- Benchmark (SWE-bench, WebArena, GAIA)
- Prompt Injection 防御
- 权限控制、最小权限
- 安全架构 (沙箱、guardrails)
- 对齐、可控性
- 可解释性
- 审计日志

### 10-real-world-cases (真实案例)
- 客服 Agent、代码 Agent
- 搜索 Agent、数据分析 Agent
- 环境交互 (机器人、游戏)
- Demo 到生产的挑战
- 延迟与成本优化
- 失败恢复
- Computer Use Agent
- 端侧 Agent
