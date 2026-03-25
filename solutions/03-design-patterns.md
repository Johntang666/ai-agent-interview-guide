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

---

### Q: 详细解释 ReAct 模式的工作流程和优缺点。

#### 1. 网络整合回答 / Comprehensive Answer

ReAct 的核心流程是 `Thought -> Action -> Observation` 循环。模型先根据当前上下文给出一步推理，再决定是否调用搜索、数据库、浏览器、代码执行器等工具；工具返回观察结果后，模型再继续下一步推理，直到任务完成或触发终止条件。它的优势在于把"思考"和"拿事实"绑在一起，减少纯语言推理一路脑补到底带来的幻觉。  
它尤其适合开放式信息获取、多步排查和工具密集型任务，因为模型可以边查边改计划。缺点也很明显：每一步都依赖模型决策，调用轮数多时延迟和成本会迅速上升；如果工具描述不清、观察结果噪声大，模型可能进入无效循环；另外 ReAct 对终止条件和安全边界要求很高，否则容易多跑或误调用工具。

#### 2. 结合实际例子 / Practical Example

- 用户让 Agent 调查竞品价格。
- 第一步先搜索官网，第二步读取价格页，第三步调用汇率工具统一换算，第四步总结差异。
- 如果某一步网页打不开，Agent 可以改成搜索新闻或公开文档，而不是直接失败。

#### 3. 面试核心回答 / Core Interview Answer

- ReAct 用"推理一步，行动一步"形成闭环。
- 优势是更贴近真实世界反馈，缺点是成本高、轮数多、易循环。
- 适合开放任务和工具密集任务，不适合每一步都很昂贵的场景。

一句话总结：ReAct 的本质不是让模型多想，而是让模型在外部反馈中修正自己的想法。

---

### Q: Plan-and-Execute 模式与 ReAct 模式有什么区别？各自适用什么场景？

#### 1. 网络整合回答 / Comprehensive Answer

Plan-and-Execute 和 ReAct 的差别，核心在于"计划是在前面一次性形成，还是在执行过程中边走边改"。Plan-and-Execute 先让模型生成高层任务计划，再由执行器逐步完成每个子任务，适合结构稳定、步骤可预先拆分的任务，例如写报告、整理资料、批量处理流程。ReAct 则更适合环境不确定、需要频繁根据外部反馈调整路径的任务。  
从工程角度看，Plan-and-Execute 的优点是流程更可解释、并行机会更多、便于人为审查计划；缺点是前期计划可能错，执行时如果环境变化大，需要额外补 re-planning。ReAct 的优点是灵活，但对每一步都依赖模型判断，成本和不确定性更高。

#### 2. 结合实际例子 / Practical Example

- 写市场研究报告：先列大纲、再搜资料、再写正文，更适合 Plan-and-Execute。
- 线上故障排查：日志、监控、接口状态不断变化，更适合 ReAct。
- 真实项目里也常先用 Plan-and-Execute 做框架，再在单个步骤内部用 ReAct。

#### 3. 面试核心回答 / Core Interview Answer

- Plan-and-Execute 强在先拆解，ReAct 强在边执行边调整。
- 稳定流程选 Plan-and-Execute，探索型任务选 ReAct。
- 两者常组合使用，而不是二选一。

一句话总结：Plan-and-Execute 更像先出施工图，ReAct 更像现场边干边判断。

---

### Q: 解释 Reflexion 模式如何让 Agent 从错误中学习。

#### 1. 网络整合回答 / Comprehensive Answer

Reflexion 的关键不是更新模型参数，而是把失败经验转成可复用的文字反思。Agent 完成一次任务后，不直接结束，而是对结果做自评，回答"哪里错了、为什么错、下次怎么改"；这些反思会被存入短期或情景记忆，在下一轮执行时作为额外上下文，让模型避免重复犯同类错误。  
这种机制特别适合允许重试的任务，例如代码修复、网页操作、信息检索。它的价值在于把单次失败变成下一次尝试的先验约束。局限是反思本身也可能错，所以反思内容最好有依据，例如基于测试结果、规则校验或人类反馈，而不是纯自说自话。

#### 2. 结合实际例子 / Practical Example

- 代码 Agent 第一次提交的补丁没通过测试。
- Reflexion 模块总结："失败原因是只修了空指针，没有处理空数组分支。"
- 第二轮执行时，Agent 会优先检查边界条件，而不是重复上一版补丁。

#### 3. 面试核心回答 / Core Interview Answer

- Reflexion 通过"自评 + 记忆"让 Agent 在多轮尝试中改进。
- 它学到的是策略经验，不是模型权重。
- 最适合允许试错且有明确反馈信号的任务。

一句话总结：Reflexion 让 Agent 把错误经历变成下一次行动的约束条件。

---

