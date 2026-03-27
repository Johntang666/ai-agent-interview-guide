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

### 补充题目 / Additional Questions

14. **Memory 是 Agent 的一个关键模块。请问如何为 Agent 设计短期记忆和长期记忆系统？可以借助哪些外部工具或技术？**

15. **Agent 的短期记忆和长期记忆应该如何设计和配合？**

16. **请解释 RAG 的工作原理。与直接对 LLM 进行微调相比，RAG 主要解决了什么问题？有哪些优势？**
    Explain how RAG works. Compared with directly fine-tuning an LLM, what problems does RAG mainly solve and what advantages does it offer?

17. **RAG 是如何缓解 LLM 上下文窗口有限问题的？**
    How does RAG mitigate the limitations of an LLM's context window?

18. **除了基础的向量检索，你还知道哪些可以提升 RAG 检索质量的技术？**
    Beyond basic vector retrieval, what techniques can improve retrieval quality in RAG?

19. **如何全面评估一个 RAG 系统的性能？请分别从检索和生成两个阶段提出指标。**
    How do you comprehensively evaluate a RAG system? Propose metrics for both retrieval and generation stages.

20. **在什么场景下，你会选择图数据库或知识图谱来增强或替代传统向量数据库检索？**
    In what scenarios would you use a graph database or knowledge graph to enhance or replace traditional vector database retrieval?

21. **除了”先检索后生成”，你是否了解在生成过程中多次检索或自适应检索等更复杂的 RAG 范式？**
    Beyond "retrieve then generate", are you familiar with more advanced RAG paradigms such as iterative retrieval during generation or adaptive retrieval?

22. **RAG 系统在实际部署中可能面临哪些挑战？**
    What challenges can a RAG system face in real-world deployment?

23. **什么是 RAG 中的”幻觉”问题？如何预防？**
    What is the hallucination problem in RAG, and how do you prevent it?

24. **如果 RAG 系统返回 0 个检索结果，你会如何排查问题？**
    If a RAG system returns zero retrieval results, how would you troubleshoot it?

25. **了解 Transformer 吗？它与 RAG / Embedding 系统的关系是什么？**
    Are you familiar with Transformers? What is their relationship to RAG and embedding systems?

26. **Agent 使用向量数据库作为长期记忆时，如何处理记忆过时（Memory Staleness）问题？用户偏好变化时怎么办？**

27. **什么是 RAG-Fusion？其核心技术和工作流程是什么？与标准 RAG 相比有哪些优势和不足？**

28. **使用纯 LLM 上下文窗口作为主要记忆存储 vs 外部向量数据库，各有什么权衡？**

29. **什么是 Prompt Compression 和 Context Caching？它们如何优化 Agent 的性能和成本？**

30. **什么是 HyDE（Hypothetical Document Embeddings）？它如何提升零样本场景下的 RAG 检索质量？**

31. **搜索引擎与 RAG 系统有什么本质区别？各自的优势场景是什么？**

32. **RAG 中的 "Lost in the Middle" 现象是什么？它如何影响生成质量？有哪些缓解策略？**

33. **AI Agent 中 Memory 和 RAG 有哪些区别？二者分别解决什么问题，应该如何组合使用？**
