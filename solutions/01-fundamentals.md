# 01 - 基础概念 / Fundamentals

本文件收录基础概念章节的参考答案与深度解析。

---

### Q: 如何定义一个 Agent？它通常由哪些核心组件构成？

#### 1. 网络整合回答 / Comprehensive Answer

Agent 可以理解为一个以目标为驱动、能够感知环境、进行推理、采取行动并根据反馈持续调整策略的系统。它不只是“会回答问题”的大模型，而是一个“能闭环完成任务”的执行体。  
一个典型的 LLM Agent 通常包含这些核心组件：目标与约束（Goal / Policy）、推理与规划模块（Reasoning / Planning）、记忆模块（Short-term / Long-term Memory）、工具调用层（Tool Use）、执行器（Executor）、环境接口（Environment Interface）、状态与观测（State / Observability）以及安全控制（Guardrails / Permission Control）。  
面试里最好强调一点：Agent 的关键不在于用了多少模型，而在于是否具备“感知-决策-执行-反馈”闭环，以及是否能在不确定环境中持续推进任务。

#### 2. 结合实际例子 / Practical Example

以“自动报销助手”为例：

- 用户输入报销需求，这是目标和约束输入。
- Agent 先读取公司报销规则和历史记录，这是感知与记忆。
- 然后它规划步骤：识别票据、校验金额、调用财务 API 提交。
- 如果调用 API 失败，会根据错误码重试或转人工，这属于执行和反馈闭环。
- 最后系统记录整条 trace，方便审计和回放，这属于可观测性与安全治理。

#### 3. 面试核心回答 / Core Interview Answer

- Agent 是目标驱动的，不只是生成文本，而是能完成任务闭环。
- 核心组件通常包括规划、记忆、工具、执行、环境接口和安全控制。
- 判断是不是 Agent，关键看它是否具备持续感知和行动能力。

一句话总结：Agent 本质上是把 LLM 变成”可执行的决策中枢”，而不是只做一次性问答。

> 🤖 **Claude**

#### 1. 网络整合回答 / Comprehensive Answer

Agent（智能体）在 AI 领域的定义是：一个能够自主感知外部环境、基于内部状态进行推理决策、并通过动作影响环境以达成特定目标的计算实体。与传统 Chatbot 的本质区别在于，Chatbot 只做”一问一答”的文本生成，而 Agent 具备持续运行、主动规划、工具调用和自我纠错的能力。

从学术视角看，Agent 的定义可以追溯到 Russell & Norvig 的经典 AI 教材：Agent 是任何通过传感器（sensors）感知环境并通过执行器（actuators）作用于环境的实体。在 LLM 时代，这个定义被具象化为以大语言模型为推理核心的系统。

核心组件通常包括六大模块：
1. **感知层（Perception）**：接收用户输入、环境状态、工具返回值等多模态信息
2. **推理与规划引擎（Reasoning & Planning）**：LLM 作为大脑，负责理解意图、拆解任务、制定执行计划
3. **记忆系统（Memory）**：短期记忆（当前会话上下文）+ 长期记忆（向量数据库、结构化存储）
4. **工具接口（Tool Use）**：通过 Function Calling 等机制调用外部 API、数据库、代码执行器
5. **执行与行动层（Action）**：将推理结果转化为具体操作，如发送请求、修改文件、调用服务
6. **反馈与自省机制（Feedback & Reflection）**：根据执行结果评估成败，决定是否重试、修正策略或请求人工介入

这些组件通过一个循环（Agent Loop）协调运作：感知 → 推理 → 行动 → 观察结果 → 再推理，直到任务完成或达到终止条件。

#### 2. 结合实际例子 / Practical Example

以一个简单的 ReAct Agent 骨架为例，展示核心组件如何协作：

```python
from openai import OpenAI

client = OpenAI()

# 工具定义（Tool Use 组件）
tools = [
    {
        “type”: “function”,
        “function”: {
            “name”: “search_knowledge_base”,
            “description”: “搜索内部知识库获取相关文档”,
            “parameters”: {
                “type”: “object”,
                “properties”: {
                    “query”: {“type”: “string”, “description”: “搜索关键词”}
                },
                “required”: [“query”]
            }
        }
    }
]

# 记忆系统（Memory 组件）
conversation_history = []  # 短期记忆
system_prompt = “你是一个客服Agent，根据知识库回答用户问题。”

# Agent Loop（感知→推理→行动→反馈 循环）
def agent_loop(user_input: str, max_steps: int = 5):
    conversation_history.append({“role”: “user”, “content”: user_input})

    for step in range(max_steps):
        # 推理：LLM 决定下一步行动
        response = client.chat.completions.create(
            model=”claude-sonnet-4-6”,
            messages=[{“role”: “system”, “content”: system_prompt}]
                     + conversation_history,
            tools=tools,
            tool_choice=”auto”
        )

        msg = response.choices[0].message

        # 行动：如果需要调用工具
        if msg.tool_calls:
            for tool_call in msg.tool_calls:
                result = execute_tool(tool_call)  # 执行层
                conversation_history.append({
                    “role”: “tool”,
                    “content”: result,
                    “tool_call_id”: tool_call.id
                })
            continue  # 回到循环顶部，再次推理

        # 反馈：生成最终回答
        conversation_history.append({“role”: “assistant”, “content”: msg.content})
        return msg.content

    return “抱歉，我无法在有限步骤内完成任务，转接人工客服。”
```

这个例子清晰展示了 Agent 的核心循环：用户输入（感知）→ LLM 推理 → 工具调用（行动）→ 工具结果（反馈）→ 继续推理，直到生成最终回答。

#### 3. 面试核心回答 / Core Interview Answer

- Agent 是具备”感知-推理-行动-反馈”闭环能力的自主系统，不是简单的一次性文本生成。
- 六大核心组件：感知层、推理引擎（LLM）、记忆系统、工具接口、执行层、反馈机制。
- Agent 与 Chatbot 的本质区别在于自主性：Agent 能主动规划、调用工具、自我纠错。
- Agent Loop 是核心架构模式：循环执行直到任务完成或触发终止条件。
- 生产级 Agent 还需要可观测性、权限控制、成本管理等工程化组件。

一句话总结：Agent 是以 LLM 为大脑、以工具为双手、以记忆为经验、通过持续循环自主完成任务的智能系统。
