# 08 - 工具使用 / Tool Use

本文件收录工具使用章节的参考答案与深度解析。

---

### Q: Tool Use 是扩展 Agent 能力的有效途径。请解释 LLM 是如何学会调用外部 API 或工具的？可以从 Function Calling 的角度解释。

#### 1. 网络整合回答 / Comprehensive Answer

LLM 本身不会真正“执行工具”，它做的是在给定工具描述、参数 Schema 和上下文的前提下，预测“此刻是否应该调用某个工具，以及调用时参数应该长什么样”。Function Calling 机制把这个过程结构化了：开发者先把工具的名称、描述和参数定义成 schema，模型在生成时如果判断需要调用工具，就输出一个结构化的 tool call，而不是直接给自然语言答案。  
模型之所以“学会”调用工具，主要依赖两部分：一是预训练后再对齐阶段接触过大量函数调用格式和指令跟随数据，二是推理时受到 schema 和系统提示的约束。真正执行工具的是宿主程序，执行完后再把结果喂回模型，模型据此继续下一步推理。

#### 2. 结合实际例子 / Practical Example

例如天气助手有一个 `get_weather(city, date)` 工具：

- 用户问“明天上海会下雨吗？”
- 模型根据工具描述判断需要查天气，而不是硬编答案。
- 它输出结构化调用：`{"city":"Shanghai","date":"tomorrow"}`
- 宿主程序执行 API 请求，再把结果回传模型。
- 模型基于真实返回值组织最终回答。

#### 3. 面试核心回答 / Core Interview Answer

- LLM 负责决定“要不要调工具”和“参数怎么填”。
- 宿主程序负责真正执行工具并把结果回灌给模型。
- Function Calling 的本质是把工具使用从自由文本变成结构化决策。

一句话总结：Tool Use 不是模型真的会写后端逻辑，而是模型会在合适时机输出可执行的调用意图。

---

### Q: RAG，Functioncall，MCP 了解么，简单说下原理。

#### 1. 网络整合回答 / Comprehensive Answer

这三个概念解决的是不同层级的问题。RAG 解决“模型缺少外部知识”的问题，本质是检索外部内容并把相关上下文补给模型；Function Calling 解决“模型需要调用具体能力”的问题，本质是让模型输出结构化工具调用；MCP 解决“模型和外部工具 / 资源如何统一接入”的问题，本质是定义一个标准协议，让模型侧客户端能发现、调用并复用外部能力。  
可以把它们看成三层：RAG 偏知识获取，Function Calling 偏能力触发，MCP 偏协议标准化。它们可以组合使用，比如 Agent 通过 MCP 发现一个检索服务，再通过 Function Calling 触发该服务，最终把返回文档作为 RAG 上下文送入模型。

#### 2. 结合实际例子 / Practical Example

做企业知识助理时：

- RAG 用来从知识库找相关文档片段。
- Function Calling 用来调用工单系统、CRM、审批接口等具体 API。
- MCP 用来把这些检索源和工具统一封装成可发现、可复用的资源入口。

#### 3. 面试核心回答 / Core Interview Answer

- RAG 解决“知道什么”，Function Calling 解决“做什么”，MCP 解决“怎么标准化接入”。
- 三者不是替代关系，而是经常叠加使用。
- 面试里要讲清楚它们所在层级不同。

一句话总结：RAG 补知识，Function Calling 补动作，MCP 补连接规范。

---

### Q: WebSocket 和 SSE 区别是什么？

#### 1. 网络整合回答 / Comprehensive Answer

SSE（Server-Sent Events）是服务器单向推送到客户端的长连接机制，基于 HTTP，浏览器支持简单，特别适合流式文本输出。WebSocket 则是全双工通信，客户端和服务器都可以随时发消息，更适合需要双向、低延迟、频繁交互的场景。  
如果是大模型流式输出、日志推送、进度播报，SSE 往往更轻量；如果是实时协作、多人在线编辑、游戏状态同步、浏览器端持续上传事件，WebSocket 更合适。工程上还要考虑代理兼容性、断线重连、消息顺序和负载均衡策略。

#### 2. 结合实际例子 / Practical Example

- Chat UI 展示模型 token streaming，用 SSE 就很自然。
- Browser agent 持续上传点击、截图状态，同时接收控制命令，更适合 WebSocket。
- 如果只是后端到前端的单向事件流，优先 SSE，复杂度更低。

#### 3. 面试核心回答 / Core Interview Answer

- SSE 是单向推送，WebSocket 是双向实时通信。
- 模型流式输出常用 SSE，复杂交互控制更适合 WebSocket。
- 选型时要同时看交互模式、网络基础设施和实现复杂度。

一句话总结：SSE 更像“稳定推流”，WebSocket 更像“实时对话通道”。
