# 04 - 系统设计 / System Design

本文件收录系统设计章节的参考答案与深度解析。

---

### Q: 如何设计 Agent 的可观测性系统？解释 Tracing 和 Spans 在 Agent 调试中的作用。

> 🤖 **Claude**

#### 1. 网络整合回答 / Comprehensive Answer

Agent 的可观测性系统是生产环境中不可或缺的基础设施，其核心目标是让开发者能够完整地理解 Agent 在每一次运行中"做了什么、为什么这样做、在哪里出了问题"。与传统软件的日志监控不同，Agent 系统的可观测性面临独特挑战：LLM 的推理过程是黑箱的，工具调用链路是动态的，多轮交互的状态转换是非确定性的。因此，仅靠打印 LLM 的文本输出远远不够——Agent trace 比最终输出更重要，因为它记录了完整的决策链路和所有中间状态。

在分布式跟踪（Distributed Tracing）的理念下，Agent 可观测性系统的核心概念是 Trace 和 Span。一个 Trace 代表一次完整的 Agent 执行过程，从接收用户请求到返回最终结果的全链路记录。每个 Trace 由多个 Span 组成，每个 Span 代表一次具体操作，例如一次 LLM 调用、一次向量检索、一次 Embedding 计算、一次外部 API 调用等。每个 Span 会捕获该操作的输入参数、输出结果、耗时（latency）、Token 消耗与成本（cost）、错误信息以及自定义元数据。Span 之间通过父子关系形成嵌套树结构，这使得开发者可以在多 Agent 协作环境下实现细粒度的调试和根因分析——比如发现是某个子 Agent 的第三次工具调用返回了异常数据，导致整条推理链路偏离了预期。

目前主流的 Agent 可观测性工具包括 LangSmith（LangChain 官方平台，提供完整的 Agent 行为可视化，支持 Python、TypeScript、Go、Java 等多语言 SDK）、Langfuse（开源替代方案，支持自托管）、Helicone（侧重 LLM API 的代理层监控和成本分析）、以及 Phoenix（Arize AI 推出的开源可观测性工具，深度集成 OpenTelemetry 标准）。选择工具时需要考虑团队技术栈、部署模式（SaaS vs 自托管）、数据隐私要求以及与现有 CI/CD 流程的集成能力。

#### 2. 结合实际例子 / Practical Example

以 LangSmith 为例，展示如何为一个 RAG Agent 添加完整的 Tracing 支持：

```python
# 安装：pip install langsmith langchain langchain-openai
import os
from langsmith import traceable, Client
from langchain_openai import ChatOpenAI
from langchain.schema import HumanMessage

# 配置 LangSmith 环境变量
os.environ["LANGCHAIN_TRACING_V2"] = "true"
os.environ["LANGCHAIN_API_KEY"] = "your-langsmith-api-key"
os.environ["LANGCHAIN_PROJECT"] = "rag-agent-production"

# 使用 @traceable 装饰器创建自定义 Span
@traceable(name="document_retrieval", run_type="retriever")
def retrieve_documents(query: str, top_k: int = 5) -> list[dict]:
    """检索相关文档 —— 这会生成一个 retriever 类型的 Span"""
    # 模拟向量检索
    results = vector_store.similarity_search(query, k=top_k)
    return [{"content": doc.page_content, "score": doc.metadata.get("score", 0)}
            for doc in results]

@traceable(name="answer_generation", run_type="llm")
def generate_answer(query: str, context: str) -> str:
    """生成回答 —— 这会生成一个 llm 类型的 Span"""
    llm = ChatOpenAI(model="gpt-4", temperature=0)
    prompt = f"根据以下上下文回答问题。\n\n上下文：{context}\n\n问题：{query}"
    response = llm.invoke([HumanMessage(content=prompt)])
    return response.content

@traceable(name="rag_agent_run", run_type="chain")
def rag_agent(user_query: str) -> dict:
    """
    完整的 RAG Agent 执行链路 —— 这是顶层 Span (root span)

    生成的 Trace 结构如下：
    rag_agent_run (chain)          ← Root Span，记录整体耗时和最终结果
    ├── document_retrieval (retriever)  ← 子 Span，记录检索耗时和返回文档数
    └── answer_generation (llm)         ← 子 Span，记录 Token 消耗和 LLM 延迟
    """
    # Step 1: 检索（生成子 Span）
    docs = retrieve_documents(user_query)
    context = "\n".join([d["content"] for d in docs])

    # Step 2: 生成回答（生成子 Span）
    answer = generate_answer(user_query, context)

    return {
        "answer": answer,
        "sources": docs,
        "num_docs_retrieved": len(docs)
    }

# 执行后，在 LangSmith 控制台可以看到完整的 Trace 树：
# - 每个 Span 的输入/输出
# - 各步骤的延迟分布（如 retrieval 200ms, LLM 1500ms）
# - Token 使用量和预估成本
# - 如果出错，精确定位到哪个 Span 抛出了异常
result = rag_agent("公司的年假政策是什么？")
```

