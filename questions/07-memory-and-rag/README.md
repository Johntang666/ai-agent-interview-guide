# 07 - 记忆与 RAG / Memory & RAG

## 题目列表 / Questions

### 记忆机制 / Memory Mechanisms

1. **Agent 的记忆系统可以分为哪几种类型？各自的实现方式是什么？**
   What types of memory exist in Agent systems? How is each implemented?

2. **如何实现 Agent 的长期记忆持久化？有哪些存储方案？**
   How do you persist Agent long-term memory? What storage solutions exist?

3. **对话历史过长时如何处理？对比截断、摘要、和滑动窗口方案。**
   How do you handle overly long conversation history? Compare truncation, summarization, and sliding window.

4. **如何设计 Agent 的 "情景记忆（Episodic Memory）"？**
   How do you design Agent "Episodic Memory"?

5. **Mem0、Zep 等 Agent 记忆中间件有什么设计思路？**
   What are the design ideas behind Agent memory middleware like Mem0 and Zep?

### RAG 架构 / RAG Architecture

6. **解释基础 RAG 管线的完整流程：索引 → 检索 → 生成。**
   Explain the full basic RAG pipeline: Indexing → Retrieval → Generation.

7. **什么是 Advanced RAG？与 Naive RAG 相比有哪些改进？**
   What is Advanced RAG? What improvements does it have over Naive RAG?

8. **如何选择和优化 Embedding 模型？有哪些评估指标？**
   How do you choose and optimize Embedding models? What evaluation metrics exist?

9. **对比不同的文档切分（Chunking）策略及其影响。**
   Compare different document chunking strategies and their impacts.

10. **如何实现混合检索（Hybrid Search）？BM25 + 向量检索的融合策略。**
    How do you implement Hybrid Search? Fusion strategies for BM25 + vector retrieval.

### 高级话题 / Advanced Topics

11. **什么是 Self-RAG？它如何让 Agent 自主决定是否需要检索？**
    What is Self-RAG? How does it let the Agent autonomously decide when to retrieve?

12. **如何处理 RAG 中的知识冲突（Knowledge Conflict）？**
    How do you handle knowledge conflicts in RAG?

13. **Graph RAG 的核心思想是什么？与传统向量 RAG 有何区别？**
    What is the core idea of Graph RAG? How does it differ from traditional vector RAG?
