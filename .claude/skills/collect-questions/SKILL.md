---
name: collect-questions
description: "收集面试题目并自动归类到对应章节、生成月度记录和详细答案。当用户发送面试题目、面试相关问题、Agent/RAG/LLM相关题目时触发。关键词：面试题、interview question、题目、收集、整理、归类"
user-invocable: true
argument-hint: "[题目内容或主题提示]"
allowed-tools: Read, Write, Edit, Grep, Glob, Bash, WebSearch, WebFetch, Agent
effort: high
---

# 面试题收集与整理 Skill

你是 AI Agent 面试题库的智能整理助手。当用户发送面试题目（可能带有主题提示，也可能只有裸题目）时，你需要完成以下完整流程。

## 输入内容

$ARGUMENTS

## 处理流程

### 第一步：解析与分类

仔细分析每道题目的内容，将其归类到以下章节之一：

| 章节目录 | 关键词/主题 |
|---------|------------|
| `questions/01-fundamentals/` | Agent 定义、核心组件、感知-推理-行动循环、自主性、Agentic AI vs Generative AI、LLM 作为大脑 |
| `questions/02-frameworks/` | LangChain、LangGraph、AutoGen、CrewAI、Dify、LlamaIndex、框架对比、框架选型 |
| `questions/03-design-patterns/` | ReAct、Plan-and-Execute、Plan-and-Solve、Reflexion、Reflection、CoT、ToT、GoT、LATS、Router、Supervisor |
| `questions/04-system-design/` | 生产级架构、系统设计、可扩展性、状态管理、限流、成本控制、灰度发布、并发 |
| `questions/05-coding/` | 手写实现、代码实战、Prompt 工程、单元测试、微调、数据集收集 |
| `questions/06-multi-agent/` | 多智能体、多 Agent 协作、通信机制、协调、拓扑结构、冲突解决 |
| `questions/07-memory-and-rag/` | RAG、记忆系统、短期记忆、长期记忆、向量数据库、Embedding、检索、切块、GraphRAG、知识图谱、Transformer |
| `questions/08-tool-use/` | Function Calling、工具使用、MCP、API 调用、工具编排、代码执行、沙箱、WebSocket、SSE |
| `questions/09-evaluation-and-safety/` | 评估、安全、对齐、Prompt Injection、幻觉、可控性、权限控制、审计 |
| `questions/10-real-world-cases/` | 真实案例、落地挑战、环境交互（机器人/游戏）、部署挑战、行业应用 |

**分类规则：**
- 如果用户提供了主题提示（如 "Agent"、"RAG"），优先参考提示
- 一道题可能跨多个章节，选择最核心匹配的章节
- 如果题目明确涉及多个独立主题（如 "RAG, Function Call, MCP 了解么"），根据主要内容归类，但可在相关章节添加交叉引用

### 第二步：写入章节文件

对于每道题目：

1. 先用 Read 工具读取目标章节的 `README.md` 文件
2. 检查是否已存在相同或高度相似的题目（避免重复）
3. 如果是新题目，追加到对应章节 README.md 的合适分类下
4. 题目统一使用中文，不需要英文翻译：

```markdown
N. **中文题目**
```

### 第三步：月度记录

1. 根据当前日期，创建或更新 `records/YYYY-MM.md` 文件（如 `records/2026-03.md`）
2. 月度记录格式如下：

```markdown
# YYYY年MM月 面试题记录 / Interview Questions Record - YYYY-MM

## YYYY-MM-DD

### 来源/主题: [用户提供的主题提示或自动识别的主题]

| # | 题目 / Question | 章节 / Chapter | 答案引用 / Solution Ref |
|---|----------------|---------------|----------------------|
| 1 | 题目内容 | 01-fundamentals | [查看答案](../solutions/01-fundamentals.md#题目锚点) |
| 2 | 题目内容 | 07-memory-and-rag | [查看答案](../solutions/07-memory-and-rag.md#题目锚点) |
```

### 第四步：查重与生成答案（Claude 标识机制）

这是最核心的步骤。

**⚠️ 最重要的原则：没有 `> 🤖 **Claude**` 标记的回答都是别人写的，不是 Claude 的！Claude 绝不能把别人的回答加个标记就当成自己的回答。Claude 必须自己重新生成全新的三段式回答内容。**

**`> 🤖 **Claude**` 标记位于题目下方、Claude 回答上方，意思是"以下回答由 Claude 生成"。标记后面必须紧跟 Claude 自己写的全新回答。**

**不删除任何已有内容，Claude 的回答追加在已有内容下方。**

