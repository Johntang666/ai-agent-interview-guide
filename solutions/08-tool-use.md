# 08 - 工具使用 / Tool Use

本文件收录工具使用章节的参考答案与深度解析。

---

### Q: Tool Use 是扩展 Agent 能力的有效途径。请解释 LLM 是如何学会调用外部 API 或工具的？可以从 Function Calling 的角度解释。

> 🤖 **Claude**

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

> 🤖 **Claude**

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

> 🤖 **Claude**

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

一句话总结：SSE 更像”稳定推流”，WebSocket 更像”实时对话通道”。

---

### Q: 什么是”工具检索”（Tool Retrieval）机制？什么时候需要它？与静态工具列表相比有什么优势？

> 🤖 **Claude**

#### 1. 网络整合回答

当 Agent 可用的工具数量较少时（比如 5-10 个），我们可以把所有工具的 Schema 直接写进 system prompt，让模型从中选择。但在实际生产环境中，工具数量往往远超这个规模——企业级 Agent 平台可能注册了 50、100 甚至上千个工具。如果把所有工具的名称、描述、参数 Schema 和使用示例全部塞进上下文窗口，会带来两个严重问题：一是 token 消耗急剧膨胀，成本和延迟都会飙升；二是模型在过长的工具列表中”挑花眼”，选择准确率明显下降，容易调错工具或幻觉出不存在的工具。

Tool Retrieval（工具检索）机制借鉴了 RAG 的思路来解决这个问题。核心做法是：预先将每个工具的名称、功能描述、参数说明甚至典型使用示例拼接成一段文本，通过 embedding 模型转化为向量，存入向量数据库（如 Pinecone、Milvus、FAISS 等）。当用户发出请求时，先对用户输入做 embedding，然后在工具向量库中检索最相关的 top-k 个工具，只把这 k 个工具的完整 Schema 放入 LLM 上下文，再让模型做工具选择和参数填充。

这个机制的完整流程是：用户输入 → query embedding → 从工具向量库检索 top-k 相关工具 → 将 k 个工具 Schema 注入 system prompt → LLM 进行 function calling → 宿主执行工具 → 结果回传。其中 k 通常取 3-8，既保证覆盖率又不过度消耗上下文。

与静态工具列表相比，Tool Retrieval 的优势包括：大幅节省 token（只传相关工具而非全部）、提高工具选择准确率（模型面对的候选更精准）、支持动态工具注册（新工具只需入库向量即可被发现，无需修改 prompt 模板）、以及更好的可扩展性（工具数量从几十到几千都能支撑）。

#### 2. 结合实际例子

假设我们有一个企业运维 Agent，注册了 200+ 个工具，涵盖监控、部署、日志、数据库、网络等领域。

```python
from openai import OpenAI
import numpy as np

# ---- 1. 工具注册 & 向量化 ----
tools_registry = [
    {
        “name”: “query_prometheus_metrics”,
        “description”: “从 Prometheus 查询指定服务的监控指标，如 CPU、内存、QPS”,
        “parameters”: {“service”: “str”, “metric”: “str”, “time_range”: “str”},
        “example”: “查询 order-service 最近 1 小时的 CPU 使用率”
    },
    {
        “name”: “restart_k8s_deployment”,
        “description”: “重启 Kubernetes 中指定命名空间的 Deployment”,
        “parameters”: {“namespace”: “str”, “deployment”: “str”},
        “example”: “重启 production 命名空间的 payment-service”
    },
    {
        “name”: “search_elk_logs”,
        “description”: “在 ELK 中搜索指定服务的日志，支持关键词和时间范围过滤”,
        “parameters”: {“service”: “str”, “keyword”: “str”, “time_range”: “str”},
        “example”: “搜索 auth-service 最近 30 分钟包含 timeout 的日志”
    },
    # ... 还有 197 个工具
]

client = OpenAI()

# 将每个工具的描述+示例拼接后做 embedding
def embed_tool(tool):
    text = f”{tool['name']}: {tool['description']}. 示例: {tool['example']}”
    response = client.embeddings.create(input=text, model=”text-embedding-3-small”)
    return response.data[0].embedding

tool_vectors = [(tool, embed_tool(tool)) for tool in tools_registry]

# ---- 2. 用户请求进来时，检索相关工具 ----
def retrieve_tools(user_query, top_k=5):
    query_vec = client.embeddings.create(
        input=user_query, model=”text-embedding-3-small”
    ).data[0].embedding

    # 计算余弦相似度，取 top-k
    scored = []
    for tool, vec in tool_vectors:
        similarity = np.dot(query_vec, vec) / (np.linalg.norm(query_vec) * np.linalg.norm(vec))
        scored.append((similarity, tool))
    scored.sort(reverse=True)
    return [tool for _, tool in scored[:top_k]]

# ---- 3. 只把检索到的工具放入 LLM 上下文 ----
user_input = “order-service 的 CPU 飙到 90% 了，帮我看看日志有没有异常”
relevant_tools = retrieve_tools(user_input, top_k=5)
# 检索结果可能是: query_prometheus_metrics, search_elk_logs, restart_k8s_deployment 等
# 而不会包含 “create_jira_ticket”, “update_dns_record” 等无关工具

# 将 relevant_tools 转为 OpenAI function schema 格式，注入到 API 调用中
functions = [
    {
        “name”: t[“name”],
        “description”: t[“description”],
        “parameters”: {
            “type”: “object”,
            “properties”: {k: {“type”: v} for k, v in t[“parameters”].items()}
        }
    }
    for t in relevant_tools
]

response = client.chat.completions.create(
    model=”gpt-4o”,
    messages=[{“role”: “user”, “content”: user_input}],
    tools=[{“type”: “function”, “function”: f} for f in functions]
)
```

