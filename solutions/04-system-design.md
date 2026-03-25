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

---

### Q: 在大规模 Agent 部署中，如何设计分级模型路由策略（简单任务路由小模型、复杂推理路由大模型）以优化 token 成本？每次执行成本 $0.15 在日均 50 万请求下如何控制？

> 🤖 **Claude**

#### 1. 网络整合回答

在大规模 Agent 部署中，分级模型路由（Tiered Model Routing）是目前业界公认的最高 ROI 成本优化手段，据 2025-2026 年的生产数据统计，合理的路由策略可将 token 成本降低 60-85%。其核心思想是：不是所有请求都需要最强大的模型——简单的意图识别、模板化回复、信息提取等任务用小模型（如 GPT-4o mini、Gemini 2.0 Flash、Llama 4 Scout）即可胜任，而复杂的多步推理、代码生成、创意写作等任务才需要路由到大模型（如 GPT-4、Claude Opus）。

**分级路由架构**的关键组件包括：

1. **意图分类器（Intent Classifier）**：作为路由系统的"大脑"，负责在请求进入时快速判断其复杂度。通常使用轻量级模型（如 BERT 微调版或小型 LLM）作为分类器，将请求分为 3-4 个复杂度等级。ICLR 2025 发表的 RouteLLM 研究表明，经过训练的路由器可以在保持 GPT-4 95% 性能的前提下实现 85% 的成本降低。分类维度包括：任务类型（问答/推理/生成/提取）、上下文复杂度（单轮/多轮/长上下文）、领域专业度（通用/垂直领域）和精度要求（容错/精确）。

2. **模型选择策略（Model Selection Policy）**：基于分类结果将请求路由到对应的模型层级。当前主流的价格梯度为：轻量模型 $0.10-0.50/MTok（Gemini 2.0 Flash、Llama 系列）、中端模型 $10-15/MTok（GPT-4 Turbo、Claude Sonnet）、重量级模型 $30-60/MTok（GPT-4、Claude Opus）。层级之间的成本差距可达 25-100 倍，这意味着如果 70% 的请求可以被小模型处理，整体成本将大幅下降。

3. **成本计算模型**：日均 50 万请求 × $0.15/次 = $75,000/天 = $225 万/月，这是未优化的基线成本。通过分级路由优化后的成本估算：假设 70% 简单请求路由到小模型（成本降至 $0.02/次）、20% 中等请求路由到中端模型（成本 $0.08/次）、10% 复杂请求路由到大模型（保持 $0.15/次），则优化后的日均成本 = 350,000 × $0.02 + 100,000 × $0.08 + 50,000 × $0.15 = $7,000 + $8,000 + $7,500 = $22,500/天，相比原始 $75,000/天节省约 70%。

4. **语义缓存（Semantic Caching）**：对语义相似的查询直接复用历史结果，完全跳过 LLM 推理。Redis LangCache 在高重复场景下实现了约 73% 的成本削减，缓存命中时响应时间从数秒降至毫秒级。在日均 50 万请求中，如果 20-30% 的请求可被缓存命中，又能进一步节省 15-20% 的总成本。

5. **Prompt 压缩（Prompt Compression）**：使用 LLMLingua-2 等工具对输入 prompt 进行压缩，可实现 2-5 倍的 token 压缩比，在仅损失 1.5% 性能的情况下将 token 消耗降低 50-80%。尤其对 RAG 系统中的长检索上下文效果显著。生产环境建议从保守的 2-3 倍压缩开始，先在 5% 流量上验证质量，再逐步提升压缩比。

6. **前缀缓存（Prefix Caching）**：对于大量重复的 system prompt 和静态知识库内容，利用 LLM 原生的 prefix caching 机制可获得约 50% 的 token 折扣。

综合运用以上策略，生产级 Agent 系统的月度 LLM API 成本可从未优化的 $225 万控制到 $30-50 万范围内，实现 75-85% 的成本降低。

#### 2. 结合实际例子

以下是一个完整的分级模型路由系统实现，包含意图分类、模型路由、语义缓存和成本追踪：