**处理流程：**

1. 用 Read 工具读取对应的 `solutions/XX-topic.md` 文件
2. 搜索该题目区块内是否有 `🤖 **Claude**` 标记
3. **判断逻辑：**
   - 找到 `🤖 **Claude**` 标记，且标记下方有 Claude 自己写的完整三段式回答（含 `#### 1.`、`#### 2.`、`#### 3.` 三个小节）→ **跳过**
   - 找到 `🤖 **Claude**` 标记，但标记下方没有三段式回答（说明上次只加了标记没写内容）→ 在标记下方**生成全新的完整回答**
   - 找到该题目，有回答但没有 Claude 标记 → **那些回答是别人的，不是 Claude 的**。保留原有内容不动，在其下方追加 `> 🤖 **Claude**` 标记 + Claude 自己生成的全新三段式回答
   - 没有找到该题目 → 新增题目 + `> 🤖 **Claude**` 标记 + Claude 自己生成的全新三段式回答

**⚠️ 再次强调："有回答但没有 Claude 标记" ≠ "Claude 已经回答过了"。没有标记就代表 Claude 没回答过，必须自己重新写一份，不能偷懒只加标记！**

**答案文件命名**：`solutions/XX-topic.md`（如 `solutions/01-fundamentals.md`）

**每道题的完整答案格式如下**：

```markdown
---

### Q: [题目内容]

[这里可能有其他来源的已有回答，保持原样不动]

> 🤖 **Claude**

#### 1. 网络整合回答

[Claude 撰写的实际回答，至少 200 字]
[概念定义、原理解释、技术细节、优缺点分析等]

#### 2. 结合实际例子

[具体真实例子，含代码片段（Python 优先）、架构描述、应用场景]
[至少一个可运行的代码示例或详细技术方案]

#### 3. 面试核心回答

[精简面试要点，3-5 个核心要点 + 一句话总结]
```

**结构说明：**
```
### Q: 题目          ← 题目
[其他来源回答...]     ← 已有回答（保留不动）
> 🤖 **Claude**     ← 标记（题目下方，回答上方）
#### 1. 网络整合回答  ← Claude 的回答开始
#### 2. 结合实际例子
#### 3. 面试核心回答  ← Claude 的回答结束
---                  ← 下一道题的分隔线
```

**模板中的 `[...]` 是说明文字，必须替换为真实回答内容！**

**答案生成要求**：
- **绝不删除已有回答**，Claude 的回答追加在已有内容之后、下一道题的 `---` 之前
- 使用 WebSearch 工具搜索最新、最权威的资料来充实回答
- 第 1 部分「网络整合回答」：全面深入，至少 200 字
- 第 2 部分「结合实际例子」：必须有具体代码或技术方案
- 第 3 部分「面试核心回答」：3-5 个要点 + 一句话总结，精简有力
- 题目和答案统一使用中文，关键术语保留英文原文（如 ReAct、Function Calling、RAG 等）

## 输出总结

完成所有步骤后，输出一个简洁的总结表格：

```
✅ 处理完成！本次共收录 N 道题目：

| # | 题目（简称）| 归类章节 | 答案状态 |
|---|-----------|---------|---------|
| 1 | xxx | 01-fundamentals | ✅ 已生成 |
| 2 | xxx | 07-memory-and-rag | ⏭️ 已有 Claude 回答 |
| 3 | xxx | 03-design-patterns | ➕ 已追加 Claude 回答 |
| 4 | xxx | 08-tool-use | 🔧 已补写回答 |
```

答案状态说明：
- ✅ 已生成 — 新题目，已写入标记 + 完整三段式回答
- ⏭️ 已有 Claude 回答 — 该题已有标记且下方有完整回答，跳过
- ➕ 已追加 — 该题有旧回答但无 Claude 标记，已追加标记 + 完整回答（原回答保留）
- 🔧 已补写 — 该题有 Claude 标记但下方回答缺失，已补写完整回答

## 注意事项

- **标记位置：题目下方、回答上方**，表示"以下由 Claude 回答"
- **标记不等于回答**，标记下方必须有完整三段式回答内容
- **绝对不能删除任何已有回答**，只能在后面追加
- 如果题目已存在于章节文件中，跳过章节写入，但仍检查/生成答案
- 答案中的代码示例优先使用 Python
- 月度记录文件是累加的，不要覆盖已有记录
- 保持所有文件的格式一致性
- 如果无法确定分类，优先归入最相关的章节，并在答案中说明与其他章节的关联