如果需要更底层的控制（不依赖 LangChain 生态），可以直接用 OpenTelemetry 标准来实现：

```python
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor

# 初始化 OpenTelemetry
provider = TracerProvider()
# 可以对接 Jaeger、Zipkin 或任何 OTLP 兼容后端
provider.add_span_processor(BatchSpanProcessor(exporter))
trace.set_tracer_provider(provider)
tracer = trace.get_tracer("agent-service")

def agent_run(query: str):
    with tracer.start_as_current_span("agent_execution") as root_span:
        root_span.set_attribute("user.query", query)

        with tracer.start_as_current_span("planning") as plan_span:
            plan = planner.create_plan(query)
            plan_span.set_attribute("plan.steps", len(plan.steps))

        for i, step in enumerate(plan.steps):
            with tracer.start_as_current_span(f"tool_call_{i}") as tool_span:
                tool_span.set_attribute("tool.name", step.tool_name)
                tool_span.set_attribute("tool.input", str(step.input))
                result = execute_tool(step)
                tool_span.set_attribute("tool.output_length", len(str(result)))
                if result.error:
                    tool_span.set_status(StatusCode.ERROR, result.error)
```

#### 3. 面试核心回答 / Core Interview Answer

- **Trace 是全链路视图**：一个 Trace 对应一次完整的 Agent 执行过程，包含从请求到响应的所有操作记录，是调试和性能分析的基础单位。
- **Span 是原子操作记录**：每个 Span 代表一次具体操作（LLM 调用、检索、工具执行），捕获输入、输出、耗时、成本和错误信息，Span 之间形成嵌套树结构反映调用层级。
- **Agent trace 比文本输出更重要**：LLM 的最终回答只是冰山一角，trace 记录了完整的决策链路和中间状态，是理解 Agent 行为和定位问题的关键。
- **深度可观测性支撑多 Agent 调试**：在多 Agent 协作场景中，嵌套 Span 能精确定位到是哪个 Agent 的哪次操作导致了问题，实现细粒度的根因分析。
- **工具选型要匹配团队需求**：LangSmith 适合 LangChain 生态用户，Langfuse 适合需要自托管的团队，Helicone 适合关注成本控制的场景，Phoenix 适合已有 OpenTelemetry 基础设施的团队。

一句话总结：Agent 可观测性的核心是通过 Trace + Span 的嵌套结构，将 Agent 的"黑箱"执行过程变成可视化、可分析、可回溯的白箱链路，让调试从"猜"变成"看"。

---

### Q: 如何设计一个 Agent 来处理超长时间跨度的任务（如"研究一家公司的完整历史并撰写报告"）？如何防止 Agent 迷失或卡住？

> 🤖 **Claude**

#### 1. 网络整合回答 / Comprehensive Answer

处理超长时间跨度任务是 Agent 系统设计中最具挑战性的问题之一。与简单的单轮问答不同，这类任务可能持续数小时甚至数天，涉及数十次甚至上百次的 LLM 调用和工具执行。核心难题在于：LLM 的上下文窗口有限，Agent 容易在长链路中"忘记"初始目标；执行过程中的任何一步失败都可能导致整体任务停滞；随着信息不断积累，上下文窗口会溢出，推理质量会下降。