```python
import hashlib
import json
import time
from dataclasses import dataclass, field
from enum import Enum
from typing import Optional

import numpy as np


# ============================================================
# 1. 数据结构定义
# ============================================================
class ComplexityLevel(Enum):
    """请求复杂度等级"""
    SIMPLE = "simple"        # 简单查询、信息提取、模板化回复
    MODERATE = "moderate"    # 中等推理、摘要生成、单步工具调用
    COMPLEX = "complex"      # 多步推理、代码生成、复杂分析
    EXPERT = "expert"        # 专家级推理、创意写作、多 Agent 协作


@dataclass
class ModelTier:
    """模型层级配置"""
    name: str
    model_id: str
    cost_per_1k_tokens: float  # 每 1K token 的成本（美元）
    max_tokens: int
    avg_latency_ms: int


@dataclass
class RoutingResult:
    """路由决策结果"""
    model_tier: ModelTier
    complexity: ComplexityLevel
    confidence: float
    cache_hit: bool = False
    compressed: bool = False
    estimated_cost: float = 0.0


@dataclass
class CostTracker:
    """成本追踪器"""
    total_requests: int = 0
    total_cost: float = 0.0
    cache_hits: int = 0
    tier_distribution: dict = field(default_factory=lambda: {
        "simple": 0, "moderate": 0, "complex": 0, "expert": 0
    })
    cost_by_tier: dict = field(default_factory=lambda: {
        "simple": 0.0, "moderate": 0.0, "complex": 0.0, "expert": 0.0
    })


# ============================================================
# 2. 模型层级定义（基于 2025-2026 市场价格）
# ============================================================
MODEL_TIERS = {
    ComplexityLevel.SIMPLE: ModelTier(
        name="lightweight",
        model_id="gemini-2.0-flash",       # 或 gpt-4o-mini / llama-4-scout
        cost_per_1k_tokens=0.0001,          # $0.10 / MTok
        max_tokens=2048,
        avg_latency_ms=200
    ),
    ComplexityLevel.MODERATE: ModelTier(
        name="mid-tier",
        model_id="claude-sonnet-4",         # 或 gpt-4-turbo
        cost_per_1k_tokens=0.01,            # $10 / MTok
        max_tokens=4096,
        avg_latency_ms=800
    ),
    ComplexityLevel.COMPLEX: ModelTier(
        name="premium",
        model_id="claude-opus-4",           # 或 gpt-4
        cost_per_1k_tokens=0.03,            # $30 / MTok
        max_tokens=8192,
        avg_latency_ms=2000
    ),
    ComplexityLevel.EXPERT: ModelTier(
        name="expert-reasoning",
        model_id="o3",                      # 或 claude-opus-4 + extended thinking
        cost_per_1k_tokens=0.06,            # $60 / MTok
        max_tokens=16384,
        avg_latency_ms=5000
    ),
}


# ============================================================
# 3. 意图分类器（Intent Classifier）
# ============================================================
class IntentClassifier:
    """
    轻量级意图分类器，用于判断请求复杂度。
    生产环境中通常使用微调的 BERT/DistilBERT 模型，
    此处演示基于规则 + 特征的混合分类方案。
    """

    # 复杂度信号关键词
    SIMPLE_SIGNALS = {"你好", "是什么", "定义", "帮我查", "天气", "时间", "翻译"}
    COMPLEX_SIGNALS = {"分析", "推理", "比较", "为什么", "设计", "优化", "策略"}
    EXPERT_SIGNALS = {"多步推理", "代码重构", "架构设计", "数学证明", "论文撰写"}

    def classify(self, query: str, context: Optional[dict] = None) -> tuple[ComplexityLevel, float]:
        """
        对请求进行复杂度分类。
        返回 (复杂度等级, 置信度)。
        """
        features = self._extract_features(query, context)
        score = self._compute_complexity_score(features)

        if score < 0.25:
            return ComplexityLevel.SIMPLE, min(0.95, 1.0 - score)
        elif score < 0.50:
            return ComplexityLevel.MODERATE, 0.7 + (0.50 - score)
        elif score < 0.75:
            return ComplexityLevel.COMPLEX, 0.6 + (0.75 - score)
        else:
            return ComplexityLevel.EXPERT, 0.5 + (score - 0.75)

    def _extract_features(self, query: str, context: Optional[dict]) -> dict:
        """提取分类特征"""
        return {
            "query_length": len(query),
            "has_simple_signals": any(s in query for s in self.SIMPLE_SIGNALS),
            "has_complex_signals": any(s in query for s in self.COMPLEX_SIGNALS),
            "has_expert_signals": any(s in query for s in self.EXPERT_SIGNALS),
            "conversation_turns": context.get("turns", 0) if context else 0,
            "requires_tools": context.get("requires_tools", False) if context else False,
            "domain_specific": context.get("domain_specific", False) if context else False,
        }

    def _compute_complexity_score(self, features: dict) -> float:
        """基于特征计算复杂度分数 (0.0 ~ 1.0)"""
        score = 0.0

        # 查询长度：越长通常越复杂
        if features["query_length"] > 500:
            score += 0.3
        elif features["query_length"] > 200:
            score += 0.15
        elif features["query_length"] > 50:
            score += 0.05

        # 关键词信号
        if features["has_expert_signals"]:
            score += 0.4
        elif features["has_complex_signals"]:
            score += 0.25
        elif features["has_simple_signals"]:
            score -= 0.1

        # 上下文因素
        if features["conversation_turns"] > 5:
            score += 0.15
        if features["requires_tools"]:
            score += 0.1
        if features["domain_specific"]:
            score += 0.1

        return max(0.0, min(1.0, score))


# ============================================================
# 4. 语义缓存（Semantic Cache）
# ============================================================
class SemanticCache:
    """
    语义缓存：基于 embedding 相似度匹配历史查询。
    生产环境建议使用 Redis + 向量索引（如 Redis LangCache）。
    """

    def __init__(self, similarity_threshold: float = 0.92):
        self.cache: dict[str, dict] = {}  # hash -> {query, response, embedding, timestamp}
        self.similarity_threshold = similarity_threshold

    def get(self, query: str, query_embedding: Optional[list] = None) -> Optional[str]:
        """查询缓存，返回匹配的历史响应"""
        # 1. 精确匹配（哈希）
        query_hash = hashlib.md5(query.strip().lower().encode()).hexdigest()
        if query_hash in self.cache:
            entry = self.cache[query_hash]
            if time.time() - entry["timestamp"] < 3600:  # TTL: 1 小时
                return entry["response"]

        # 2. 语义匹配（embedding 余弦相似度）
        if query_embedding is not None:
            for entry in self.cache.values():
                if "embedding" in entry:
                    similarity = self._cosine_similarity(query_embedding, entry["embedding"])
                    if similarity >= self.similarity_threshold:
                        return entry["response"]
        return None

    def put(self, query: str, response: str, embedding: Optional[list] = None):
        """写入缓存"""
        query_hash = hashlib.md5(query.strip().lower().encode()).hexdigest()
        self.cache[query_hash] = {
            "query": query,
            "response": response,
            "embedding": embedding,
            "timestamp": time.time(),
        }

    @staticmethod
    def _cosine_similarity(a: list, b: list) -> float:
        a, b = np.array(a), np.array(b)
        return float(np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b) + 1e-8))


# ============================================================
# 5. 模型路由器（Model Router）—— 核心调度引擎
# ============================================================
class TieredModelRouter:
    """
    分级模型路由器：整合意图分类、语义缓存、模型选择和成本追踪。

    使用方式:
        router = TieredModelRouter()
        result = router.route("帮我分析这段代码的性能瓶颈并给出优化方案")
        print(f"路由到: {result.model_tier.model_id}, 预估成本: ${result.estimated_cost:.4f}")
    """

    def __init__(self):
        self.classifier = IntentClassifier()
        self.cache = SemanticCache(similarity_threshold=0.92)
        self.cost_tracker = CostTracker()

    def route(self, query: str, context: Optional[dict] = None) -> RoutingResult:
        """对请求进行路由决策"""
        # Step 1: 检查语义缓存
        cached_response = self.cache.get(query)
        if cached_response is not None:
            self.cost_tracker.cache_hits += 1
            self.cost_tracker.total_requests += 1
            return RoutingResult(
                model_tier=MODEL_TIERS[ComplexityLevel.SIMPLE],
                complexity=ComplexityLevel.SIMPLE,
                confidence=1.0,
                cache_hit=True,
                estimated_cost=0.0,  # 缓存命中零成本
            )

        # Step 2: 意图分类
        complexity, confidence = self.classifier.classify(query, context)

        # Step 3: 置信度不足时升级（宁可多花钱也不降低质量）
        if confidence < 0.6 and complexity != ComplexityLevel.EXPERT:
            complexity = ComplexityLevel(
                list(ComplexityLevel)[min(
                    list(ComplexityLevel).index(complexity) + 1,
                    len(ComplexityLevel) - 1
                )].value
            )

        # Step 4: 选择模型层级
        model_tier = MODEL_TIERS[complexity]

        # Step 5: 估算成本（基于平均 token 消耗）
        avg_tokens = {
            ComplexityLevel.SIMPLE: 500,
            ComplexityLevel.MODERATE: 1500,
            ComplexityLevel.COMPLEX: 3000,
            ComplexityLevel.EXPERT: 6000,
        }
        estimated_cost = (avg_tokens[complexity] / 1000) * model_tier.cost_per_1k_tokens

        # Step 6: 更新成本追踪
        self.cost_tracker.total_requests += 1
        self.cost_tracker.total_cost += estimated_cost
        self.cost_tracker.tier_distribution[complexity.value] += 1
        self.cost_tracker.cost_by_tier[complexity.value] += estimated_cost

        return RoutingResult(
            model_tier=model_tier,
            complexity=complexity,
            confidence=confidence,
            estimated_cost=estimated_cost,
        )

    def get_daily_cost_report(self, daily_requests: int = 500_000) -> str:
        """生成日成本预估报告"""
        if self.cost_tracker.total_requests == 0:
            return "暂无数据"

        avg_cost = self.cost_tracker.total_cost / self.cost_tracker.total_requests
        projected_daily = avg_cost * daily_requests
        baseline_daily = 0.15 * daily_requests  # 未优化基线

        report = f"""
╔══════════════════════════════════════════════════════╗
║           日成本预估报告 (Daily Cost Report)           ║
╠══════════════════════════════════════════════════════╣
║ 样本请求数:  {self.cost_tracker.total_requests:>10,}                         ║
║ 缓存命中率:  {self.cost_tracker.cache_hits/self.cost_tracker.total_requests*100:>9.1f}%                         ║
║ 平均单次成本: ${avg_cost:>9.4f}                          ║
╠══════════════════════════════════════════════════════╣
║ 层级分布:                                              ║
║   Simple:   {self.cost_tracker.tier_distribution["simple"]:>6} 次  ${self.cost_tracker.cost_by_tier["simple"]:>10.2f}  ║
║   Moderate: {self.cost_tracker.tier_distribution["moderate"]:>6} 次  ${self.cost_tracker.cost_by_tier["moderate"]:>10.2f}  ║
║   Complex:  {self.cost_tracker.tier_distribution["complex"]:>6} 次  ${self.cost_tracker.cost_by_tier["complex"]:>10.2f}  ║
║   Expert:   {self.cost_tracker.tier_distribution["expert"]:>6} 次  ${self.cost_tracker.cost_by_tier["expert"]:>10.2f}  ║
╠══════════════════════════════════════════════════════╣
║ 日均 {daily_requests:,} 请求预估:                        ║
║   未优化基线:  ${baseline_daily:>12,.0f} / 天              ║
║   优化后预估:  ${projected_daily:>12,.0f} / 天              ║
║   节省比例:    {(1 - projected_daily/baseline_daily)*100:>10.1f}%                       ║
║   月度节省:    ${(baseline_daily - projected_daily)*30:>12,.0f}                  ║
╚══════════════════════════════════════════════════════╝
"""
        return report


# ============================================================
# 6. 使用示例
# ============================================================
if __name__ == "__main__":
    router = TieredModelRouter()

    # 模拟不同复杂度的请求
    test_queries = [
        ("你好，今天天气怎么样？", None),
        ("帮我翻译这句话：Hello World", None),
        ("分析一下 React 和 Vue 的优缺点比较", None),
        ("请设计一个高并发分布式消息队列的架构方案", {"domain_specific": True}),
        ("帮我进行多步数学证明：费马大定理的简化说明", {"requires_tools": True}),
        ("你好，今天天气怎么样？", None),  # 重复请求，测试缓存
    ]

    for query, ctx in test_queries:
        # 第一次请求后将结果存入缓存
        result = router.route(query, ctx)
        if not result.cache_hit:
            router.cache.put(query, f"[模拟回复] {query[:20]}...")

        print(f"Query: {query[:30]:30s} → "
              f"Model: {result.model_tier.model_id:20s} | "
              f"Complexity: {result.complexity.value:10s} | "
              f"Cache: {'✓' if result.cache_hit else '✗'} | "
              f"Cost: ${result.estimated_cost:.4f}")

    print(router.get_daily_cost_report())
```

