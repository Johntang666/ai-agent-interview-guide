# 08 - 工具使用 / Tool Use

## 题目列表 / Questions

### Function Calling

1. **解释 OpenAI Function Calling 的工作原理。LLM 是如何"选择"工具的？**
   Explain how OpenAI Function Calling works. How does the LLM "choose" tools?

2. **对比不同 LLM 的工具调用方式：OpenAI、Anthropic、Google。**
   Compare tool-calling approaches across LLMs: OpenAI, Anthropic, Google.

3. **如何设计工具的 Schema 使 LLM 能更准确地调用？**
   How do you design tool schemas so the LLM calls them more accurately?

4. **Parallel Function Calling 的实现原理和应用场景。**
   Implementation principles and use cases of Parallel Function Calling.

### 工具编排 / Tool Orchestration

5. **如何设计一个可扩展的工具注册中心（Tool Registry）？**
   How do you design a scalable Tool Registry?

6. **当 Agent 可用工具超过 50 个时，如何优化工具选择？**
   How do you optimize tool selection when an Agent has 50+ available tools?

7. **如何实现工具调用的链式组合（Tool Chaining）？**
   How do you implement tool chaining?

8. **工具调用失败时的错误处理和降级策略。**
   Error handling and fallback strategies when tool calls fail.

### MCP 与生态 / MCP & Ecosystem

9. **什么是 MCP（Model Context Protocol）？它解决了什么问题？**
   What is MCP (Model Context Protocol)? What problem does it solve?

10. **如何设计和实现一个 MCP Server？**
    How do you design and implement an MCP Server?

11. **对比 MCP 与传统 API 集成方式的优劣。**
    Compare pros and cons of MCP vs. traditional API integration.

12. **Agent 如何安全地执行代码（Code Execution / Sandboxing）？**
    How does an Agent safely execute code (Code Execution / Sandboxing)?