这样，无论平台上注册了多少工具，每次 LLM 看到的只有最相关的 5 个左右，既省 token 又准确。

#### 3. 面试核心回答

- **问题背景**：当工具数量多（50+），全部塞进 prompt 会浪费 token 且降低模型选择准确率。
- **核心思路**：借鉴 RAG，将工具描述向量化，根据用户意图动态检索最相关的 top-k 工具子集再注入上下文。
- **标准流程**：用户输入 → embedding → 工具向量库检索 → 取 top-k 工具 Schema → LLM function calling。
- **关键优势**：节省 token、提高选择准确率、支持动态工具注册与发现、可扩展到海量工具。
- **实现要点**：用工具名 + 描述 + 参数说明 + 使用示例拼接后做 embedding 索引，检索时 k 通常取 3-8。

一句话总结：Tool Retrieval 本质是把 RAG 的检索思路用在工具选择上，让 Agent 在海量工具中精准、高效地找到当前最需要的那几个。

---

### Q: 对比 MCP 与 A2A（Agent-to-Agent）协议，它们分别解决什么问题？如何共同构成智能体生态的基础协议？

> 🤖 **Claude**

#### 1. 网络整合回答

MCP（Model Context Protocol）和 A2A（Agent-to-Agent Protocol）是智能体生态中两个互补的基础协议，分别解决不同维度的连接问题。

**MCP** 由 Anthropic 创建，于 2025 年 12 月捐赠给 Linux Foundation 旗下的 AAIF（AI Alliance & Innovation Foundation），是一个标准化 Agent 与外部工具、数据源之间连接的协议。MCP 解决的是”垂直连接”问题——即一个 Agent 如何发现、认证并调用外部能力（数据库查询、API 调用、文件系统操作等）。它定义了工具描述的标准格式、客户端-服务器通信方式、能力协商机制等。截至 2026 年 2 月，MCP 的 Python 和 TypeScript SDK 月下载量已超过 9700 万次，被 Anthropic、OpenAI、Google、Microsoft、Amazon 等主要 AI 厂商采纳，已成为事实上的 Agent-工具连接标准。

**A2A**（Agent-to-Agent Protocol）由 Google 于 2025 年 4 月创建，2025 年 6 月捐赠给 Linux Foundation，专注于标准化 Agent 之间的发现、通信和协作。A2A 解决的是”水平通信”问题——即不同 Agent 之间如何互相发现彼此的能力、协商任务分工、传递中间结果并协作完成复杂目标。A2A 引入了 “Agent Card” 机制，每个 Agent 发布一张描述自身能力、接口和约束的卡片，其他 Agent 可以据此发现并决定是否与之协作。A2A 还支持长时间运行的有状态任务（long-running stateful tasks），允许 Agent 之间进行异步的、多轮次的复杂协作流程。

两者的关系可以用一个精确类比来理解：**MCP 像 USB 接口**，定义了设备（工具/数据源）如何插到电脑（Agent）上；**A2A 像网络协议（TCP/IP）**，定义了多台电脑（多个 Agent）之间如何互相发现和通信。USB 让你的电脑能连接键盘、摄像头、硬盘，而网络协议让多台电脑能协同工作。两者解决的问题不同，但共同构成了完整的连接体系。

在实际应用中，简单的单 Agent + 工具场景只需要 MCP 即可——一个 Agent 通过 MCP 连接各种工具完成任务。但在多 Agent 系统中，就需要 MCP + A2A 的组合：每个 Agent 各自通过 MCP 连接自己擅长的工具集，Agent 之间再通过 A2A 协议互相发现、分配任务、传递结果。这种分层架构既保证了每个 Agent 的工具连接标准化，又实现了 Agent 间的高效协作。

#### 2. 结合实际例子

以一个”企业智能运营平台”为例，展示 MCP 和 A2A 如何协同工作：