解决这些问题需要从架构层面进行系统性设计。整体流程遵循一个核心循环：Request Intake（接收请求）→ Context Assembly（上下文组装）→ LLM Reasoning（推理决策）→ Action Validation（行动验证）→ Sandboxed Execution（沙箱执行）→ Result Processing（结果处理）→ State Update（状态更新）→ Loop or Terminate（继续循环或终止）。

第一个关键设计是**任务分解与里程碑管理**。Agent 在接收到长任务后，首先应该将其拆解为多个里程碑（milestones），每个里程碑再细分为可执行的子任务。例如"研究一家公司的完整历史"可以拆分为：确定公司基本信息 → 收集创立与早期发展资料 → 研究关键产品线演变 → 分析重大事件与转型 → 汇总财务数据趋势 → 撰写报告草稿 → 审校与定稿。每个里程碑有明确的完成标准和预期产出。

第二个关键设计是**状态检查点（Checkpointing）**。Agent 在每完成一个子任务后，应将当前进度持久化保存，包括已完成的步骤、收集到的关键信息摘要、当前所处的阶段等。这样当系统崩溃、超时或需要暂停时，可以从最近的检查点恢复，而不是从头开始。

第三个关键设计是**上下文管理策略**。随着任务推进，积累的信息量会远超 LLM 的上下文窗口。解决方案是使用"摘要压缩"机制——每完成一个阶段，将该阶段的详细执行记录压缩为结构化摘要，只保留关键结论和数据。当前活跃阶段保留详细上下文，历史阶段只保留摘要，从而在有限窗口内维持足够的信息密度。

第四个关键设计是**超时与回退策略**。为每个子任务设置最大执行时间和最大重试次数，当 Agent 在某个步骤上卡住时（例如反复调用同一工具得到相同错误），触发回退机制——可以尝试替代方案、跳过当前步骤、降级处理（用已有信息生成部分结果），或者请求人工介入。

#### 2. 结合实际例子 / Practical Example

以下是一个处理长时间任务的 Agent 架构实现：

