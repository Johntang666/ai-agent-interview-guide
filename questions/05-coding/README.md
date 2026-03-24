# 05 - 编码实战 / Coding

## 题目列表 / Questions

### 手写 Agent / Implement from Scratch

1. **用 Python 实现一个最简单的 ReAct Agent（不使用任何框架）。**
   Implement a minimal ReAct Agent in Python without any framework.

2. **实现一个支持 Function Calling 的 Agent 循环。**
   Implement an Agent loop that supports Function Calling.

3. **编写一个具有短期记忆（对话历史）和长期记忆（向量存储）的 Agent。**
   Write an Agent with short-term memory (conversation history) and long-term memory (vector store).

4. **实现一个 Plan-and-Execute Agent，先制定计划再逐步执行。**
   Implement a Plan-and-Execute Agent that creates a plan then executes step by step.

### Prompt 工程 / Prompt Engineering

5. **编写一个让 LLM 稳定输出结构化 JSON 的 System Prompt。**
   Write a System Prompt that makes an LLM reliably output structured JSON.

6. **设计一个 Router Agent 的 Prompt，能根据用户意图路由到不同的子 Agent。**
   Design a Router Agent Prompt that routes to different sub-Agents based on user intent.

7. **如何通过 Prompt 实现 few-shot 工具选择？**
   How do you implement few-shot tool selection through Prompts?

### 工具与集成 / Tool & Integration

8. **实现一个通用的工具注册和调用机制。**
   Implement a generic tool registration and invocation mechanism.

9. **编写一个带有错误重试和降级策略的工具调用包装器。**
   Write a tool-calling wrapper with error retry and fallback strategies.

10. **实现一个 Agent 的流式输出（Streaming），同时支持中间步骤展示。**
    Implement Agent streaming output with intermediate step display.

### 测试 / Testing

11. **如何为 Agent 编写单元测试？Mock LLM 调用的最佳实践是什么？**
    How do you write unit tests for Agents? What are best practices for mocking LLM calls?

12. **实现一个简单的 Agent 评估框架，支持多维度打分。**
    Implement a simple Agent evaluation framework with multi-dimensional scoring.

### 补充题目 / Additional Questions

13. **有微调过 Agent 能力吗？相关数据集如何收集与构建？**
    Have you fine-tuned Agent capabilities? How do you collect and build the relevant datasets?

14. **用 Python 实现一个简单的记忆检查点（Memory Checkpointer），支持保存和恢复 Agent 状态。**

15. **如何设计一个对 LLM 幻觉参数名或参数值具有鲁棒性的工具调用函数？**
