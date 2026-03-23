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

> 🤖 **Claude**

#### 1. 网络整合回答 / Comprehensive Answer

LangChain 和 LlamaIndex 是当前 LLM 应用开发中最主流的两大框架，但它们的设计哲学和核心定位有本质差异。LangChain 的核心抽象是 Chain（链）和 Agent（智能体），关注的是"如何把多个 LLM 调用、工具调用、条件分支串成一个可执行的工作流"。它后来演化出 LangGraph，引入了有向图和状态机的概念，使复杂 Agent 工作流具备了持久化状态、条件路由、人工介入等生产级能力。

LlamaIndex 的核心抽象是 Index（索引）和 Query Engine（查询引擎），关注的是"如何高效地把外部数据连接到 LLM"。它在文档解析、切块策略、索引构建、检索算法、重排序等数据链路上做得非常深入，是 RAG 场景的首选。

从架构层面看，LangChain 更像是一个"应用层编排框架"，而 LlamaIndex 更像是一个"数据层连接框架"。两者不是竞争关系，在大型项目中经常搭配使用：用 LlamaIndex 构建检索管线，用 LangGraph 编排整体 Agent 工作流。

此外还需关注其他框架的定位：AutoGen（微软）专注多 Agent 对话协作；CrewAI 强调角色扮演式多 Agent 编排；Dify 则是低代码可视化平台，适合快速交付。

#### 2. 结合实际例子 / Practical Example

```python
# LangChain/LangGraph: 适合有状态的多步骤 Agent 编排
from langgraph.graph import StateGraph, END
from typing import TypedDict

class AgentState(TypedDict):
    query: str
    retrieved_docs: list
    answer: str
    needs_review: bool

def retrieve_node(state: AgentState):
    """检索节点 - 可用 LlamaIndex 实现底层检索"""
    docs = vector_store.similarity_search(state["query"], k=5)
    return {"retrieved_docs": docs}

def generate_node(state: AgentState):
    """生成节点"""
    context = "\n".join(state["retrieved_docs"])
    answer = llm.invoke(f"基于以下内容回答：{context}\n问题：{state['query']}")
    return {"answer": answer, "needs_review": "不确定" in answer}

def review_router(state: AgentState):
    return "human_review" if state["needs_review"] else END

# 构建状态图
graph = StateGraph(AgentState)
graph.add_node("retrieve", retrieve_node)
graph.add_node("generate", generate_node)
graph.add_edge("retrieve", "generate")
graph.add_conditional_edges("generate", review_router)
```

```python
# LlamaIndex: 适合 RAG 数据管线构建
from llama_index.core import VectorStoreIndex, SimpleDirectoryReader
from llama_index.core.node_parser import SentenceSplitter

# 数据接入和索引 - LlamaIndex 的强项
documents = SimpleDirectoryReader("./knowledge_base").load_data()
parser = SentenceSplitter(chunk_size=512, chunk_overlap=50)
nodes = parser.get_nodes_from_documents(documents)
index = VectorStoreIndex(nodes)
query_engine = index.as_query_engine(similarity_top_k=5)
response = query_engine.query("公司差旅报销政策是什么？")
```

#### 3. 面试核心回答 / Core Interview Answer

- LangChain/LangGraph 的核心能力是工作流编排、状态管理和多步骤 Agent 控制流。
- LlamaIndex 的核心能力是数据连接、索引构建和检索优化，是 RAG 首选。
- 两者不是替代关系，大型项目中常组合使用：LlamaIndex 做数据层，LangGraph 做编排层。
- 选型关键看主矛盾：流程复杂选 LangGraph，数据链路复杂选 LlamaIndex。
- 其他框架各有定位：AutoGen 做多 Agent 对话，CrewAI 做角色协作，Dify 做低代码交付。

一句话总结：LangChain 解决"怎么串起来"，LlamaIndex 解决"怎么接数据"，真实项目往往两者都要。

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

> 🤖 **Claude**

#### 1. 网络整合回答 / Comprehensive Answer

这道题的本质不是考你"用过几个框架"，而是考你在真实场景中的技术决策能力。面试官关心的是：你为什么在特定场景下选了这个框架而不是那个，你怎么评估选择是否正确。

框架选型应该从需求出发，而不是从技术出发。可以把选型决策分成三个维度：

**维度一：任务复杂度**
- 单轮问答或简单 RAG → 直接用 OpenAI SDK + 向量数据库，不需要框架
- 多步骤工具调用、条件分支 → LangGraph 或手写状态机
- 多 Agent 协作、角色分工 → AutoGen、CrewAI 或自研调度层
- 快速原型/低代码团队 → Dify、Coze

**维度二：数据复杂度**
- 简单文档问答 → LlamaIndex 开箱即用
- 多源异构数据、需要精细 chunk 策略和 rerank → LlamaIndex 深度定制
- 数据不是瓶颈 → 不需要专门的数据框架

**维度三：工程约束**
- 团队熟悉度、社区活跃度、文档质量
- 是否需要持久化状态、分布式部署、可观测性
- 框架的 lock-in 程度和可替换性

评价指标不应只看"效果"，还要看"效率"和"可运营性"。常见指标矩阵：
- **效果**：任务成功率、工具调用准确率、检索命中率、答案 faithfulness
- **效率**：P50/P95 延迟、单任务 token 消耗、单任务成本
- **稳定性**：人工接管率、重试率、超时率、错误分布
- **可运营性**：调试耗时、trace 完整度、灰度能力

#### 2. 结合实际例子 / Practical Example

```python
# 一个真实的选型评估框架
class FrameworkEvaluation:
    """用于对比不同 Agent 框架在同一场景下的表现"""

    def __init__(self, test_cases: list[dict]):
        self.test_cases = test_cases  # 标准测试集
        self.metrics = {
            "task_success_rate": 0.0,     # 任务成功率
            "tool_call_accuracy": 0.0,    # 工具调用准确率
            "avg_latency_ms": 0.0,        # 平均延迟
            "avg_cost_usd": 0.0,          # 平均成本
            "human_escalation_rate": 0.0, # 人工接管率
        }

    def run_evaluation(self, agent_fn):
        results = []
        for case in self.test_cases:
            start = time.time()
            try:
                output = agent_fn(case["input"])
                latency = (time.time() - start) * 1000
                success = self.check_answer(output, case["expected"])
                results.append({
                    "success": success,
                    "latency_ms": latency,
                    "cost": output.get("total_cost", 0),
                    "escalated": output.get("escalated", False),
                })
            except Exception as e:
                results.append({"success": False, "error": str(e)})

        # 汇总指标
        self.metrics["task_success_rate"] = (
            sum(r["success"] for r in results) / len(results)
        )
        return self.metrics
```

实际选型过程通常是：先用最简单的方案（SDK + prompt）做 baseline，然后在 baseline 不够时引入框架，每次引入新框架都要跑相同测试集做对比。

#### 3. 面试核心回答 / Core Interview Answer

- 选型要从需求出发：任务复杂度决定编排框架，数据复杂度决定 RAG 框架。
- 不要为了用框架而用框架，简单场景直接 SDK + prompt 足够。
- 评价指标必须覆盖效果、效率、成本、稳定性四个维度。
- 最重要的指标往往不是准确率，而是人工接管率和可调试性。
- 框架选型是动态的，应该随业务复杂度升级而逐步引入。

一句话总结：框架选型的核心问题不是"哪个最强"，而是"哪个在你的约束下 ROI 最高"。