```python
import json
import time
from dataclasses import dataclass, field
from enum import Enum
from typing import Optional

class TaskStatus(Enum):
    PENDING = "pending"
    IN_PROGRESS = "in_progress"
    COMPLETED = "completed"
    FAILED = "failed"
    SKIPPED = "skipped"

@dataclass
class Checkpoint:
    """检查点：记录 Agent 当前状态，支持断点恢复"""
    milestone_index: int
    subtask_index: int
    collected_data: dict
    summaries: list[str]       # 已完成阶段的压缩摘要
    timestamp: float = field(default_factory=time.time)

    def save(self, path: str):
        with open(path, "w") as f:
            json.dump(self.__dict__, f, ensure_ascii=False, indent=2)

    @classmethod
    def load(cls, path: str) -> "Checkpoint":
        with open(path, "r") as f:
            return cls(**json.load(f))

@dataclass
class SubTask:
    name: str
    description: str
    status: TaskStatus = TaskStatus.PENDING
    result: Optional[str] = None
    retries: int = 0
    max_retries: int = 3
    timeout_seconds: int = 300  # 每个子任务最多 5 分钟

class LongRunningAgent:
    """处理超长时间跨度任务的 Agent"""

    def __init__(self, llm, tools, checkpoint_path="checkpoint.json"):
        self.llm = llm
        self.tools = tools
        self.checkpoint_path = checkpoint_path
        self.summaries = []        # 历史阶段的压缩摘要
        self.current_context = ""  # 当前阶段的详细上下文

    def decompose_task(self, task_description: str) -> list[list[SubTask]]:
        """第一步：让 LLM 将长任务分解为里程碑和子任务"""
        prompt = f"""将以下任务分解为里程碑和子任务，以 JSON 格式返回。
        任务：{task_description}
        格式：[{{"milestone": "名称", "subtasks": ["子任务1", "子任务2"]}}]"""
        plan = self.llm.invoke(prompt)
        # 解析返回的计划并构建结构化的任务列表
        milestones = json.loads(plan)
        return [
            [SubTask(name=st, description=st) for st in m["subtasks"]]
            for m in milestones
        ]

    def compress_context(self, detailed_context: str) -> str:
        """上下文压缩：将阶段详情压缩为摘要，防止上下文溢出"""
        prompt = f"""将以下执行记录压缩为简洁摘要，只保留关键结论和数据：
        {detailed_context}"""
        return self.llm.invoke(prompt)

    def build_context(self, current_task: SubTask) -> str:
        """组装上下文：历史摘要 + 当前详细上下文"""
        history = "\n".join([f"[已完成] {s}" for s in self.summaries])
        return f"""## 历史进度摘要
{history}

## 当前阶段详细上下文
{self.current_context}

## 当前任务
{current_task.name}: {current_task.description}"""

    def execute_subtask(self, subtask: SubTask) -> bool:
        """执行单个子任务，带超时和重试"""
        while subtask.retries < subtask.max_retries:
            try:
                context = self.build_context(subtask)
                start_time = time.time()

                # LLM 推理 + 工具调用
                result = self.llm.invoke_with_tools(
                    context=context,
                    tools=self.tools,
                    timeout=subtask.timeout_seconds
                )

                elapsed = time.time() - start_time
                if elapsed > subtask.timeout_seconds:
                    raise TimeoutError(f"子任务超时：{elapsed:.1f}s")

                subtask.result = result
                subtask.status = TaskStatus.COMPLETED
                self.current_context += f"\n[完成] {subtask.name}: {result}"
                return True

            except Exception as e:
                subtask.retries += 1
                if subtask.retries >= subtask.max_retries:
                    # 回退策略：标记为失败，跳过继续
                    subtask.status = TaskStatus.SKIPPED
                    self.current_context += f"\n[跳过] {subtask.name}: 重试{subtask.max_retries}次后失败"
                    return False
        return False

    def run(self, task_description: str):
        """主执行循环"""
        # 尝试从检查点恢复
        checkpoint = self._try_load_checkpoint()

        if checkpoint:
            milestones = self.decompose_task(task_description)  # 重新分解
            self.summaries = checkpoint.summaries
            start_m, start_s = checkpoint.milestone_index, checkpoint.subtask_index
        else:
            milestones = self.decompose_task(task_description)
            start_m, start_s = 0, 0

        for m_idx in range(start_m, len(milestones)):
            subtasks = milestones[m_idx]
            s_start = start_s if m_idx == start_m else 0

            for s_idx in range(s_start, len(subtasks)):
                self.execute_subtask(subtasks[s_idx])

                # 每完成一个子任务，保存检查点
                Checkpoint(
                    milestone_index=m_idx,
                    subtask_index=s_idx + 1,
                    collected_data={},
                    summaries=self.summaries
                ).save(self.checkpoint_path)

            # 里程碑完成：压缩当前阶段上下文为摘要
            summary = self.compress_context(self.current_context)
            self.summaries.append(summary)
            self.current_context = ""  # 清空，为下一阶段腾出上下文空间

        # 最终汇总：基于所有里程碑摘要生成报告
        return self.llm.invoke(
            f"基于以下各阶段研究摘要，撰写完整报告：\n" +
            "\n".join(self.summaries)
        )
```

#### 3. 面试核心回答 / Core Interview Answer

- **任务分解是第一步**：将超长任务拆分为里程碑和子任务，每个子任务有明确的完成标准，Agent 不会面对一个模糊的大目标而迷失方向。
- **检查点机制防止前功尽弃**：定期将进度持久化保存（milestone 索引、已收集数据、历史摘要），系统崩溃或超时后可以从断点恢复，而非从头开始。
- **上下文压缩解决窗口限制**：已完成阶段的详细记录压缩为摘要，当前阶段保留完整上下文，在有限的上下文窗口内维持信息密度和推理质量。
- **超时 + 重试 + 回退三层防卡机制**：每个子任务设置超时和最大重试次数，卡住后自动降级（跳过非关键步骤、用已有信息生成部分结果、请求人工介入），确保整体任务不会因单点失败而停滞。

