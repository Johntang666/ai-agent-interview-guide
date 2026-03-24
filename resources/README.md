# 学习资源 / Learning Resources

## 经典论文 / Key Papers

| 论文 / Paper | 主题 / Topic | 链接 / Link |
|-------------|-------------|-------------|
| ReAct (Yao et al., 2022) | Reasoning + Acting | arXiv:2210.03629 |
| Reflexion (Shinn et al., 2023) | Self-reflection | arXiv:2303.11366 |
| Toolformer (Schick et al., 2023) | Tool learning | arXiv:2302.04761 |
| Generative Agents (Park et al., 2023) | Agent simulation | arXiv:2304.03442 |
| LATS (Zhou et al., 2023) | Tree search + LLM | arXiv:2310.04406 |
| A Survey on LLM-based Agents (Wang et al., 2023) | Comprehensive survey | arXiv:2308.11432 |
| Self-RAG (Asai et al., 2023) | Adaptive retrieval | arXiv:2310.11511 |

## 开源框架 / Open-Source Frameworks

| 框架 / Framework | 简介 / Description |
|-----------------|-------------------|
| LangChain / LangGraph | LLM 应用开发框架 / LLM application framework |
| AutoGen | 微软多 Agent 对话框架 / Microsoft multi-agent framework |
| CrewAI | 角色驱动的多 Agent 框架 / Role-driven multi-agent framework |
| Dify | 开源 LLM 应用平台 / Open-source LLM app platform |
| Claude Agent SDK | Anthropic Agent 开发 SDK / Anthropic Agent SDK |
| OpenAI Agents SDK | OpenAI Agent 开发 SDK / OpenAI Agent SDK |

## 推荐课程 / Recommended Courses

- Andrew Ng - AI Agentic Design Patterns with AutoGen
- DeepLearning.AI - Building Agentic RAG with LlamaIndex
- LangChain Academy - Introduction to LangGraph

## 博客与文章 / Blogs & Articles

- Lilian Weng - LLM Powered Autonomous Agents
- Anthropic - Building effective agents
- LangChain Blog - Agent architectures

## 面试专题参考 / Interview Prep References

### OpenClaw + AI Agent 面试八股文

- 链接：[OpenClaw + AI Agent 面试八股文：背完这篇，你懂的比面试官还多！](https://zhuanlan.zhihu.com/p/2013536456132554764)
- 类型：中文专题长文，偏面试准备与架构梳理
- 适合搭配章节：
  - `01-fundamentals`
  - `02-frameworks`
  - `03-design-patterns`
  - `07-memory-and-rag`
  - `08-tool-use`
  - `10-real-world-cases`

#### 参考说明

- 这篇文章适合用来快速建立对 OpenClaw 的整体认知，尤其是“Agent 执行框架”和“普通对话工具”的区别。
- 文中把 OpenClaw 拆成 Gateway、Brain、Memory、Skills、Heartbeat 五个组件，比较适合用于组织面试回答结构。
- 对本地优先、消息平台接入、Skill 组织方式、Memory 分层、WebSocket 网关等工程点有较强的面试导向，适合做项目表达训练。
- 文章本身偏“面试八股整理”，适合拿来建立答题框架，但具体实现细节、项目热度和趋势类表述，建议结合 OpenClaw 官方仓库与文档再做交叉确认。
