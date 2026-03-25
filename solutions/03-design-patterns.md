# 03 - 设计模式 / Design Patterns

本文件收录设计模式章节的参考答案与深度解析。

---

### Q: 说下 ReAct 框架。它是如何将思维链和行动结合起来，以完成复杂任务的？了解 Plan-and-Solve 吗，Reflection 吗？

#### 1. 网络整合回答 / Comprehensive Answer

ReAct 的核心思想是把 Reasoning 和 Acting 交替进行：模型先基于当前信息形成一个短的思考，再决定下一步行动，比如调用搜索、数据库、计算器等工具；拿到工具结果后，再继续更新推理，直到完成任务。这样做的价值在于，模型不是一次性“脑补”完整答案，而是在外部反馈的帮助下逐步逼近正确结果。  
Plan-and-Solve 则更强调“先规划再执行”，先让模型把复杂任务拆成几个子步骤，再按计划求解，适合结构更稳定的长任务。Reflection / Reflexion 则是在任务结束后加入自我复盘，让模型识别错误来源、更新下一轮策略。三者可以组合：先规划，再按 ReAct 执行，中间或结束后做 Reflection。

#### 2. 结合实际例子 / Practical Example

以“调研竞品并输出分析报告”为例：

- ReAct：先搜索官网，再抓功能列表，再比较价格，边查边改判断。
- Plan-and-Solve：先列出“信息收集、维度整理、结论输出”三阶段，再逐步执行。
- Reflection：发现结论和原始证据不一致时，复盘是检索不足还是比较维度有问题，然后重跑部分步骤。

#### 3. 面试核心回答 / Core Interview Answer

- ReAct 通过“思考一步，行动一步”把推理和工具反馈闭环起来。
- Plan-and-Solve 强在先拆解任务，Reflection 强在事后纠错和策略改进。
- 复杂 Agent 往往不是只用一种模式，而是多模式组合。

一句话总结：ReAct 解决”边想边做”，Plan-and-Solve 解决”先拆再做”，Reflection 解决”做完再改”。

> 🤖 **Claude**

#### 1. 网络整合回答 / Comprehensive Answer

ReAct（Reasoning + Acting）是 Yao et al. 2022 年提出的框架，核心创新是让 LLM 在推理过程中交替输出 Thought（思考）和 Action（动作），形成 Thought → Action → Observation 的循环。与纯 Chain-of-Thought 只在内部推理不同，ReAct 每一步都可以通过外部工具获取真实信息，避免了 LLM 在长推理链中累积幻觉的问题。

ReAct 的关键优势在于”可验证的中间步骤”：每个 Action 调用工具后都会返回 Observation，模型可以根据真实反馈修正推理方向。这比一次性生成完整答案要可靠得多。

Plan-and-Solve（Wang et al. 2023）是对 Zero-shot CoT 的改进。标准 CoT 只说”let's think step by step”，模型可能遗漏步骤或计算错误。Plan-and-Solve 先让模型制定详细计划（”devise a plan”），再按计划逐步执行。它的增强版 PS+ 还会要求模型”extract relevant variables”和”calculate intermediate results”，进一步降低推理错误。

Reflection / Reflexion（Shinn et al. 2023）的核心是引入自我评估循环。Agent 执行完任务后，不是直接结束，而是评估结果质量、识别失败原因、生成改进策略，然后在下一轮尝试中应用这些反思。Reflexion 把短期经验存入 episodic memory，使模型能在多次尝试中持续改进，不需要更新权重。

三者的关系可以用一个公式概括：Plan-and-Solve 负责”制定计划”，ReAct 负责”执行计划中的每一步”，Reflection 负责”执行后复盘并改进下一轮计划”。

#### 2. 结合实际例子 / Practical Example