一句话总结：处理超长任务的核心思路是"化整为零、步步为营"——通过任务分解降低复杂度，通过检查点保护进度，通过上下文压缩控制信息量，通过回退策略防止卡死。

---

### Q: Agent 系统的工程化挑战：为什么说 AI 只完成了 30% 的工作，剩余 70% 是工具工程？如何设计可靠的工具工程架构？

> 🤖 **Claude**

#### 1. 网络整合回答 / Comprehensive Answer

阿里云在其生产级 Agent 系统实践中总结出一个重要观察：AI（LLM 推理）只完成了整个 Agent 系统 30% 的工作，剩余 70% 的工作量落在了工具工程上。这个比例打破了很多人对 Agent 系统的认知——大多数人以为 Agent 的核心难题是"让 LLM 更聪明"，但实际上在生产环境中，真正消耗工程资源的是围绕 LLM 的工具生态建设。

为什么会这样？因为 LLM 本身只能做推理和生成文本，但一个 Agent 要完成实际任务，必须依赖大量外部工具——数据库查询、API 调用、文件操作、代码执行、第三方服务集成等。这些工具的可靠性直接决定了 Agent 系统的可靠性。工具工程的核心挑战包括四个方面：

第一，**设计对 AI 友好的反馈接口**。工具不能只返回原始数据或晦涩的错误码，它需要返回 LLM 能理解的结构化反馈——成功时给出清晰的结果摘要，失败时给出人类可读的错误描述和建议的下一步操作。这本质上是在为 LLM 设计 UX。

第二，**高效管理上下文**。Agent 在一次任务中可能调用数十个工具，每个工具的输入输出都会占用上下文窗口。需要智能地决定哪些工具返回值应该完整保留、哪些应该压缩、哪些可以丢弃。

第三，**处理部分失败（Partial Failure）**。在一个包含 10 个步骤的任务中，第 7 步的工具调用失败了——系统应该重试、跳过、回滚还是用替代方案？这需要精心设计的错误恢复策略。

第四，**构建适配 AI 的错误恢复机制**。传统软件的错误处理是面向程序员的（抛异常、写日志），但 Agent 系统的错误处理必须面向 LLM——让 LLM 能根据错误信息自主决定如何恢复，而不是简单地中断执行。

在架构层面，工具工程的设计需要根据规模分级考虑。小规模场景可以采用单服务器部署（LLM 服务 + 向量数据库 + MCP 服务器 all-in-one），适合 PoC 和小团队。中大规模场景需要微服务架构，将 LLM 服务、向量数据库集群、MCP 服务集群、API 网关分别独立部署，通过消息队列解耦。高可用场景则需要多区域部署、服务熔断机制（某个工具不可用时自动隔离，不影响其他功能）、降级策略（主力 LLM 不可用时切换备用模型）。状态管理通常使用 Redis 或本地内存记录会话上下文和工具调用状态。发布策略上，灰度发布是必须的——先给新版本工具或新模型分配 1% 的流量，对比关键指标（成功率、延迟、用户满意度）后逐步放量。

#### 2. 结合实际例子 / Practical Example

以下展示一个生产级工具工程架构的核心组件实现：