### Q: LATS（Language Agent Tree Search）是如何结合树搜索与 LLM 的？

#### 1. 网络整合回答 / Comprehensive Answer

LATS 可以看成把经典树搜索引入语言智能体。模型不再只沿单一路径往下走，而是先生成多个候选动作或思路，把它们当作树的分支，再通过价值评估选择更有前景的分支继续展开。这样做的目的是避免单条推理路径早早走偏。  
LLM 在这里通常承担三个角色：生成候选动作、评估当前状态、给分支打分或剪枝。相比普通 ReAct，LATS 更适合复杂决策和搜索空间较大的任务，比如网页导航、博弈式规划、复杂排查。但代价是计算量高，实现复杂，而且必须控制分支爆炸。

#### 2. 结合实际例子 / Practical Example

- 网页任务中，Agent 可以同时考虑"去搜索栏搜"、"打开帮助页"、"进入账户页"三条路径。
- 每探索一步都根据当前页面反馈给分，低分路径被剪掉，高分路径继续扩展。
- 最终选择得分最高的一条完成任务。

#### 3. 面试核心回答 / Core Interview Answer

- LATS = LLM 生成分支 + 搜索算法评估分支。
- 它通过多路径探索降低早期错误决策带来的损失。
- 适合复杂搜索任务，但成本明显高于单路径 Agent。

一句话总结：LATS 的核心不是让模型想更长，而是让模型同时保留多条可能的路。

---

### Q: Chain-of-Thought（CoT）和 Tree-of-Thought（ToT）在 Agent 推理中分别扮演什么角色？

#### 1. 网络整合回答 / Comprehensive Answer

CoT 的作用是把原本隐式的思考过程显式化，让模型按步骤拆解问题，适合线性、单路径的推理任务。它是很多 Agent 规划和工具选择的基础能力，因为没有分步推理，模型很难稳定地决定下一步。ToT 则进一步引入"多候选分支"，让模型在中间状态就能比较不同思路，而不是一次押注一条路线。  
因此，CoT 更像基础推理骨架，ToT 更像在复杂空间里的分支搜索机制。前者成本低、实现简单；后者更强但更贵。Agent 设计里，简单任务通常 CoT 足够，只有在路径依赖强、局部选择影响全局时，ToT 才值得引入。

#### 2. 结合实际例子 / Practical Example

- 算一笔报销总额，用 CoT 逐项相加就够了。
- 规划一次跨城市差旅行程，需要比较多条路线和时间窗口，用 ToT 更合理。

#### 3. 面试核心回答 / Core Interview Answer

- CoT 负责线性拆解，ToT 负责多路径探索。
- CoT 是低成本基础能力，ToT 是高成本增强能力。
- Agent 是否需要 ToT，取决于任务是否存在明显分支决策。

一句话总结：CoT 让模型会分步想，ToT 让模型会比较不同想法。

---

### Q: 什么是 Router Agent 模式？如何设计一个高效的路由策略？

#### 1. 网络整合回答 / Comprehensive Answer

Router Agent 的职责是先不做任务本身，而是判断"这个请求该交给谁处理"。它通常位于系统入口，根据用户意图、风险等级、上下文状态、成本预算等因素，把请求路由到 RAG、代码 Agent、客服 Agent、人工审核链路或不同模型。  
高效路由策略的关键是分层判断。第一层做轻量分类，快速识别领域和风险；第二层再决定模型、工具或子 Agent；第三层设置回退机制，当置信度不足时走保守路径。好的路由器不追求每次都最聪明，而是追求稳定、快、误路由代价低。生产里常见做法是让小模型或规则先做粗路由，大模型只处理边界情况。

#### 2. 结合实际例子 / Practical Example

- 售后请求先经过 Router。
- "查物流" 走 FAQ + 工具查询链路。
- "申请退款" 走高风险审批链路。
- "投诉订单异常并要求补偿" 则同时触发人工兜底。

#### 3. 面试核心回答 / Core Interview Answer

- Router Agent 本质是任务分发器。
- 路由策略要考虑意图、风险、成本和置信度。
- 先做粗分类，再做精细分发，低置信度优先走保守路径。

一句话总结：高效路由不是把请求分得最细，而是把高代价的误分发降到最低。

---

### Q: 解释 Supervisor 模式在多 Agent 系统中的应用。

#### 1. 网络整合回答 / Comprehensive Answer

Supervisor 模式指由一个中心协调 Agent 负责拆解任务、分配子任务、汇总结果和处理冲突，其他专业 Agent 专注完成各自职责。它的优点是控制面集中，流程清晰，权限边界好管，适合企业内部多角色协作场景。  
它的代价是 Supervisor 容易成为瓶颈和单点故障，而且如果它的判断有偏差，会把错误放大到整个系统。为降低风险，通常会把 Supervisor 设计成轻决策、重编排角色，重要事实仍由专业 Agent 或工具返回，冲突解决也尽量基于规则或证据，而不是全靠中心 Agent 主观拍板。