#### 3. 面试核心回答

- **分级路由是最高 ROI 的成本优化手段**：通过意图分类器将请求按复杂度分为 3-4 个层级，简单请求（约 70%）路由到轻量模型（Gemini Flash / GPT-4o mini，成本 $0.10/MTok），复杂请求（约 10%）才路由到大模型（Claude Opus / GPT-4，成本 $30-60/MTok），层级间成本差距可达 100 倍，综合节省 60-85%。
- **成本控制的数学模型**：未优化场景 50 万请求 × $0.15 = $75,000/天（$225 万/月）；分级路由后 70% 简单 × $0.02 + 20% 中等 × $0.08 + 10% 复杂 × $0.15 = $22,500/天，节省 70%；再叠加语义缓存（命中率 20-30%）和 prompt 压缩（2-3x），总成本可控制在 $15,000/天以内（$45 万/月），相比基线节省 80%。
- **语义缓存 + Prompt 压缩是乘数效应**：语义缓存对相似查询直接返回历史结果（Redis LangCache 实现 73% 命中率下的成本削减），LLMLingua-2 实现 2-5 倍 prompt 压缩且性能损失仅 1.5%，两者与路由策略叠加可形成多层成本优化。
- **置信度不足时向上升级（Escalation）**：分类器置信度低于阈值时，宁可路由到更贵的模型也不降低回答质量，这是生产环境中"质量优先、成本次之"的核心原则。RouteLLM（ICLR 2025）证明经过训练的路由器可在保持 GPT-4 95% 性能的同时实现 85% 成本降低。
- **监控与灰度是安全网**：实时监控各层级的成本分布、缓存命中率和质量指标（准确率、用户满意度），新的路由策略先在 1-5% 流量上灰度验证，确保成本优化不以质量为代价。

一句话总结：分级模型路由的本质是"用对的模型做对的事"——通过意图分类器 + 语义缓存 + prompt 压缩构建三层成本防线，将日均 50 万请求的 $75,000/天基线成本压缩至 $15,000/天以内，实现 80% 的成本降低而几乎不损失质量。

---

### Q: 如何设计一个 AI 驱动的投资顾问 Agent 系统？需要考虑哪些关键架构决策和风控机制？

> 🤖 **Claude**

#### 1. 网络整合回答

设计一个 AI 驱动的投资顾问 Agent 系统，核心挑战在于将 LLM 的推理能力与金融领域的严格合规要求、实时数据处理和风险控制深度融合。从架构维度看，系统应采用**四层分层架构**：

**数据层（Data Perception Layer）**：负责接入多源异构数据，包括实时行情数据（股票/基金/债券/期货的 tick-level 数据流）、宏观经济指标（CPI、PMI、利率决议等）、财务报表结构化数据、新闻舆情和社交媒体非结构化数据、以及另类数据（卫星图像、供应链数据等）。数据层需要构建统一的 Data Pipeline，通过 Apache Kafka 或 Flink 实现流批一体的数据接入，并维护一个金融知识图谱（Knowledge Graph）来关联公司关系、行业链条和宏观传导路径。数据质量校验（Data Validation）是这一层的关键——脏数据直接导致错误的投资建议，后果比普通 AI 应用严重得多。

**分析层（Analysis & Reasoning Layer）**：这是 LLM Agent 的核心推理引擎。Agent 在此层执行多维度分析，包括基本面分析（财务指标解读、估值模型计算）、技术面分析（趋势识别、形态判断）、情绪面分析（新闻舆情 NLP 打分）和宏观面分析（政策影响评估）。关键的架构决策是选择 **Single-Agent 还是 Multi-Agent**：根据 2025-2026 年的实践经验，建议采用**主 Agent + 专家子 Agent** 的混合架构——一个 Orchestrator Agent 负责全局协调，下辖多个专业子 Agent（如 Macro Analyst Agent、Equity Analyst Agent、Risk Analyst Agent），各子 Agent 拥有独立的工具集和 Prompt 策略。这种架构比纯 Single-Agent 更易扩展，比完全去中心化的 Multi-Agent 更易调试和监控。

**决策层（Strategy & Decision Layer）**：将分析结果转化为具体的投资建议。此层需实现**用户画像系统（Investor Profiling）**，通过 KYC（Know Your Customer）问卷和历史行为数据，建立用户的风险承受能力（Risk Tolerance）、投资期限（Time Horizon）、资产规模和投资目标的多维画像。投资建议必须与用户画像严格匹配——这是合规的硬性要求。决策层还需要实现 **Portfolio Optimization**，基于 Modern Portfolio Theory（MPT）或 Black-Litterman 模型生成资产配置方案，并通过 Monte Carlo 模拟评估不同市场情景下的预期收益和最大回撤。

**执行层（Execution & Control Layer）**：负责建议的最终下达和执行监控。这一层必须实现**强制性的 Human-in-the-Loop（HITL）机制**——在金融领域，完全自主执行的 AI Agent 面临严重的监管和法律风险。根据 2026 年最新的监管趋势（参考 NIST AI Risk Management Framework 和各国金融监管要求），AI 投资顾问必须在关键决策点获得人类审批。执行层还需要实现完整的审计日志（Audit Trail），记录每一条建议的生成依据、推理链路和用户确认状态，确保满足监管的可解释性要求。

**风控机制**是整个系统的生命线，需要贯穿所有层级：（1）**Pre-Trade 风控**：在建议生成前检查合规规则，如单一标的持仓上限、行业集中度限制、杠杆率约束等；（2）**Real-Time 风控**：实时监控组合风险指标（VaR、CVaR、最大回撤），触发阈值时自动降级或暂停建议功能；（3）**Model Risk 风控**：对 LLM 输出进行 Hallucination Detection，使用 Guardrails 框架过滤不合规、不合理的投资建议，防止模型幻觉导致的错误推荐；（4）**对抗性风控**：防御 Prompt Injection、数据投毒（Data Poisoning）和 Deepfake 新闻对分析引擎的干扰，这在 2026 年已成为金融 AI 系统的头号安全威胁。此外，系统需实现 **Circuit Breaker** 机制——当市场出现极端波动（如日内跌幅超过阈值）时，自动切换到保守模式或直接暂停 AI 建议，交由人类顾问接管。

#### 2. 结合实际例子

以下是一个投资顾问 Agent 系统的核心模块设计，展示 Orchestrator 如何协调多个专家 Agent 完成一次投资咨询：