```python
from dataclasses import dataclass
from enum import Enum
from typing import Any, Callable, Optional
import time
import logging

logger = logging.getLogger("tool_engine")

# ============ 1. AI 友好的工具返回格式 ============

class ToolResultStatus(Enum):
    SUCCESS = "success"
    PARTIAL = "partial"       # 部分成功
    RETRYABLE_ERROR = "retryable_error"
    FATAL_ERROR = "fatal_error"

@dataclass
class ToolResult:
    """对 LLM 友好的工具返回结构"""
    status: ToolResultStatus
    data: Any                          # 实际返回数据
    summary: str                       # LLM 可直接使用的结果摘要
    error_message: Optional[str] = None
    suggested_action: Optional[str] = None  # 失败时建议 LLM 做什么
    context_size: int = 0              # 返回数据的 Token 估算

    def to_llm_message(self) -> str:
        """转换为 LLM 能直接消费的文本"""
        if self.status == ToolResultStatus.SUCCESS:
            return f"[工具调用成功] {self.summary}"
        elif self.status == ToolResultStatus.RETRYABLE_ERROR:
            return (f"[工具调用失败-可重试] {self.error_message}\n"
                    f"建议操作：{self.suggested_action}")
        elif self.status == ToolResultStatus.FATAL_ERROR:
            return (f"[工具调用失败-不可恢复] {self.error_message}\n"
                    f"建议操作：{self.suggested_action}")
        return f"[部分成功] {self.summary}"


# ============ 2. 熔断器：防止故障工具拖垮整个系统 ============

class CircuitBreaker:
    """服务熔断器：连续失败达到阈值后自动断开，防止雪崩"""

    def __init__(self, failure_threshold: int = 5, recovery_timeout: int = 60):
        self.failure_threshold = failure_threshold
        self.recovery_timeout = recovery_timeout
        self.failure_count = 0
        self.last_failure_time = 0
        self.is_open = False  # True = 熔断中，拒绝请求

    def record_success(self):
        self.failure_count = 0
        self.is_open = False

    def record_failure(self):
        self.failure_count += 1
        self.last_failure_time = time.time()
        if self.failure_count >= self.failure_threshold:
            self.is_open = True
            logger.warning(f"熔断器开启：连续失败 {self.failure_count} 次")

    def allow_request(self) -> bool:
        if not self.is_open:
            return True
        # 熔断后经过 recovery_timeout，允许一次试探性请求
        if time.time() - self.last_failure_time > self.recovery_timeout:
            return True
        return False


# ============ 3. 工具注册与执行引擎 ============

class ToolEngine:
    """生产级工具执行引擎"""

    def __init__(self):
        self.tools: dict[str, Callable] = {}
        self.fallbacks: dict[str, Callable] = {}     # 降级方案
        self.breakers: dict[str, CircuitBreaker] = {}
        self.call_stats: dict[str, dict] = {}         # 调用统计

    def register(self, name: str, func: Callable,
                 fallback: Optional[Callable] = None):
        """注册工具，可选注册降级方案"""
        self.tools[name] = func
        self.breakers[name] = CircuitBreaker()
        self.call_stats[name] = {"total": 0, "success": 0, "failed": 0}
        if fallback:
            self.fallbacks[name] = fallback

    def execute(self, name: str, **kwargs) -> ToolResult:
        """执行工具调用，带熔断、超时、降级"""
        self.call_stats[name]["total"] += 1
        breaker = self.breakers[name]

        # 检查熔断状态
        if not breaker.allow_request():
            # 熔断中：尝试降级方案
            if name in self.fallbacks:
                return self._execute_fallback(name, **kwargs)
            return ToolResult(
                status=ToolResultStatus.FATAL_ERROR,
                data=None,
                summary="",
                error_message=f"工具 {name} 已熔断，暂时不可用",
                suggested_action="请跳过此步骤或使用其他方式获取信息"
            )

        try:
            start = time.time()
            result = self.tools[name](**kwargs)
            latency = time.time() - start

            breaker.record_success()
            self.call_stats[name]["success"] += 1
            logger.info(f"工具 {name} 执行成功，耗时 {latency:.2f}s")

            return ToolResult(
                status=ToolResultStatus.SUCCESS,
                data=result,
                summary=self._generate_summary(name, result),
                context_size=self._estimate_tokens(result)
            )
        except Exception as e:
            breaker.record_failure()
            self.call_stats[name]["failed"] += 1

            # 判断是否可重试
            if self._is_retryable(e):
                return ToolResult(
                    status=ToolResultStatus.RETRYABLE_ERROR,
                    data=None, summary="",
                    error_message=str(e),
                    suggested_action=f"可以在稍后重试调用 {name}"
                )

            # 不可重试：尝试降级
            if name in self.fallbacks:
                return self._execute_fallback(name, **kwargs)

            return ToolResult(
                status=ToolResultStatus.FATAL_ERROR,
                data=None, summary="",
                error_message=str(e),
                suggested_action="请跳过此步骤，使用已有信息继续任务"
            )

    def _execute_fallback(self, name, **kwargs) -> ToolResult:
        """执行降级方案"""
        try:
            result = self.fallbacks[name](**kwargs)
            return ToolResult(
                status=ToolResultStatus.PARTIAL,
                data=result,
                summary=f"[降级模式] {self._generate_summary(name, result)}"
            )
        except Exception:
            return ToolResult(
                status=ToolResultStatus.FATAL_ERROR,
                data=None, summary="",
                error_message=f"工具 {name} 及其降级方案均不可用",
                suggested_action="请跳过此步骤"
            )


# ============ 4. 灰度发布控制 ============

class GrayReleaseRouter:
    """灰度路由：控制新版本工具/模型的流量比例"""

    def __init__(self):
        self.routes = {}  # {"tool_v2": {"weight": 0.01, "metrics": {...}}}

    def add_canary(self, name: str, new_version: Callable, weight: float = 0.01):
        """添加灰度版本，默认 1% 流量"""
        self.routes[name] = {"func": new_version, "weight": weight, "metrics": {
            "calls": 0, "success": 0, "avg_latency": 0
        }}

    def should_use_canary(self, name: str) -> bool:
        """根据权重随机决定是否使用灰度版本"""
        import random
        if name in self.routes:
            return random.random() < self.routes[name]["weight"]
        return False

    def promote(self, name: str, new_weight: float):
        """指标达标后逐步提升灰度比例：1% → 5% → 25% → 100%"""
        if name in self.routes:
            self.routes[name]["weight"] = new_weight
```