```
场景：用户说”分析上季度销售下滑原因，并生成改进方案发给管理层”

整体架构：

┌──────────────────────────────────────────────────────────┐
│                    协调 Agent (Orchestrator)               │
│                                                          │
│  通过 A2A 协议发现并协调以下专业 Agent：                     │
│                                                          │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐      │
│  │ 数据分析Agent │  │ 报告生成Agent │  │ 通知推送Agent │      │
│  │             │  │             │  │             │      │
│  │ (MCP连接)   │  │ (MCP连接)   │  │ (MCP连接)   │      │
│  │ ├─SQL数据库  │  │ ├─模板引擎   │  │ ├─邮件服务   │      │
│  │ ├─BI工具    │  │ ├─图表服务   │  │ ├─企业微信   │      │
│  │ └─Excel文件 │  │ └─PDF生成   │  │ └─日历系统   │      │
│  └─────────────┘  └─────────────┘  └─────────────┘      │
└──────────────────────────────────────────────────────────┘
```

```python
# ---- A2A 层：Agent 发现与任务协作 ----

# 每个 Agent 发布自己的 Agent Card（A2A 标准）
data_analyst_card = {
    “name”: “DataAnalystAgent”,
    “description”: “专业数据分析 Agent，擅长 SQL 查询、统计分析和趋势洞察”,
    “capabilities”: [“sql_analysis”, “trend_detection”, “anomaly_analysis”],
    “endpoint”: “https://agents.company.com/data-analyst”,
    “protocol”: “a2a/1.0”,
    “input_schema”: {“task”: “str”, “data_source”: “str”, “time_range”: “str”},
    “supports_streaming”: True,
    “max_task_duration”: “30m”  # 支持长时间运行的有状态任务
}

report_generator_card = {
    “name”: “ReportGeneratorAgent”,
    “description”: “报告生成 Agent，将分析结果转化为专业报告”,
    “capabilities”: [“report_writing”, “chart_generation”, “pdf_export”],
    “endpoint”: “https://agents.company.com/report-gen”,
    “protocol”: “a2a/1.0”
}

# 协调 Agent 通过 A2A 发现可用 Agent
available_agents = a2a_client.discover(
    capability_required=[“sql_analysis”, “report_writing”, “notification”]
)

# 协调 Agent 通过 A2A 分配任务（支持异步有状态流程）
task_1 = a2a_client.create_task(
    target_agent=”DataAnalystAgent”,
    task={
        “action”: “analyze_sales_decline”,
        “data_source”: “sales_db”,
        “time_range”: “2025-Q4”
    }
)
# task_1 是长时间运行任务，A2A 协议会维护其状态
status = a2a_client.get_task_status(task_1.id)  # → “in_progress”

# ---- MCP 层：Agent 内部连接工具 ----

# 数据分析 Agent 内部通过 MCP 连接具体工具
# 这是 DataAnalystAgent 的内部实现
class DataAnalystAgent:
    def __init__(self):
        # 通过 MCP 连接各种数据工具
        self.sql_tool = MCPClient(“mcp://tools.company.com/sql-executor”)
        self.bi_tool = MCPClient(“mcp://tools.company.com/bi-dashboard”)

    async def handle_task(self, task):
        # 使用 MCP 调用 SQL 工具
        sales_data = await self.sql_tool.call(
            “execute_query”,
            query=”SELECT region, product, revenue FROM sales WHERE quarter='2025-Q4'”
        )
        # 使用 MCP 调用 BI 分析工具
        insights = await self.bi_tool.call(
            “detect_trends”,
            data=sales_data,
            comparison_period=”2025-Q3”
        )
        return {“analysis”: insights, “raw_data”: sales_data}
```

这个例子清楚地展示了两层协议的分工：A2A 负责 Agent 之间的发现和任务协调，MCP 负责每个 Agent 内部与具体工具的连接。

#### 3. 面试核心回答

- **MCP 解决垂直连接**：标准化 Agent 与外部工具/数据源的接入方式，由 Anthropic 创建并捐赠给 Linux Foundation AAIF，截至 2026 年 2 月月下载量超 9700 万，已被五大 AI 厂商（Anthropic、OpenAI、Google、Microsoft、Amazon）采纳。
- **A2A 解决水平通信**：标准化 Agent 之间的发现（Agent Card）、通信和协作（支持长时间有状态任务），由 Google 创建并捐赠给 Linux Foundation。
- **经典类比**：MCP 像 USB 接口（设备连电脑），A2A 像网络协议（电脑连电脑）。
- **互补关系**：单 Agent 场景用 MCP 即可；多 Agent 协作系统需要 MCP + A2A 配合——每个 Agent 用 MCP 连工具，Agent 之间用 A2A 通信。
- **生态意义**：两个协议都捐赠给了 Linux Foundation，走向开放治理，共同构成智能体生态的”连接层”基础设施。

一句话总结：MCP 让 Agent 能连接任何工具，A2A 让 Agent 能协作任何 Agent，两者互补构成了智能体生态从”单体能力”到”群体协作”的完整协议栈。