```python
"""
AI 投资顾问 Agent 系统 - 核心架构设计
四层架构：数据层 → 分析层 → 决策层 → 执行层
"""
from dataclasses import dataclass, field
from enum import Enum
from typing import Optional
import asyncio
from datetime import datetime


# ==================== 用户画像模块 ====================
class RiskTolerance(Enum):
    CONSERVATIVE = "conservative"      # 保守型：年化波动 < 5%
    MODERATE = "moderate"              # 稳健型：年化波动 5-15%
    AGGRESSIVE = "aggressive"          # 进取型：年化波动 15-25%
    SPECULATIVE = "speculative"        # 激进型：年化波动 > 25%


@dataclass
class InvestorProfile:
    """用户画像：KYC 合规要求的核心数据结构"""
    user_id: str
    risk_tolerance: RiskTolerance
    investment_horizon_months: int          # 投资期限（月）
    total_assets: float                     # 可投资资产总额
    annual_income: float                    # 年收入（用于适当性评估）
    investment_objectives: list[str]        # 如 ["capital_preservation", "steady_income"]
    restricted_sectors: list[str] = field(default_factory=list)  # 限制行业（ESG 偏好等）
    max_single_position_pct: float = 0.10   # 单一标的最大仓位占比
    max_sector_concentration_pct: float = 0.30  # 单一行业最大集中度


# ==================== 风控引擎 ====================
@dataclass
class RiskCheckResult:
    passed: bool
    violations: list[str] = field(default_factory=list)
    risk_score: float = 0.0  # 0-100，越高风险越大


class RiskControlEngine:
    """三级风控引擎：Pre-Trade / Real-Time / Model-Risk"""

    def __init__(self, circuit_breaker_threshold: float = 0.07):
        self.circuit_breaker_threshold = circuit_breaker_threshold  # 7% 日跌幅熔断
        self.is_circuit_breaker_active = False
        self.daily_recommendation_count = 0
        self.max_daily_recommendations = 50  # 单用户每日建议上限

    def pre_trade_check(
        self, recommendation: "InvestmentRecommendation", profile: InvestorProfile
    ) -> RiskCheckResult:
        """Pre-Trade 风控：在建议下达前检查合规约束"""
        violations = []

        # 1. 单一标的仓位检查
        if recommendation.position_pct > profile.max_single_position_pct:
            violations.append(
                f"单一标的仓位 {recommendation.position_pct:.1%} "
                f"超过限额 {profile.max_single_position_pct:.1%}"
            )

        # 2. 风险等级匹配检查
        risk_mapping = {
            RiskTolerance.CONSERVATIVE: 30,
            RiskTolerance.MODERATE: 60,
            RiskTolerance.AGGRESSIVE: 80,
            RiskTolerance.SPECULATIVE: 100,
        }
        max_allowed_risk = risk_mapping[profile.risk_tolerance]
        if recommendation.risk_score > max_allowed_risk:
            violations.append(
                f"建议风险评分 {recommendation.risk_score} "
                f"超过用户风险承受上限 {max_allowed_risk}"
            )

        # 3. 限制行业检查
        if recommendation.sector in profile.restricted_sectors:
            violations.append(f"标的所属行业 {recommendation.sector} 在用户限制列表中")

        # 4. 每日建议频率限制
        if self.daily_recommendation_count >= self.max_daily_recommendations:
            violations.append("已达单用户每日建议上限，防止过度交易")

        return RiskCheckResult(
            passed=len(violations) == 0,
            violations=violations,
            risk_score=recommendation.risk_score,
        )

    def check_market_circuit_breaker(self, market_daily_change_pct: float) -> bool:
        """市场熔断检查：极端行情下暂停 AI 建议"""
        if abs(market_daily_change_pct) >= self.circuit_breaker_threshold:
            self.is_circuit_breaker_active = True
            return True  # 触发熔断
        return False

    def validate_llm_output(self, raw_output: str) -> tuple[bool, list[str]]:
        """Model-Risk 风控：检测 LLM 输出的幻觉和合规问题"""
        issues = []

        # Guardrail 1：检测绝对化收益承诺（合规红线）
        forbidden_patterns = [
            "保证收益", "稳赚不赔", "guaranteed return",
            "零风险", "100% 盈利", "一定会涨",
        ]
        for pattern in forbidden_patterns:
            if pattern in raw_output.lower():
                issues.append(f"检测到违规表述: '{pattern}'（违反投资顾问合规要求）")

        # Guardrail 2：检测缺少风险提示
        if "风险" not in raw_output and "risk" not in raw_output.lower():
            issues.append("建议内容缺少风险提示语句（合规要求每条建议必须附带风险提示）")

        # Guardrail 3：检测虚构的股票代码或基金名称（Hallucination Detection）
        # 实际生产中需对接证券主数据库进行校验
        return len(issues) == 0, issues


# ==================== 专家子 Agent ====================
@dataclass
class AnalysisResult:
    agent_name: str
    summary: str
    confidence: float          # 0-1
    signals: dict              # 具体的分析信号
    timestamp: datetime = field(default_factory=datetime.now)


class BaseAnalystAgent:
    """分析型子 Agent 的基类"""

    def __init__(self, name: str, llm_client, tools: list):
        self.name = name
        self.llm_client = llm_client
        self.tools = tools  # 如 [MarketDataTool, FinancialStatementTool, ...]

    async def analyze(self, query: str, context: dict) -> AnalysisResult:
        raise NotImplementedError


class MacroAnalystAgent(BaseAnalystAgent):
    """宏观分析 Agent：解读宏观经济数据和政策影响"""

    def __init__(self, llm_client):
        super().__init__(
            name="MacroAnalyst",
            llm_client=llm_client,
            tools=["EconomicIndicatorTool", "CentralBankPolicyTool", "GeopoliticalRiskTool"],
        )

    async def analyze(self, query: str, context: dict) -> AnalysisResult:
        # 1. 调用工具获取最新宏观数据
        macro_data = await self._fetch_macro_indicators()
        # 2. LLM 推理：评估宏观环境对投资标的的影响
        prompt = f"""你是专业的宏观经济分析师。根据以下宏观数据，分析当前宏观环境
对 "{query}" 相关投资标的的影响方向和力度。

宏观数据：{macro_data}
用户问题：{query}

请输出：1. 宏观环境评估（利好/中性/利空）2. 关键驱动因素 3. 置信度(0-1)"""

        response = await self.llm_client.chat(prompt)
        return AnalysisResult(
            agent_name=self.name,
            summary=response.content,
            confidence=response.parsed_confidence,
            signals={"macro_stance": response.parsed_stance, "key_factors": response.parsed_factors},
        )

    async def _fetch_macro_indicators(self) -> dict:
        """获取宏观经济指标（模拟）"""
        return {
            "gdp_growth": 4.8, "cpi_yoy": 2.1, "pmi": 51.3,
            "interest_rate": 3.45, "policy_stance": "neutral",
        }


class EquityAnalystAgent(BaseAnalystAgent):
    """权益分析 Agent：个股/基金的基本面和技术面分析"""

    def __init__(self, llm_client):
        super().__init__(
            name="EquityAnalyst",
            llm_client=llm_client,
            tools=["FinancialStatementTool", "TechnicalIndicatorTool", "ValuationModelTool"],
        )

    async def analyze(self, query: str, context: dict) -> AnalysisResult:
        # 并行执行基本面分析和技术面分析
        fundamental, technical = await asyncio.gather(
            self._fundamental_analysis(context.get("ticker")),
            self._technical_analysis(context.get("ticker")),
        )
        combined_prompt = f"""你是专业的权益分析师。综合以下基本面和技术面数据给出分析。
基本面：{fundamental}
技术面：{technical}
用户问题：{query}"""

        response = await self.llm_client.chat(combined_prompt)
        return AnalysisResult(
            agent_name=self.name,
            summary=response.content,
            confidence=response.parsed_confidence,
            signals={"fundamental_score": fundamental.get("score"), "technical_signal": technical.get("signal")},
        )

    async def _fundamental_analysis(self, ticker: str) -> dict:
        return {"pe_ratio": 15.2, "roe": 0.18, "debt_ratio": 0.35, "score": 72}

    async def _technical_analysis(self, ticker: str) -> dict:
        return {"trend": "bullish", "rsi": 55, "macd_signal": "buy", "signal": "positive"}


# ==================== 投资建议数据结构 ====================
@dataclass
class InvestmentRecommendation:
    """投资建议：风控引擎的检查对象"""
    ticker: str
    action: str                 # "buy" / "sell" / "hold"
    position_pct: float         # 建议仓位占比
    risk_score: float           # 0-100
    sector: str
    rationale: str              # LLM 生成的推理依据
    risk_disclaimer: str        # 风险提示（合规必须）
    confidence: float           # 综合置信度
    analyst_reports: list[AnalysisResult] = field(default_factory=list)


# ==================== 主 Orchestrator Agent ====================
class InvestmentAdvisorOrchestrator:
    """
    投资顾问 Orchestrator Agent
    职责：协调多个专家子 Agent，汇总分析，生成合规的投资建议
    架构：主 Agent + 专家子 Agent 的混合模式
    """

    def __init__(self, llm_client, risk_engine: RiskControlEngine):
        self.llm_client = llm_client
        self.risk_engine = risk_engine
        self.macro_agent = MacroAnalystAgent(llm_client)
        self.equity_agent = EquityAnalystAgent(llm_client)
        # 可扩展更多子 Agent：FixedIncomeAgent, AlternativeAgent, SentimentAgent ...

    async def handle_consultation(
        self, user_query: str, profile: InvestorProfile
    ) -> dict:
        """
        处理一次完整的投资咨询请求
        流程：熔断检查 → 并行分析 → 综合推理 → 风控校验 → HITL 审批 → 输出
        """
        # Step 0: 市场熔断检查
        if self.risk_engine.is_circuit_breaker_active:
            return {
                "status": "circuit_breaker_active",
                "message": "当前市场波动剧烈，AI 建议功能已暂停，请联系人工顾问。",
            }

        # Step 1: 并行调度多个专家子 Agent 进行分析
        context = {"user_query": user_query, "ticker": self._extract_ticker(user_query)}
        macro_result, equity_result = await asyncio.gather(
            self.macro_agent.analyze(user_query, context),
            self.equity_agent.analyze(user_query, context),
        )

        # Step 2: Orchestrator 综合推理，生成投资建议
        synthesis_prompt = f"""你是资深投资顾问。请根据以下多维分析结果，为用户生成个性化投资建议。

用户画像：
- 风险偏好：{profile.risk_tolerance.value}
- 投资期限：{profile.investment_horizon_months} 个月
- 可投资资产：{profile.total_assets:,.0f} 元
- 投资目标：{', '.join(profile.investment_objectives)}

宏观分析（{macro_result.agent_name}，置信度 {macro_result.confidence:.0%}）：
{macro_result.summary}

权益分析（{equity_result.agent_name}，置信度 {equity_result.confidence:.0%}）：
{equity_result.summary}

要求：
1. 建议必须匹配用户风险偏好
2. 必须包含具体的资产配置建议和仓位比例
3. 必须附带风险提示
4. 禁止出现"保证收益""稳赚不赔"等违规表述"""

        recommendation_response = await self.llm_client.chat(synthesis_prompt)

        # Step 3: Model-Risk 风控 —— 检测 LLM 输出合规性
        is_compliant, issues = self.risk_engine.validate_llm_output(
            recommendation_response.content
        )
        if not is_compliant:
            # 输出不合规，触发重新生成或人工审核
            return {
                "status": "compliance_review_required",
                "issues": issues,
                "message": "AI 建议未通过合规检查，已转交合规团队审核。",
            }

        # Step 4: 构建推荐对象并执行 Pre-Trade 风控
        recommendation = InvestmentRecommendation(
            ticker=context.get("ticker", "PORTFOLIO"),
            action="buy",
            position_pct=0.08,
            risk_score=55,
            sector="technology",
            rationale=recommendation_response.content,
            risk_disclaimer="投资有风险，入市需谨慎。过往业绩不预示未来表现。",
            confidence=(macro_result.confidence + equity_result.confidence) / 2,
            analyst_reports=[macro_result, equity_result],
        )

        risk_check = self.risk_engine.pre_trade_check(recommendation, profile)
        if not risk_check.passed:
            return {
                "status": "risk_check_failed",
                "violations": risk_check.violations,
                "message": "建议未通过风控检查，已自动调整或拦截。",
            }

        # Step 5: Human-in-the-Loop —— 标记为待人工确认
        return {
            "status": "pending_human_approval",
            "recommendation": {
                "action": recommendation.action,
                "rationale": recommendation.rationale,
                "risk_disclaimer": recommendation.risk_disclaimer,
                "confidence": f"{recommendation.confidence:.0%}",
                "risk_score": recommendation.risk_score,
                "analyst_reports": [
                    {"agent": r.agent_name, "confidence": f"{r.confidence:.0%}"}
                    for r in recommendation.analyst_reports
                ],
            },
            "audit_trail": {
                "timestamp": datetime.now().isoformat(),
                "user_id": profile.user_id,
                "query": user_query,
                "risk_check": "passed",
                "compliance_check": "passed",
                "awaiting": "human_advisor_approval",  # 关键：必须经人工确认
            },
        }

    def _extract_ticker(self, query: str) -> str:
        """从自然语言中提取股票/基金代码（简化示意）"""
        return "600519"  # 实际应使用 NER 或正则匹配


# ==================== 使用示例 ====================
async def main():
    """模拟一次完整的投资咨询流程"""
    # 初始化系统
    llm_client = None  # 实际替换为 ChatOpenAI / Claude 等
    risk_engine = RiskControlEngine(circuit_breaker_threshold=0.07)
    orchestrator = InvestmentAdvisorOrchestrator(llm_client, risk_engine)

    # 创建用户画像
    user_profile = InvestorProfile(
        user_id="USR_20260325_001",
        risk_tolerance=RiskTolerance.MODERATE,
        investment_horizon_months=24,
        total_assets=500_000.0,
        annual_income=300_000.0,
        investment_objectives=["steady_income", "moderate_growth"],
        restricted_sectors=["gambling", "tobacco"],
    )

    # 发起咨询
    result = await orchestrator.handle_consultation(
        user_query="我想了解一下贵州茅台当前是否值得配置？大概配多少仓位合适？",
        profile=user_profile,
    )

    print("=== 投资顾问 Agent 响应 ===")
    for key, value in result.items():
        print(f"  {key}: {value}")

# asyncio.run(main())
```