部署架构上，小规模与中大规模的差异：

```
# 小规模：单服务器部署
[用户请求] → [Nginx] → [Agent 服务 (LLM + 向量库 + MCP 工具)] → [响应]

# 中大规模：微服务架构
[用户请求] → [API 网关 (限流/鉴权/路由)]
                ├── [Agent 编排服务] ──→ [LLM 服务集群 (主力模型 + 备用模型)]
                ├── [MCP 工具服务集群] ──→ [外部 API / 数据库 / 文件系统]
                ├── [向量数据库集群]
                └── [状态管理 (Redis 集群)]

# 高可用：多区域 + 熔断 + 降级
[Region A] ←→ [Region B]  (互为备份)
每个 Region 内部：
  - 服务熔断：工具不可用时自动隔离
  - 模型降级：GPT-4 不可用 → 切换 Claude → 切换本地模型
  - 灰度发布：新工具版本先 1% 流量验证
```

#### 3. 面试核心回答 / Core Interview Answer

- **AI 只是大脑，工具才是手脚**：LLM 负责推理和决策（30%），但 Agent 要完成实际任务必须依赖工具调用数据库、API、文件系统等（70%），工具的可靠性直接决定了系统的可靠性。
- **工具工程的四大核心挑战**：设计 LLM 可理解的反馈接口（不是给程序员看的错误码，而是给 AI 看的结构化描述）、高效管理上下文（控制工具返回值的大小）、处理部分失败（重试/跳过/降级的策略选择）、构建 AI 适配的错误恢复机制（让 LLM 能自主决策如何处理异常）。
- **架构分级设计**：小规模用单服务器 all-in-one 快速验证，中大规模用微服务架构（LLM 服务、工具集群、向量库、API 网关分离），高可用场景加上多区域部署、服务熔断和模型降级。
- **灰度发布是安全网**：新工具或新模型上线先分配 1% 流量，对比成功率、延迟、用户满意度等关键指标，确认无问题后逐步放量到 5% → 25% → 100%，避免全量发布导致线上事故。
- **状态管理不可忽视**：使用 Redis 集群或本地内存管理会话状态和工具调用上下文，确保 Agent 在多轮交互中不丢失关键信息。

一句话总结：生产级 Agent 系统的瓶颈不在 AI 的聪明程度，而在工具工程的可靠程度——熔断、降级、灰度、错误恢复这些"脏活累活"才是 Agent 能否上线的关键。
