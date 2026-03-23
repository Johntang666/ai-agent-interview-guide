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

一句话总结：Agent 本质上是把 LLM 变成“可执行的决策中枢”，而不是只做一次性问答。