```python
# ReAct 模式的实现骨架
class ReActAgent:
    def __init__(self, llm, tools: dict):
        self.llm = llm
        self.tools = tools  # {“search”: search_fn, “calculator”: calc_fn}
        self.scratchpad = []  # 记录 Thought/Action/Observation

    def run(self, question: str, max_steps: int = 8):
        prompt = f”Question: {question}\n”

        for step in range(max_steps):
            # Thought: 模型推理当前该做什么
            response = self.llm.generate(
                prompt + self._format_scratchpad()
                + f”\nThought {step+1}:”
            )
            thought = response.strip()
            self.scratchpad.append((“Thought”, thought))

            # 判断是否已有最终答案
            if “Final Answer:” in thought:
                return thought.split(“Final Answer:”)[-1].strip()

            # Action: 解析要调用的工具
            action_response = self.llm.generate(
                prompt + self._format_scratchpad()
                + f”\nAction {step+1}:”
            )
            tool_name, tool_input = self._parse_action(action_response)

            # Observation: 执行工具并获取结果
            observation = self.tools[tool_name](tool_input)
            self.scratchpad.append((“Action”, f”{tool_name}[{tool_input}]”))
            self.scratchpad.append((“Observation”, observation))

        return “超过最大步数，任务未完成”


# Reflection 模式：在 ReAct 之上增加反思循环
class ReflectiveAgent(ReActAgent):
    def __init__(self, llm, tools, max_trials=3):
        super().__init__(llm, tools)
        self.reflections = []  # 长期反思记忆

    def run_with_reflection(self, question: str):
        for trial in range(self.max_trials):
            result = self.run(question)
            # 自我评估
            evaluation = self.llm.generate(
                f”任务：{question}\n结果：{result}\n”
                f”请评估结果质量，指出不足并给出改进建议。”
            )
            if “质量合格” in evaluation:
                return result
            # 将反思存入记忆，下一轮使用
            self.reflections.append(evaluation)
            self.scratchpad = []  # 重置，准备下一轮
        return result
```

#### 3. 面试核心回答 / Core Interview Answer

- ReAct 通过 Thought-Action-Observation 循环，让推理和工具调用交替进行，避免长链幻觉。
- Plan-and-Solve 在执行前先制定结构化计划，比纯 CoT 的步骤遗漏率更低。
- Reflection 通过自我评估和改进记忆，让 Agent 在多次尝试中持续进步，无需更新权重。
- 三者不是互斥的，实战中常组合：Plan-and-Solve 定计划 → ReAct 执行 → Reflection 复盘。
- 选择哪种模式取决于任务特征：简单查询用 ReAct，结构化长任务用 Plan-and-Solve，允许重试的任务加 Reflection。

一句话总结：ReAct 让 Agent “边想边做”，Plan-and-Solve 让它”先想清楚再做”，Reflection 让它”做完了知道哪里做得不好”。

---

### Q: 在 Agent 的设计中，”规划能力”至关重要。请谈谈目前有哪些主流方法可以赋予 LLM 规划能力？例如 CoT、ToT、GoT 等。

#### 1. 网络整合回答 / Comprehensive Answer

给 LLM 规划能力，常见思路可以分成几类。第一类是线性推理，如 CoT，让模型显式分步思考；第二类是“先规划后执行”，如 Plan-and-Solve、Plan-and-Execute；第三类是分支搜索，如 Tree-of-Thought，把多个候选思路展开再评估；第四类是图结构推理，如 Graph-of-Thought，用节点和边表达依赖关系；第五类是带自我修正的规划，如 Reflection、Re-planning。  
如果任务结构简单，CoT 往往已经够用；如果任务路径分叉多、局部决策影响全局，则 ToT / GoT 更适合；如果任务中途经常遇到工具失败或外界变化，就需要 Re-planning。规划能力本质上不是一个技巧，而是一套“任务分解 + 状态评估 + 策略更新”的机制。

#### 2. 结合实际例子 / Practical Example

例如让 Agent 完成“排查线上故障”：

- CoT：顺序列出检查步骤，适合简单故障。
- ToT：同时展开“网络问题、数据库问题、配置问题”三条分支。
- GoT：把服务依赖关系、日志证据、报警节点画成图，再沿图搜索根因。
- Re-planning：发现最初假设错误后，动态调整排查顺序。

#### 3. 面试核心回答 / Core Interview Answer

- CoT 适合线性任务，ToT / GoT 适合分支与依赖复杂的任务。
- Plan-and-Execute 解决任务拆分，Reflection / Re-planning 解决执行中的修正。
- 真正的规划能力不止靠 Prompt，还要配合状态、工具和反馈机制。

一句话总结：LLM 的规划能力来自”显式分解任务并在执行中持续更新计划”，而不是一次生成一个完美答案。

---

### Q: 对比 Agent 执行中的并行与串行任务模式。如何在 Agentic 工作流中决定何时并行、何时串行？设计上有什么原则和权衡？

> 🤖 **Claude**

#### 1. 网络整合回答

