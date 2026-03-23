# 02 - 主流框架 / Frameworks

本文件收录主流框架章节的参考答案与深度解析。

---

### Q: 请比较一下两个流行的 Agent 开发框架，如 LangChain 和 LlamaIndex。它们的核心应用场景有何不同？

#### 1. 网络整合回答 / Comprehensive Answer

LangChain 更偏向“应用编排层”，擅长把模型、Prompt、Tool、Memory、Agent Loop 串成工作流；它的优势是生态广、组件多、和 LangGraph 搭配后适合做有状态的 Agent 编排。  
LlamaIndex 更偏向“数据连接与知识增强层”，最强的场景通常是 RAG、知识库问答、文档索引、数据源接入以及检索增强。它把“数据怎么进来、怎么切、怎么索引、怎么查回来”做得更系统。  
所以两者不是简单替代关系。若核心问题是多步骤任务编排、工具调用、状态控制，优先看 LangChain / LangGraph；若核心问题是知识接入、检索质量、数据连接器，优先看 LlamaIndex。很多生产系统里两者甚至会混用。

#### 2. 结合实际例子 / Practical Example

一个企业知识助理可以这样选型：

- 如果重点是“接 20 个内部知识源，提升召回率和引用质量”，更适合以 LlamaIndex 为核心。
- 如果重点是“先检索，再审批，再调用 CRM 工具，再生成邮件”，更适合以 LangGraph 为主流程编排。
- 实际落地时，也可以用 LlamaIndex 做 indexing / retrieval，再把检索结果接入 LangChain Agent。

#### 3. 面试核心回答 / Core Interview Answer

- LangChain 强在 Agent 编排和工具链整合。
- LlamaIndex 强在 RAG、数据连接和索引检索。
- 选型不要只看热度，要看你的主矛盾是“流程编排”还是“知识接入”。

一句话总结：LangChain 更像 orchestration framework，LlamaIndex 更像 data and retrieval framework。

---

### Q: 你用过哪些 Agent 框架？选型是如何选的？你最终场景的评价指标是什么？

#### 1. 网络整合回答 / Comprehensive Answer

这类题面试里不要只报框架名称，而要讲清楚“为什么选”。一个稳妥的回答方式是：按场景把框架分成三类，轻量原型、复杂工作流、知识增强。  
例如，快速验证想法时可以直接手写 loop 或选 LangChain；当任务存在状态机、分支、重试、人工介入时，更适合 LangGraph；如果场景以知识库检索为主，则更适合 LlamaIndex。低代码团队则可能选 Dify，强调交付效率。  
最终评价指标通常不是“模型答得像不像”，而是任务成功率、工具调用准确率、检索命中率、延迟、成本、可观测性和失败恢复能力。生产里真正决定选型的，往往是状态管理、调试成本和团队维护复杂度。

#### 2. 结合实际例子 / Practical Example

可以把回答组织成一个真实项目模板：

- 第一阶段用 Python + LangChain 快速做 PoC，验证用户价值。
- 第二阶段发现流程有审批、分支和重试，迁移到 LangGraph 管理状态图。
- 如果知识库成为瓶颈，再引入 LlamaIndex 优化数据接入和检索层。
- 评估指标则看任务成功率、单任务平均成本、P95 延迟、人工接管率和线上错误率。

#### 3. 面试核心回答 / Core Interview Answer

- 框架选型看主场景：编排、RAG、多 Agent、还是低代码交付。
- 指标至少要覆盖效果、效率、成本和稳定性。
- 生产环境里，状态管理和可观测性往往比“Demo 是否惊艳”更重要。

一句话总结：Agent 框架选型本质上是工程权衡，不是功能表对比。
