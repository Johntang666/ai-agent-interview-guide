# 04 - 系统设计 / System Design

## 题目列表 / Questions

### 架构设计 / Architecture Design

1. **设计一个生产级的客服 Agent 系统，需要处理多轮对话、工具调用和人工接管。**
   Design a production-grade customer service Agent system handling multi-turn conversations, tool calling, and human handoff.

2. **设计一个代码生成 Agent，能理解需求、生成代码、运行测试并自动修复 bug。**
   Design a code generation Agent that understands requirements, generates code, runs tests, and auto-fixes bugs.

3. **如何设计一个可扩展的 Agent 网关（Agent Gateway）？**
   How would you design a scalable Agent Gateway?

4. **设计一个多租户 Agent 平台的架构，支持不同客户自定义 Agent 行为。**
   Design the architecture of a multi-tenant Agent platform supporting customizable Agent behavior per client.

### 工程挑战 / Engineering Challenges

5. **如何处理 Agent 系统中的长时间运行任务（Long-running tasks）？**
   How do you handle long-running tasks in Agent systems?

6. **Agent 系统的状态管理如何设计？如何处理中断恢复？**
   How do you design state management in Agent systems? How do you handle interruption recovery?

7. **如何设计 Agent 的限流和成本控制机制？**
   How do you design rate limiting and cost control for Agents?

8. **如何实现 Agent 调用链的可追溯性（Traceability）？**
   How do you achieve traceability of Agent call chains?

9. **设计一个 Agent 的灰度发布和 A/B 测试方案。**
   Design a canary release and A/B testing strategy for Agents.

### 规模化 / Scaling

10. **当 Agent 需要处理 10,000 并发请求时，系统架构应如何设计？**
    How should you architect the system when the Agent needs to handle 10,000 concurrent requests?

11. **如何设计 Agent 的缓存策略以降低 LLM 调用成本？**
    How do you design caching strategies for Agents to reduce LLM call costs?

12. **如何实现跨地域的 Agent 部署？需要考虑哪些因素？**
    How do you implement cross-region Agent deployment? What factors need consideration?

13. **如何设计 Agent 的可观测性系统？解释 Tracing 和 Spans 在 Agent 调试中的作用。**

14. **如何设计一个 Agent 来处理超长时间跨度的任务（如"研究一家公司的完整历史并撰写报告"）？如何防止 Agent 迷失或卡住？**

15. **Agent 系统的工程化挑战：为什么说 AI 只完成了 30% 的工作，剩余 70% 是工具工程？如何设计可靠的工具工程架构？**