#### 3. 面试核心回答

- **四层分层架构是系统骨架**：数据层（多源实时数据接入 + 金融知识图谱）→ 分析层（Multi-Agent 协作：宏观/权益/情绪等专家 Agent 并行分析）→ 决策层（用户画像匹配 + Portfolio Optimization）→ 执行层（Human-in-the-Loop 强制审批 + 审计日志），每一层都有独立的风控检查点。
- **Multi-Agent 架构选择 Orchestrator 模式**：采用主 Agent 协调 + 专家子 Agent 执行的混合架构，而非完全自治的去中心化模式。原因是金融场景对可控性和可审计性要求极高，Orchestrator 模式便于全局风控、统一审计和故障定位，同时通过并行调度子 Agent 保持分析效率。
- **三级风控贯穿全链路**：Pre-Trade 风控（仓位限额/行业集中度/适当性匹配）+ Real-Time 风控（VaR 监控/市场熔断 Circuit Breaker）+ Model-Risk 风控（LLM 幻觉检测/合规 Guardrails/禁止绝对化收益承诺），任何一级不通过都拦截建议。2026 年的新威胁还包括对抗性攻击（Prompt Injection、Deepfake 新闻注入），需要额外的 AI 安全防护层。
- **Human-in-the-Loop 是合规刚需而非可选项**：根据全球金融监管趋势（NIST AI RMF、欧盟 AI Act、中国《生成式人工智能服务管理暂行办法》），AI 投资建议必须经过持牌人类顾问审批后才能下达给客户，系统需要完整的审计链路（Audit Trail）记录每条建议的生成依据和审批状态。
- **用户画像驱动个性化是核心差异化能力**：通过 KYC 合规流程构建风险承受能力、投资期限、资产规模、ESG 偏好等多维画像，所有投资建议必须与画像严格匹配——这既是合规要求，也是提升建议质量和用户信任的关键。

一句话总结：AI 投资顾问 Agent 的本质是"在合规框架下用多智能体协作实现个性化投资分析"——通过四层架构承载数据到决策的全链路，用三级风控（Pre-Trade + Real-Time + Model-Risk）和强制 Human-in-the-Loop 确保每条建议安全合规，用 Orchestrator + 专家子 Agent 的架构平衡分析深度与系统可控性。

---

### Q: 如何设计一个 AI 驱动的投资顾问 Agent 系统？需要考虑哪些关键架构决策和风控机制？

> 🤖 **Claude**

#### 1. 网络整合回答

AI 驱动的投资顾问 Agent 系统是当前 Agentic AI 在金融领域最具挑战性的应用场景之一。据 Gartner 预测，到 2026 年底将有 40% 的企业应用集成任务专属 AI Agent，而金融投顾场景因其高合规性、高实时性和高风险性要求，成为检验 Agent 架构成熟度的试金石。设计此类系统需要从**架构分层、合规治理、风控机制、可解释性**四个维度进行系统性思考。

**整体四层架构设计**：

（1）**数据接入层（Data Ingestion Layer）**：构建实时与批量混合的数据管道。实时数据包括 Level-2 行情（逐笔成交、五档盘口）、宏观经济事件流（央行利率决议、CPI 发布）和突发新闻流；批量数据包括上市公司财报（10-K/10-Q）、行业研报和另类数据（ESG 评分、供应链物流数据等）。关键技术选型是采用 Event-Driven Architecture，通过 Apache Kafka 做消息中间件，Flink 做实时流处理，并维护一个**金融知识图谱**（Financial Knowledge Graph）以关联公司关系网络、产业链上下游和宏观传导路径。2026 年的新趋势是引入 **Multimodal Data Processing**——将财报 PDF 中的表格、电话会议音频转录、甚至卫星遥感图像统一转化为结构化向量表示，供下游 Agent 消费。

（2）**分析层（Multi-Agent Analysis Layer）**：这是系统的智能核心，采用 **Orchestrator + Specialist Agents** 的多智能体编排模式。据 2025-2026 年的行业实践，multi-agent 系统查询量增长了 1445%，金融机构普遍从单一 Agent 迁移到多 Agent 协作架构。具体而言，Orchestrator Agent 接收用户查询后，将任务拆解并分派给多个专家 Agent 并行处理：Macro Analyst Agent 负责宏观环境评估（货币政策、地缘政治）；Equity Research Agent 负责个股基本面分析（DCF 估值、同业比较）；Quant Signal Agent 负责技术指标计算（RSI、MACD、布林带）；Sentiment Agent 负责舆情情绪量化评分。各 Agent 拥有独立的 System Prompt、Tool Set 和 Memory，最终由 Orchestrator 汇总各方观点，通过 **Weighted Ensemble** 策略生成综合研判。选择 Orchestrator 模式而非 Decentralized 模式的核心原因是：金融场景对**可审计性（Auditability）**和**确定性（Determinism）**要求极高，集中编排便于全局风控拦截和完整审计链路记录。