在 Agentic AI 系统中，任务执行模式分为**串行（Sequential）**和**并行（Parallel）**两大类，它们是工作流编排的基础构建块。

**串行执行**是指任务按预定义顺序依次执行，Agent A 的输出作为 Agent B 的输入，形成线性流水线（Pipeline）。串行模式天然保证了上下文的连贯性和因果一致性，适用于存在**强数据依赖**的场景——例如”先检索文档 → 再摘要 → 再生成回答”这类链式任务。串行的缺点是延迟会线性累加：如果每个步骤耗时 2 秒，5 个步骤就需要 10 秒。

**并行执行**是指多个无依赖关系的子任务同时分发给不同 Agent 并发处理，最后由一个 Aggregator 汇总结果。这对应分布式系统中经典的 **Fan-out/Fan-in（扇出/扇入）** 模式，也叫 **Scatter-Gather** 或 **Map-Reduce** 模式。并行执行的核心优势是**降低端到端延迟**——总耗时取决于最慢的那条分支（critical path），而不是所有任务耗时之和。

如何决定何时并行、何时串行？核心在于**依赖分析（Dependency Analysis）**。业界主流做法是将任务建模为 **DAG（有向无环图）**：每个任务是图中的节点，数据依赖是有向边。没有入边的节点可以立即并行启动；有入边的节点必须等所有上游完成后才能执行。Google 在 2026 年初总结的八大 Multi-Agent 设计模式中，将 Sequential、Loop 和 Parallel 列为三种基础执行模式，并建议**默认使用串行，当延迟成为瓶颈且任务相互独立时再切换到并行**。

设计原则与权衡包括：（1）**正确性优先**——有状态共享或写冲突的任务必须串行，否则会出现竞态条件（Race Condition）；（2）**延迟 vs 资源成本**——并行降低延迟但消耗更多计算资源和 API 调用配额；（3）**错误传播**——串行模式中一个步骤失败即可停止整条链，并行模式需要设计部分失败处理策略（Partial Failure Handling），如超时重试、降级返回；（4）**结果一致性**——并行分支的结果可能存在冲突，Aggregator 需要有冲突解决（Conflict Resolution）机制；（5）**可观测性**——并行执行的调试和日志追踪比串行复杂得多，需要 Trace ID 和 Span 来关联并发分支。

LAMaS（Latency-Aware Multi-Agent System）等前沿框架进一步提出了**层级并行（Layer-wise Parallel Execution）**思路：通过自动识别并移除不必要的执行依赖，学习出更短 critical path 的执行图，在保证正确性的前提下最大化并行度。

#### 2. 结合实际例子

以下是一个支持并行和串行混合调度的 Agent 任务调度器实现：