#### 2. 结合实际例子 / Practical Example

- 采购主 Agent 接到请求后，把"供应商筛选"交给采购 Agent，"合规审查"交给法务 Agent，"合同生成"交给文档 Agent。
- 最终由 Supervisor 汇总结果并决定是否进入人工审批。

#### 3. 面试核心回答 / Core Interview Answer

- Supervisor 模式的核心是中心调度、专业执行。
- 它提升了可控性，但也引入中心瓶颈。
- 适合强调治理和权限边界的企业场景。

一句话总结：Supervisor 模式牺牲了一部分去中心化灵活性，换来了更强的流程治理能力。

---

### Q: 什么是 "Agent as a Tool" 模式？它如何实现 Agent 的层级化组织？

#### 1. 网络整合回答 / Comprehensive Answer

"Agent as a Tool" 的思路是把一个子 Agent 封装成上层 Agent 可调用的工具。对上层来说，它不需要知道子 Agent 内部怎么规划和执行，只知道这个工具能完成某类复杂任务，例如"做网页研究"、"生成合同草案"、"修复单个 bug"。  
这样做的价值在于分层组织复杂系统。顶层 Agent 只负责任务拆解和编排，底层专业 Agent 自己处理细节。这既降低了主 Agent 的认知负担，也让不同子系统可以独立演进。需要注意的是，子 Agent 作为工具后，仍然要暴露清晰的输入输出、时延预期、失败语义和权限边界，否则系统会出现"工具调用看似简单，实际黑盒过深"的问题。

#### 2. 结合实际例子 / Practical Example

- 主 Agent 收到"生成月度经营分析报告"的请求。
- 它调用 `research_agent` 收集外部信息，调用 `sql_agent` 跑内部报表，调用 `writer_agent` 生成总结。
- 对主 Agent 来说，这三个都是工具；对各子 Agent 来说，它们内部仍可能各自运行完整的 ReAct 流程。

#### 3. 面试核心回答 / Core Interview Answer

- Agent as a Tool 是把复杂能力封装成更高层可调用的能力单元。
- 它能实现分层编排和职责隔离。
- 关键在于定义好输入输出和失败边界，避免子 Agent 变成不可控黑盒。

一句话总结：它让 Agent 系统像服务架构一样分层，而不是所有逻辑都堆在一个大脑里。

---

### Q: 对比串行（Sequential）、并行（Parallel）、和图（Graph）Agent 编排方式。

#### 1. 网络整合回答 / Comprehensive Answer

串行编排最简单，适合每一步都依赖上一步输出的任务，优势是易理解、易调试，缺点是延迟会线性累积。并行编排适合多个彼此独立的子任务，可以显著缩短端到端耗时，但会增加资源消耗、冲突合并和部分失败处理复杂度。图式编排则是在两者之上，把节点依赖、条件路由、循环、回退和人工介入都显式表达出来，适合复杂生产系统。  
所以三者更像复杂度阶梯，而不是谁先进谁落后。多数系统从串行起步，在局部引入并行，在整体上用图来承载复杂控制流。真正的工程重点不是追求图，而是让编排方式和任务依赖结构匹配。

#### 2. 结合实际例子 / Practical Example

- 串行：检索 -> 生成 -> 校验。
- 并行：同时查数据库、搜索网页、调第三方 API。
- 图：如果校验失败则回到重写节点，如果高风险则走人工审批，否则直接执行。

#### 3. 面试核心回答 / Core Interview Answer

- 串行最稳、并行最快、图最灵活。
- 是否选图，取决于你是否需要分支、回环和恢复。
- 最佳实践通常是局部并行、全局图式、从简单开始。

一句话总结：串行和并行解决的是执行方式，图解决的是复杂流程的表达方式。

---

### Q: 如何设计一个具备 self-correction 能力的 Agent？

#### 1. 网络整合回答 / Comprehensive Answer

self-correction Agent 的关键不是"让模型再想一遍"，而是给它明确的校验信号和纠错入口。常见设计分三步：先执行主任务，再用规则、测试、对比模型或外部验证器检查结果，最后基于错误类型决定重试、重规划还是人工接管。没有校验器的 self-correction 很容易退化成无意义的自我重复。  
有效的纠错系统通常会把错误分类，例如格式错误、事实错误、工具调用错误、逻辑错误，对不同类型采用不同修复策略。再往前一步，可以把纠错经验写入短期记忆，避免每次都从零重试。

#### 2. 结合实际例子 / Practical Example

- 代码 Agent 写完补丁后先跑测试。
- 如果是 lint 失败，就走格式修正器。
- 如果是单测失败，就把失败堆栈和相关代码喂回修复节点。
- 连续失败超过阈值时升级到人工 review。