（3）**决策层（Portfolio Decision Layer）**：将多 Agent 分析结论转化为可执行的投资建议。此层的核心是**用户适当性匹配引擎**——根据 KYC（Know Your Customer）合规流程获取的用户画像（风险偏好、投资期限、资产规模、ESG 限制等），对 Agent 生成的原始建议进行个性化过滤和调整。投资组合优化采用 Black-Litterman 模型或 Risk Parity 策略，通过 Monte Carlo 模拟评估尾部风险（Tail Risk），确保建议方案在 VaR（Value at Risk）和 CVaR（Conditional VaR）约束下最优。Origin 公司的实践表明，其 AI 财务顾问通过 multi-agent 架构结合用户完整财务历史，在 SEC 监管框架下每条输出需通过 100+ 条受托责任、隐私和准确性标准的合规验证。

（4）**执行层（Execution & Compliance Layer）**：负责建议的合规审批、下达执行和事后监控。此层必须实现**强制性 Human-in-the-Loop（HITL）网关**——根据欧盟 AI Act 和 NIST AI Risk Management Framework，金融投资建议被归类为"高风险 AI 应用"，要求完整的人工审批链路和模型可解释性文档。执行层需要实现三级审批矩阵：低风险操作（如定投计划微调）可自动执行；中风险操作（如单次建仓）需持牌顾问确认；高风险操作（如大额调仓、衍生品建议）需合规官+投资总监双签。所有操作记录写入不可篡改的 Audit Trail，支持监管回溯查询。

**风控机制——贯穿全链路的三道防线**：

第一道防线是 **Pre-Trade Risk Control**（事前风控）：在建议生成前进行合规规则检查，包括单一标的持仓上限（如不超过总资产 10%）、行业集中度限制（如单一行业不超过 30%）、杠杆率约束、禁投名单（Restricted List）筛查、以及用户适当性匹配验证。

第二道防线是 **Real-Time Risk Monitoring**（实时风控）：持续监控组合级风险指标，包括 Portfolio VaR、最大回撤（Max Drawdown）、Beta 暴露、流动性风险等。当指标触发预设阈值时，系统自动执行 Circuit Breaker 机制——降级到保守模式或暂停 AI 建议功能，交由人类顾问接管。2026 年的新实践是引入**异常交易行为检测**（Anomaly Detection），通过对比用户历史操作模式，识别可能由 Prompt Injection 或账号盗用导致的异常请求。

第三道防线是 **Model Risk & AI Safety Control**（模型风控）：针对 LLM 特有风险的专项防护。据 BizTech Magazine 报道，LLM 幻觉（Hallucination）在金融场景的后果尤为严重——可能导致虚假的收益率承诺、不存在的投资标的推荐、或错误的财务数据引用。缓解策略包括：（a）**RAG 强制溯源**——所有数据引用必须来自可信数据库，Agent 输出必须附带 Citation；（b）**Guardrails 框架**——通过语义规则过滤绝对化收益承诺（如"保证年化 20%"）、未授权投资建议和误导性陈述；（c）**Cross-Verification**——关键数值（股价、市盈率、股息率）通过至少两个独立数据源交叉验证后才允许进入最终建议；（d）**对抗性防护**——防御 Prompt Injection（恶意用户试图绕过风控规则）、Data Poisoning（通过注入虚假新闻影响情绪分析）和 Deepfake 信息污染。EY 的实践表明，传统的不确定性度量（如 Entropy）在 LLM 产生高置信度幻觉时会失效，需要引入 Expected Calibration Error（ECE）等更精准的校准指标。

**数据安全与隐私保护**：用户的财务数据（资产规模、持仓明细、收入信息）属于高度敏感信息，系统需实现端到端加密、数据脱敏（Data Masking）、基于角色的访问控制（RBAC），并确保符合 GDPR/《个人信息保护法》等法规要求。LLM 的调用应采用 Private Deployment 或 Data Residency 方案，确保用户数据不泄露给第三方模型提供商。

#### 2. 结合实际例子

以下展示一个生产级投资顾问 Agent 系统的核心架构，重点突出风控引擎的三道防线和 Multi-Agent 协作流程：