```python
import asyncio
from dataclasses import dataclass, field
from enum import Enum
from typing import Any, Callable, Awaitable


class ExecMode(Enum):
    SEQUENTIAL = “sequential”
    PARALLEL = “parallel”


@dataclass
class Task:
    name: str
    execute: Callable[..., Awaitable[Any]]  # 异步可调用
    dependencies: list[str] = field(default_factory=list)  # 依赖的任务名称


class AgentTaskScheduler:
    “””基于 DAG 的 Agent 任务调度器，自动识别可并行的任务组”””

    def __init__(self, tasks: list[Task]):
        self.tasks = {t.name: t for t in tasks}
        self.results: dict[str, Any] = {}

    def _build_execution_layers(self) -> list[list[str]]:
        “””拓扑排序，将任务分成可并行执行的层”””
        in_degree = {name: 0 for name in self.tasks}
        for t in self.tasks.values():
            for dep in t.dependencies:
                in_degree[t.name] += 1  # 计算入度

        layers = []
        remaining = set(self.tasks.keys())

        while remaining:
            # 找出所有入度为 0 的任务 —— 它们可以并行执行
            ready = [
                name for name in remaining
                if all(dep not in remaining for dep in self.tasks[name].dependencies)
            ]
            if not ready:
                raise ValueError(“检测到循环依赖，无法调度”)
            layers.append(ready)
            remaining -= set(ready)

        return layers

    async def run(self) -> dict[str, Any]:
        “””按层执行：层内并行（Fan-out），层间串行（等待 Fan-in）”””
        layers = self._build_execution_layers()
        print(f”执行计划: {len(layers)} 层”)

        for i, layer in enumerate(layers):
            mode = ExecMode.PARALLEL if len(layer) > 1 else ExecMode.SEQUENTIAL
            print(f”  第 {i+1} 层 [{mode.value}]: {layer}”)

            if mode == ExecMode.PARALLEL:
                # Fan-out: 所有无依赖任务并发执行
                coros = [
                    self._execute_task(name) for name in layer
                ]
                results = await asyncio.gather(*coros, return_exceptions=True)
                for name, result in zip(layer, results):
                    if isinstance(result, Exception):
                        print(f”    ⚠ 任务 {name} 失败: {result}”)
                        self.results[name] = None  # 降级处理
                    else:
                        self.results[name] = result
            else:
                # 单任务直接串行执行
                for name in layer:
                    self.results[name] = await self._execute_task(name)

        return self.results

    async def _execute_task(self, name: str) -> Any:
        task = self.tasks[name]
        # 收集依赖任务的结果作为输入
        dep_results = {dep: self.results[dep] for dep in task.dependencies}
        print(f”    ▶ 执行 {name} (依赖输入: {list(dep_results.keys())})”)
        result = await task.execute(dep_results)
        print(f”    ✓ 完成 {name}”)
        return result


# ---------- 使用示例 ----------
async def search_web(inputs):
    await asyncio.sleep(1)  # 模拟网络请求
    return {“web_results”: “搜索结果...”}

async def query_database(inputs):
    await asyncio.sleep(1.5)
    return {“db_results”: “数据库记录...”}

async def call_api(inputs):
    await asyncio.sleep(0.8)
    return {“api_results”: “API 返回...”}

async def summarize(inputs):
    “””依赖前三个任务的结果，必须串行等待”””
    all_data = {k: v for dep in inputs.values() for k, v in dep.items()}
    await asyncio.sleep(0.5)
    return {“summary”: f”基于 {len(all_data)} 个数据源的综合摘要”}

async def generate_report(inputs):
    summary = inputs[“summarize”][“summary”]
    await asyncio.sleep(0.3)
    return {“report”: f”最终报告 — {summary}”}


async def main():
    tasks = [
        Task(“search_web”, search_web),          # 无依赖 → 第1层
        Task(“query_database”, query_database),   # 无依赖 → 第1层
        Task(“call_api”, call_api),               # 无依赖 → 第1层
        Task(“summarize”, summarize,              # 依赖前三个 → 第2层
             dependencies=[“search_web”, “query_database”, “call_api”]),
        Task(“generate_report”, generate_report,  # 依赖 summarize → 第3层
             dependencies=[“summarize”]),
    ]

    scheduler = AgentTaskScheduler(tasks)
    results = await scheduler.run()
    # 执行计划: 3 层
    #   第 1 层 [parallel]:   search_web, query_database, call_api  (~1.5s)
    #   第 2 层 [sequential]: summarize                             (~0.5s)
    #   第 3 层 [sequential]: generate_report                       (~0.3s)
    # 总耗时约 2.3s，而纯串行需要 1+1.5+0.8+0.5+0.3 = 4.1s

asyncio.run(main())
```

该调度器的核心思路：通过拓扑排序自动将任务分层，同一层内的任务没有相互依赖，可以用 `asyncio.gather` 并行执行（Fan-out）；层与层之间必须等待上一层全部完成（Fan-in）才能继续。这就是 DAG 调度的精髓——**自动发现最大并行度，同时保证依赖正确性**。

#### 3. 面试核心回答

- **串行（Sequential）适用于存在数据依赖的链式任务**，保证上下文连贯和因果一致性，缺点是延迟线性累加。
- **并行（Parallel）适用于多个独立子任务**，通过 Fan-out/Fan-in 模式将总耗时压缩到最慢分支的耗时，但需要更多资源和更复杂的错误处理。
- **决策依据是依赖分析**：将任务建模为 DAG，通过拓扑排序自动识别可并行的任务层，无依赖的任务并行、有依赖的任务串行。
- **关键权衡包括四点**：正确性（避免竞态条件）、延迟 vs 资源成本、部分失败处理策略、以及并发执行的可观测性。
- **设计原则是”默认串行，按需并行”**：先保证正确性，当延迟成为瓶颈且任务确实独立时，再引入并行。

一句话总结：串行保证顺序正确，并行压缩执行时间，核心决策工具是 DAG 依赖分析——无依赖则并行，有依赖则串行，先求正确再求快。
