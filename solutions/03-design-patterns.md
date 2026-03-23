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

一句话总结：LLM 的规划能力来自“显式分解任务并在执行中持续更新计划”，而不是一次生成一个完美答案。