```python
"""
AI 投资顾问 Agent 系统 V2 - 风控优先架构
重点：三道防线风控引擎 + 合规 Guardrails + Hallucination Detection
"""
import asyncio
from abc import ABC, abstractmethod
from dataclasses import dataclass, field
from datetime import datetime
from enum import Enum
from typing import Optional


# ==================== 风险等级与审批矩阵 ====================
class RiskLevel(Enum):
    LOW = "low"          # 自动执行：如定投微调
    MEDIUM = "medium"    # 持牌顾问确认
    HIGH = "high"        # 合规官 + 投资总监双签
    CRITICAL = "critical"  # 熔断：暂停 AI 建议，人工全面接管


class ApprovalStatus(Enum):
    PENDING = "pending"
    APPROVED = "approved"
    REJECTED = "rejected"
    ESCALATED = "escalated"


@dataclass
class AuditRecord:
    """不可篡改的审计记录——监管回溯查询的核心"""
    timestamp: str
    action: str
    agent_id: str
    input_summary: str
    output_summary: str
    risk_level: RiskLevel
    approval_status: ApprovalStatus
    reviewer_id: Optional[str] = None
    rejection_reason: Optional[str] = None
    citations: list[str] = field(default_factory=list)  # RAG 溯源引用


# ==================== 第一道防线：Pre-Trade Risk Control ====================
class PreTradeRiskGate:
    """
    事前风控门：建议生成前的合规规则检查
    任何一条规则不通过即拦截
    """

    def __init__(self, restricted_list: list[str] = None):
        self.restricted_list = restricted_list or []

    def check(self, ticker: str, action: str, position_pct: float,
              sector: str, user_max_position: float,
              user_max_sector_concentration: float,
              current_sector_exposure: float,
              user_restricted_sectors: list[str]) -> dict:

        violations = []

        # 规则 1：禁投名单检查
        if ticker in self.restricted_list:
            violations.append(f"[BLOCKED] {ticker} 在禁投名单中")

        # 规则 2：单一标的仓位上限
        if position_pct > user_max_position:
            violations.append(
                f"[POSITION] 建议仓位 {position_pct:.1%} 超过用户上限 "
                f"{user_max_position:.1%}"
            )

        # 规则 3：行业集中度限制
        projected_sector_exp = current_sector_exposure + position_pct
        if projected_sector_exp > user_max_sector_concentration:
            violations.append(
                f"[SECTOR] 行业 {sector} 预计暴露 {projected_sector_exp:.1%} "
                f"超过上限 {user_max_sector_concentration:.1%}"
            )

        # 规则 4：用户 ESG/偏好限制
        if sector in user_restricted_sectors:
            violations.append(f"[ESG] 行业 {sector} 在用户限制列表中")

        return {
            "passed": len(violations) == 0,
            "violations": violations,
            "checked_at": datetime.now().isoformat()
        }


# ==================== 第二道防线：Real-Time Risk Monitor ====================
class RealTimeRiskMonitor:
    """
    实时风控监控：持续追踪组合级风险指标
    触发阈值时自动执行 Circuit Breaker
    """

    def __init__(self, var_limit: float = 0.05,
                 max_drawdown_limit: float = 0.15,
                 intraday_loss_circuit_breaker: float = 0.08):
        self.var_limit = var_limit
        self.max_drawdown_limit = max_drawdown_limit
        self.intraday_loss_circuit_breaker = intraday_loss_circuit_breaker

    def evaluate_portfolio_risk(self, portfolio_var: float,
                                current_drawdown: float,
                                intraday_pnl_pct: float) -> dict:
        alerts = []
        risk_level = RiskLevel.LOW

        if abs(intraday_pnl_pct) >= self.intraday_loss_circuit_breaker:
            alerts.append(
                f"[CIRCUIT BREAKER] 日内损益 {intraday_pnl_pct:.2%} "
                f"触发熔断阈值 {self.intraday_loss_circuit_breaker:.2%}"
            )
            risk_level = RiskLevel.CRITICAL

        elif current_drawdown >= self.max_drawdown_limit:
            alerts.append(
                f"[DRAWDOWN] 当前回撤 {current_drawdown:.2%} "
                f"超过阈值 {self.max_drawdown_limit:.2%}"
            )
            risk_level = RiskLevel.HIGH

        elif portfolio_var >= self.var_limit:
            alerts.append(
                f"[VAR] 组合 VaR {portfolio_var:.2%} 超过限额 "
                f"{self.var_limit:.2%}"
            )
            risk_level = RiskLevel.MEDIUM

        return {
            "risk_level": risk_level,
            "alerts": alerts,
            "circuit_breaker_triggered": risk_level == RiskLevel.CRITICAL,
            "recommendation": self._get_action(risk_level)
        }

    @staticmethod
    def _get_action(risk_level: RiskLevel) -> str:
        actions = {
            RiskLevel.LOW: "正常运行，AI 建议可自动执行",
            RiskLevel.MEDIUM: "降级模式：仅允许减仓/保守建议",
            RiskLevel.HIGH: "警戒模式：所有建议需人工审批",
            RiskLevel.CRITICAL: "熔断：暂停 AI 建议，人工全面接管",
        }
        return actions[risk_level]


# ==================== 第三道防线：Model Risk & Guardrails ====================
class LLMOutputGuardrails:
    """
    模型风控：对 LLM 输出进行合规性检查和幻觉检测
    金融场景的 Guardrails 比通用场景严格得多
    """

    # 绝对禁止出现的表述模式
    FORBIDDEN_PATTERNS = [
        "保证收益", "稳赚不赔", "guaranteed return",
        "零风险", "100% safe", "必定上涨",
        "年化收益率超过", "内幕消息", "insider information",
    ]

    def validate_output(self, llm_response: str,
                        cited_data_points: list[dict]) -> dict:
        issues = []

        # 检查 1：禁止绝对化收益承诺
        for pattern in self.FORBIDDEN_PATTERNS:
            if pattern.lower() in llm_response.lower():
                issues.append(
                    f"[GUARDRAIL] 检测到违规表述: '{pattern}'"
                )

        # 检查 2：RAG 溯源验证——所有数据引用必须有出处
        if cited_data_points:
            for dp in cited_data_points:
                if not dp.get("source") or dp["source"] == "unknown":
                    issues.append(
                        f"[CITATION] 数据点 '{dp.get('claim', 'N/A')}' "
                        f"缺少可信来源"
                    )

        # 检查 3：数值合理性校验（检测幻觉）
        # 例如：PE 为负但推荐买入，股价偏差超过 5% 等
        for dp in (cited_data_points or []):
            if dp.get("type") == "price":
                claimed = dp.get("claimed_value", 0)
                verified = dp.get("verified_value", 0)
                if verified > 0 and abs(claimed - verified) / verified > 0.05:
                    issues.append(
                        f"[HALLUCINATION] 价格数据偏差过大: "
                        f"声称 {claimed} vs 实际 {verified}"
                    )

        # 检查 4：必须包含风险提示
        risk_keywords = ["风险提示", "风险警示", "投资有风险", "risk disclaimer"]
        has_risk_disclaimer = any(
            kw in llm_response.lower() for kw in risk_keywords
        )
        if not has_risk_disclaimer:
            issues.append("[COMPLIANCE] 缺少风险提示声明")

        return {
            "passed": len(issues) == 0,
            "issues": issues,
            "severity": "BLOCK" if any(
                "GUARDRAIL" in i or "HALLUCINATION" in i for i in issues
            ) else "WARNING"
        }


# ==================== Human-in-the-Loop 审批网关 ====================
class HITLApprovalGateway:
    """
    Human-in-the-Loop 审批网关
    根据风险等级路由到不同审批流
    """

    async def route_for_approval(self, recommendation: dict,
                                 risk_level: RiskLevel) -> ApprovalStatus:
        if risk_level == RiskLevel.LOW:
            # 低风险：自动批准，记录日志
            return ApprovalStatus.APPROVED

        elif risk_level == RiskLevel.MEDIUM:
            # 中风险：推送给持牌顾问审批
            print(f"  [HITL] 中风险建议已推送给持牌顾问审批: "
                  f"{recommendation.get('summary', '')}")
            # 实际场景中这里会调用审批系统 API
            # approval = await approval_system.submit(recommendation)
            return ApprovalStatus.PENDING

        elif risk_level == RiskLevel.HIGH:
            # 高风险：需要合规官 + 投资总监双签
            print(f"  [HITL] 高风险建议需双人审批: "
                  f"{recommendation.get('summary', '')}")
            return ApprovalStatus.ESCALATED

        else:
            # CRITICAL：熔断，直接拒绝
            print("  [HITL] 熔断状态：AI 建议已暂停")
            return ApprovalStatus.REJECTED


# ==================== 专家子 Agent 基类 ====================
class SpecialistAgent(ABC):
    """所有专家子 Agent 的基类"""

    def __init__(self, agent_id: str, llm_client=None):
        self.agent_id = agent_id
        self.llm_client = llm_client

    @abstractmethod
    async def analyze(self, context: dict) -> dict:
        """执行专项分析，返回结构化结果"""
        pass


class MacroEnvironmentAgent(SpecialistAgent):
    """宏观环境分析 Agent：评估货币政策、地缘政治、经济周期"""

    async def analyze(self, context: dict) -> dict:
        # 实际中调用 LLM + 宏观数据 API
        return {
            "agent_id": self.agent_id,
            "dimension": "macro",
            "assessment": "偏谨慎",
            "key_factors": [
                "美联储维持高利率环境，全球流动性偏紧",
                "国内稳增长政策持续发力，消费复苏缓慢",
            ],
            "impact_score": -0.2,  # -1 到 1，负值偏空
            "confidence": 0.75,
            "citations": ["央行 Q1 货币政策执行报告", "IMF 全球展望 2026.03"]
        }


class EquityResearchAgent(SpecialistAgent):
    """个股研究 Agent：基本面分析、估值、同业比较"""

    async def analyze(self, context: dict) -> dict:
        ticker = context.get("ticker", "未知")
        return {
            "agent_id": self.agent_id,
            "dimension": "equity",
            "ticker": ticker,
            "valuation": "合理偏高",
            "pe_ttm": 28.5,
            "industry_avg_pe": 25.0,
            "dcf_fair_value": 1680.0,
            "current_price": 1720.0,
            "margin_of_safety": -0.024,  # 负值表示高于公允价值
            "moat_score": 0.9,
            "confidence": 0.80,
            "citations": ["公司 2025 年报", "Wind 行业数据"]
        }


class SentimentAnalysisAgent(SpecialistAgent):
    """舆情情绪 Agent：新闻、社交媒体、分析师观点情绪量化"""

    async def analyze(self, context: dict) -> dict:
        return {
            "agent_id": self.agent_id,
            "dimension": "sentiment",
            "overall_sentiment": 0.35,  # -1 到 1
            "news_sentiment": 0.40,
            "social_sentiment": 0.25,
            "analyst_consensus": "增持（12 买入 / 5 持有 / 2 卖出）",
            "confidence": 0.65,
            "citations": ["东方财富舆情指数", "Wind 一致预期"]
        }


# ==================== Orchestrator Agent ====================
class InvestmentAdvisorOrchestratorV2:
    """
    投资顾问 Orchestrator Agent V2 - 风控优先架构
    核心流程：并行分析 → 综合研判 → 三道防线过滤 → HITL 审批 → 执行
    """

    def __init__(self, llm_client=None):
        self.llm_client = llm_client
        # 初始化专家子 Agent
        self.specialists = [
            MacroEnvironmentAgent("macro_agent", llm_client),
            EquityResearchAgent("equity_agent", llm_client),
            SentimentAnalysisAgent("sentiment_agent", llm_client),
        ]
        # 初始化三道防线
        self.pre_trade_gate = PreTradeRiskGate(
            restricted_list=["FAKE_COIN", "SCAM_STOCK"]
        )
        self.realtime_monitor = RealTimeRiskMonitor()
        self.guardrails = LLMOutputGuardrails()
        self.hitl_gateway = HITLApprovalGateway()
        # 审计日志
        self.audit_trail: list[AuditRecord] = []

    async def process_consultation(self, query: str, ticker: str,
                                   user_profile: dict) -> dict:
        """完整的咨询处理流程"""
        print(f"\n{'='*60}")
        print(f"用户查询: {query}")
        print(f"{'='*60}")

        # ---- 阶段 1：并行调度专家子 Agent ----
        print("\n[阶段 1] 并行调度专家子 Agent 分析...")
        context = {"query": query, "ticker": ticker}
        analysis_tasks = [agent.analyze(context) for agent in self.specialists]
        analysis_results = await asyncio.gather(*analysis_tasks)

        for result in analysis_results:
            print(f"  ✓ {result['dimension']} Agent 完成，"
                  f"置信度: {result['confidence']:.0%}")

        # ---- 阶段 2：加权综合研判 ----
        print("\n[阶段 2] 加权综合研判...")
        synthesis = self._weighted_synthesis(analysis_results)
        print(f"  综合评分: {synthesis['composite_score']:.2f} "
              f"(建议: {synthesis['action']})")

        # ---- 阶段 3：第一道防线 - Pre-Trade 风控 ----
        print("\n[阶段 3] 第一道防线: Pre-Trade 风控检查...")
        pre_trade_result = self.pre_trade_gate.check(
            ticker=ticker,
            action=synthesis["action"],
            position_pct=synthesis["suggested_position"],
            sector=user_profile.get("sector", "消费"),
            user_max_position=user_profile.get("max_position", 0.10),
            user_max_sector_concentration=user_profile.get("max_sector", 0.30),
            current_sector_exposure=user_profile.get("current_sector_exp", 0.15),
            user_restricted_sectors=user_profile.get("restricted_sectors", [])
        )
        if not pre_trade_result["passed"]:
            print(f"  ✗ Pre-Trade 风控拦截: {pre_trade_result['violations']}")
            return {"status": "BLOCKED", "reason": pre_trade_result["violations"]}
        print("  ✓ Pre-Trade 风控通过")

        # ---- 阶段 4：第二道防线 - 实时组合风险 ----
        print("\n[阶段 4] 第二道防线: 实时组合风险评估...")
        portfolio_risk = self.realtime_monitor.evaluate_portfolio_risk(
            portfolio_var=0.035,
            current_drawdown=0.06,
            intraday_pnl_pct=-0.02
        )
        print(f"  风险等级: {portfolio_risk['risk_level'].value}")
        print(f"  建议: {portfolio_risk['recommendation']}")
        if portfolio_risk["circuit_breaker_triggered"]:
            return {"status": "CIRCUIT_BREAKER", "reason": portfolio_risk["alerts"]}

        # ---- 阶段 5：生成 LLM 建议文本（模拟） ----
        print("\n[阶段 5] 生成投资建议文本...")
        llm_response = self._generate_advice_text(synthesis, analysis_results)

        # ---- 阶段 6：第三道防线 - Guardrails & 幻觉检测 ----
        print("\n[阶段 6] 第三道防线: Guardrails & 幻觉检测...")
        cited_data = [
            {"claim": f"{ticker} 当前股价", "type": "price",
             "claimed_value": 1720.0, "verified_value": 1718.5,
             "source": "Wind 实时行情"},
            {"claim": "PE-TTM", "type": "ratio",
             "claimed_value": 28.5, "source": "Wind 财务数据"},
        ]
        guardrail_result = self.guardrails.validate_output(
            llm_response, cited_data
        )
        if not guardrail_result["passed"]:
            severity = guardrail_result["severity"]
            print(f"  ✗ Guardrails 检测到问题 (severity={severity}): "
                  f"{guardrail_result['issues']}")
            if severity == "BLOCK":
                return {"status": "GUARDRAIL_BLOCKED",
                        "issues": guardrail_result["issues"]}
        else:
            print("  ✓ Guardrails 检查通过")

        # ---- 阶段 7：HITL 审批路由 ----
        risk_level = portfolio_risk["risk_level"]
        print(f"\n[阶段 7] HITL 审批路由 (风险等级: {risk_level.value})...")
        approval = await self.hitl_gateway.route_for_approval(
            {"summary": f"{ticker} {synthesis['action']}", "text": llm_response},
            risk_level
        )
        print(f"  审批状态: {approval.value}")

        # ---- 记录审计日志 ----
        self.audit_trail.append(AuditRecord(
            timestamp=datetime.now().isoformat(),
            action=f"{ticker}_{synthesis['action']}",
            agent_id="orchestrator_v2",
            input_summary=query[:100],
            output_summary=llm_response[:200],
            risk_level=risk_level,
            approval_status=approval,
            citations=[dp["source"] for dp in cited_data if dp.get("source")],
        ))

        return {
            "status": "SUCCESS" if approval == ApprovalStatus.APPROVED else approval.value,
            "advice": llm_response,
            "risk_level": risk_level.value,
            "approval_status": approval.value,
            "audit_id": len(self.audit_trail),
            "analysis_summary": {r["dimension"]: r.get("confidence", 0)
                                 for r in analysis_results},
        }

    def _weighted_synthesis(self, results: list[dict]) -> dict:
        """加权综合多 Agent 分析结论"""
        weights = {"macro": 0.25, "equity": 0.50, "sentiment": 0.25}
        scores = {}
        for r in results:
            dim = r["dimension"]
            if dim == "macro":
                scores[dim] = r.get("impact_score", 0)
            elif dim == "equity":
                scores[dim] = r.get("margin_of_safety", 0)
            elif dim == "sentiment":
                scores[dim] = r.get("overall_sentiment", 0)

        composite = sum(weights.get(d, 0) * s for d, s in scores.items())

        if composite > 0.1:
            action, position = "买入", 0.08
        elif composite > -0.1:
            action, position = "持有/小幅配置", 0.04
        else:
            action, position = "观望", 0.0

        return {
            "composite_score": composite,
            "action": action,
            "suggested_position": position,
            "dimension_scores": scores
        }

    @staticmethod
    def _generate_advice_text(synthesis: dict, analyses: list[dict]) -> str:
        """生成合规的建议文本（实际应调用 LLM）"""
        return (
            f"【投资建议】综合评分 {synthesis['composite_score']:.2f}，"
            f"建议操作：{synthesis['action']}。\n"
            f"宏观环境偏谨慎，个股基本面护城河深厚但估值偏高，"
            f"市场情绪中性偏乐观。建议小幅配置，仓位控制在 "
            f"{synthesis['suggested_position']:.0%} 以内。\n"
            f"风险提示：以上建议仅供参考，不构成任何投资承诺。"
            f"投资有风险，入市需谨慎。过往业绩不代表未来表现。"
        )


# ==================== 使用示例 ====================
async def demo():
    orchestrator = InvestmentAdvisorOrchestratorV2()

    user_profile = {
        "risk_tolerance": "moderate",
        "max_position": 0.10,
        "max_sector": 0.30,
        "current_sector_exp": 0.12,
        "restricted_sectors": ["博彩", "军工"],
        "sector": "消费",
    }

    result = await orchestrator.process_consultation(
        query="贵州茅台当前是否适合配置？建议多少仓位？",
        ticker="600519.SH",
        user_profile=user_profile,
    )

    print(f"\n{'='*60}")
    print("最终结果:")
    for k, v in result.items():
        print(f"  {k}: {v}")


# asyncio.run(demo())
```

