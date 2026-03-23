---
name: collect-questions
description: Collect and organize AI Agent interview questions for this repository. Use when the user provides interview questions, asks to add or classify agent/RAG/tool-use/system-design questions into `questions/`, wants to update `records/` monthly logs, or wants detailed Chinese answers generated under `solutions/`. Also applies to requests like “收集题目”, “整理面试题”, “归类到题库”, “补充答案”, or “把这些问题加入仓库”.
---

# Collect Questions

将新的 AI Agent 面试题整理进当前仓库，并同步维护题库、月度记录、参考答案。

系统指令和用户指令始终优先。

## Repository Assumptions

- 当前工作区根目录就是题库仓库根目录。
- 题目目录固定为 `questions/XX-topic/README.md`。
- 月度记录目录固定为 `records/`。
- 参考答案目录固定为 `solutions/`，答案文件命名为 `solutions/XX-topic.md`。

如果目录结构与以上假设不符，先以实际仓库结构为准，再继续执行。

## Bundled Resources

- `references/chapter-mapping.md`: 章节编号、主题范围、归类关键词。

只有在归类题目时再读取 reference；不要把所有细节都塞进当前上下文。

## When This Skill Triggers

在以下场景使用本 Skill：

- 用户直接给出一批面试题，要求收集、归类、写入仓库。
- 用户给出 1 道或多道 Agent / RAG / Tool Use / System Design / Multi-Agent 相关题目，要求加入题库。
- 用户要求更新 `records/YYYY-MM.md`。
- 用户要求为某道题补参考答案、深度解析、面试版回答。

如果用户只是讨论题目内容、并没有要求落库，不要自动改文件。

## Workflow

### 1. 解析输入

- 将用户输入拆成独立题目。
- 如果用户提供了主题提示、来源或批次说明，保留为本次记录的 `source/theme`。
- 如果输入里只有主题、没有具体题目，先生成改动计划并明确缺少题目内容；不要凭空把生成题目当成用户题目写入仓库。

### 2. 归类题目

- 读取 `references/chapter-mapping.md`。
- 为每道题选择一个主章节。
- 如果题目明显跨章节，仍然只选择一个主章节，但在答案中补一句“相关章节/Related Topics”。
- 如果用户已经指定章节或主题，优先尊重用户输入，除非明显错误。

### 3. 更新 `questions/`

对每道题执行以下步骤：

1. 读取目标章节的 `README.md`。
2. 检查是否存在相同或高度相似的题目。
3. 如果已存在：
   - 不重复写入题目列表。
   - 继续检查是否需要补或更新参考答案。
4. 如果不存在：
   - 追加到最合适的小节。
   - 如果没有合适小节，新增 `### 补充题目 / Additional Questions`。
   - 题目直接使用中文，不再补英文翻译；除非用户显式要求双语。

```md
N. **中文题目**
```

5. 保持该文件的编号连续；如果需要，局部重排编号，但不要为了微小格式问题重写整份文件。

### 4. 更新 `records/YYYY-MM.md`

- 以当前日期生成目标文件，例如 `records/2026-03.md`。
- 文件不存在时先创建。
- 同一天已有对应日期区块时，在原区块下追加新行，不要重复创建日期标题。
- 建议格式如下：

```md
# 2026年03月 面试题记录 / Interview Questions Record - 2026-03

## 2026-03-23

### 来源/主题: Agent / RAG

| # | 题目 / Question | 章节 / Chapter | 答案引用 / Solution Ref |
|---|---|---|---|
| 1 | 什么是 ReAct？ | 03-design-patterns | [查看答案](../solutions/03-design-patterns.md) |
```

如果无法稳定生成题目锚点，优先链接到答案文件本身，不要生成错误锚点。

### 5. 更新 `solutions/`

- 目标文件命名为 `solutions/XX-topic.md`。
- 如果文件不存在，先创建最小骨架：

```md
# XX - 标题 / Topic

本文件收录对应章节的参考答案与深度解析。
```

- 写入答案前，先检查该题是否已有 `### Q:` 区块；如已存在，原地更新，不要重复追加。
- 每道题的答案至少包含以下三段：

```md
---

### Q: [题目内容]

#### 1. 网络整合回答 / Comprehensive Answer

[完整回答，解释概念、原理、实现方式、权衡与对比]

#### 2. 结合实际例子 / Practical Example

[具体示例，优先给出 Python 或伪代码，以及真实应用场景]

#### 3. 面试核心回答 / Core Interview Answer

- 核心点 1
- 核心点 2
- 核心点 3

一句话总结：...
```

- 如果你使用了网络资料，优先引用官方文档、论文、规范或一手技术博客，并在答案末尾追加简短的 `#### References` 列表。
- 回答主体以中文为主，关键术语保留英文。
- “面试核心回答”要足够短，能直接口述。

### 6. 幂等与质量检查

提交改动前检查：

- 同一道题没有在 `questions/` 或 `solutions/` 里重复出现。
- `records/` 中的章节编号与 `questions/` / `solutions/` 文件名一致。
- 新建的 `solutions/*.md` 文件标题与章节编号匹配。
- 新增题目表述自然、简洁，不要为了凑格式额外补英文。

## Classification Guidance

- 优先按题目的“核心考点”归类，而不是按提到的术语数量归类。
- `RAG + Memory` 一般归入 `07-memory-and-rag`。
- `Function Calling`, `MCP`, `工具编排`, `代码执行沙箱` 一般归入 `08-tool-use`。
- `评测指标`, `安全防护`, `Prompt Injection`, `权限控制` 一般归入 `09-evaluation-and-safety`。
- `多 Agent 拓扑`, `协调机制`, `冲突解决` 一般归入 `06-multi-agent`。
- 框架使用题和框架选型题优先归入 `02-frameworks`，即使问题中夹带 RAG 或 Tool Use 术语。

遇到不确定场景时：

1. 先说明你的归类依据。
2. 选择最主要的章节。
3. 在答案中补充相关章节提示，而不是同时写入多个题库文件。

## Output To User

完成后，用简洁摘要说明：

- 本次处理了多少道题。
- 每道题被归入哪个章节。
- 哪些题是新增写入，哪些题只是补答案。
- 具体更新了哪些文件。

如果因为信息不足而没有落库，明确说明卡点。