#### 3. 面试核心回答 / Core Interview Answer

- self-correction 要依赖外部校验，不靠模型自言自语。
- 错误要分类处理，不同错误走不同修复路径。
- 要设置重试上限和升级机制，防止无限循环。

一句话总结：真正的 self-correction 是"先验证，再改正"，而不是"多生成几次碰碰运气"。

---

### Q: 什么是 Agentic RAG？它与传统 RAG 有什么区别？

#### 1. 网络整合回答 / Comprehensive Answer

传统 RAG 通常是固定流程：用户提问后做检索，再把结果塞进上下文生成答案。Agentic RAG 则把"是否检索、检索几次、检索什么、什么时候改写 query、什么时候换数据源"这些决策交给 Agent 动态决定。也就是说，检索不再是一个固定前置步骤，而是执行过程中的可调用能力。  
这样做的好处是灵活，尤其适合复杂问题、信息不完整或需要多跳检索的任务；代价是系统更复杂，更依赖路由、工具治理和评测。如果底层检索质量一般、任务又很简单，Agentic RAG 反而可能比传统 RAG 更慢、更贵、更不稳定。

#### 2. 结合实际例子 / Practical Example

- 用户问："对比公司 A 和 B 最近两年的海外并购策略。"
- 传统 RAG 可能只检一次。
- Agentic RAG 会先检公司 A，再检公司 B，再发现年份信息不足，改写 query 做第二轮检索，最后再总结。

#### 3. 面试核心回答 / Core Interview Answer

- 传统 RAG 是固定流程，Agentic RAG 是动态检索决策。
- Agentic RAG 更灵活，但成本和治理复杂度更高。
- 简单问答未必需要 Agentic RAG，复杂多跳任务更受益。

一句话总结：Agentic RAG 的升级点，不是检索更花哨，而是检索从流程步骤变成了可调度能力。

---

### Q: 解释 "Human-in-the-Loop" 模式在 Agent 系统中的设计要点。

#### 1. 网络整合回答 / Comprehensive Answer

Human-in-the-Loop 的目标不是让人把所有步骤都手工接管，而是在高风险、不确定或业务关键节点引入人工判断。设计要点通常有四个：第一，明确什么情况下必须让人介入，比如高金额支付、删除数据、低置信度决策；第二，给人工足够上下文，包括 Agent 计划、证据、待执行动作和风险说明；第三，人工决策后系统要能继续执行，而不是整个流程推倒重来；第四，人工反馈要能沉淀成后续规则或评测样本。  
很多团队做不好这一模式，是因为只做了一个"人工审批按钮"，却没有把审批点、恢复点和反馈闭环设计进去。真正有价值的 HITL，不是打断系统，而是让系统学会在该交棒时交棒。

#### 2. 结合实际例子 / Practical Example

- 财务 Agent 在准备打款前触发审批。
- 审批界面展示：收款方、金额、依据、工具返回结果、Agent 计划。
- 财务人员可以批准、拒绝或要求重查，Agent 根据结果继续或回退。

#### 3. 面试核心回答 / Core Interview Answer

- 人工介入点要基于风险和不确定性设计。
- 给人看的不是最终结论，而是足够做决策的证据和上下文。
- 审批后要支持恢复执行和反馈沉淀。

一句话总结：Human-in-the-Loop 不是为了替代 Agent，而是为了在关键节点校正 Agent。

---

### Q: 如何实现 Agent 的动态规划与重规划（Re-planning）？

#### 1. 网络整合回答 / Comprehensive Answer

动态规划与重规划的核心是让计划成为"可更新对象"，而不是一次性文本。实践中通常会把计划拆成任务列表、依赖关系、完成状态和失败原因；当外部环境变化、工具失败、约束更新时，系统只重算受影响的部分，而不是整条链路全部重来。  
要实现这一点，至少需要三个能力：状态持久化，知道当前做到了哪；失败分类，知道为什么要改计划；决策策略，知道是局部修补还是整体重算。很多系统名义上支持 re-planning，实际上只是重新 prompt 一次，这样容易丢上下文、重复工作、成本失控。

#### 2. 结合实际例子 / Practical Example

- 旅行规划 Agent 已订好机票，准备订酒店时发现城市临时有大型活动。
- 系统不应重做整份旅行计划，而是只更新酒店搜索、预算和时间安排相关子任务。
- 如果预算上限也变化，再局部重算机票和酒店组合。

#### 3. 面试核心回答 / Core Interview Answer

- re-planning 的前提是计划结构化和状态持久化。
- 重规划应尽量局部化，避免整链路重跑。
- 触发条件通常来自失败、外界变化或新约束输入。

一句话总结：好的 re-planning 不是每次都重新想一遍，而是知道该改哪一段。