#### 3. 面试核心回答

- **四层架构是金融 Agent 的工程骨架**：数据接入层（实时行情 + 另类数据 + 金融知识图谱）→ 分析层（Orchestrator + 专家子 Agent 并行分析）→ 决策层（用户画像匹配 + Portfolio Optimization）→ 执行层（HITL 审批 + Audit Trail）。每层都内嵌独立的风控检查点，形成纵深防御。
- **三道防线风控体系贯穿全链路**：Pre-Trade 风控（仓位限额、行业集中度、禁投名单、适当性匹配）→ Real-Time 风控（VaR/CVaR 监控、Circuit Breaker 熔断机制）→ Model Risk 风控（LLM 幻觉检测、Guardrails 合规过滤、RAG 强制溯源、对抗性攻击防御）。任何一道防线未通过都会拦截或降级建议。
- **Human-in-the-Loop 是监管刚需**：根据欧盟 AI Act 和 NIST AI RMF，金融投资建议属于高风险 AI 应用，必须实现分级审批矩阵（低风险自动执行 / 中风险顾问确认 / 高风险双签 / 熔断时人工接管），并维护完整的 Audit Trail 供监管回溯。
- **Multi-Agent 选 Orchestrator 模式而非去中心化模式**：金融场景对可审计性和确定性要求极高，集中编排便于全局风控拦截、统一审计和故障定位。通过并行调度专家子 Agent（宏观、个股、舆情、量化）兼顾分析深度与系统可控性。
- **模型可解释性是合规底线**：每条投资建议必须附带推理依据（Chain-of-Thought 记录）、数据来源引用（Citation）和风险提示声明，杜绝"黑箱推荐"。RAG 强制溯源 + Cross-Verification 双源校验是防幻觉的核心手段。

一句话总结：AI 投资顾问 Agent 系统的设计核心是"风控优先、合规驱动"——用四层架构组织数据到执行的全链路，用三道防线（Pre-Trade + Real-Time + Model Risk）实现纵深风控，用 Orchestrator + 专家子 Agent 的多智能体架构平衡分析能力与系统可控性，通过强制 Human-in-the-Loop 和完整 Audit Trail 满足金融监管对高风险 AI 应用的严格要求。
