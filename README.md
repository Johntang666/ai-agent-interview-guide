# AI Agent 面试指南

一份面向 AI Agent 岗位准备的系统化面试题库，覆盖基础概念、框架、设计模式、系统设计、编码实战、多智能体、RAG、工具使用、安全评估与真实案例。

> 文档由 `Opus4.6 Thinking(High)` 和 `gpt5.4(xhigh)` 共同创作。

## 项目简介

本仓库用于整理 AI Agent 相关岗位的面试题、参考答案和月度记录，适合：

- 准备 AI / LLM Agent 方向面试的候选人
- 希望系统化整理题库的面试官或团队
- 对 Agent 技术栈感兴趣的开发者

## 仓库结构

```text
ai-agent-interview-guide/
│
├── README.md                          # 项目说明
├── CONTRIBUTING.md                    # 贡献指南
│
├── questions/                         # 面试题目（按主题分类）
│   ├── 01-fundamentals/              # 基础概念
│   ├── 02-frameworks/                # 主流框架
│   ├── 03-design-patterns/           # 设计模式
│   ├── 04-system-design/             # 系统设计
│   ├── 05-coding/                    # 编码实战
│   ├── 06-multi-agent/               # 多智能体协作
│   ├── 07-memory-and-rag/            # 记忆与 RAG
│   ├── 08-tool-use/                  # 工具使用
│   ├── 09-evaluation-and-safety/     # 评估与安全
│   └── 10-real-world-cases/          # 真实案例
│
├── solutions/                         # 参考答案与深度解析
├── records/                           # 按月归档的题目记录
├── resources/                         # 学习资源
├── interview-experiences/             # 公开面经与高频考点整理
├── docs/                              # 补充文档
└── collect-questions/                 # Codex Skill：整理题目到仓库
```

## 内容模块

| 模块 | 说明 | 难度 |
|------|------|------|
| 01-基础概念 | Agent 定义、感知-推理-行动循环、与 Chatbot 区别 | ⭐ |
| 02-主流框架 | LangChain / LangGraph / LlamaIndex / AutoGen / Dify 等对比 | ⭐⭐ |
| 03-设计模式 | ReAct / Plan-and-Solve / Reflexion / ToT 等模式 | ⭐⭐⭐ |
| 04-系统设计 | 生产级 Agent 系统架构设计 | ⭐⭐⭐⭐ |
| 05-编码实战 | 手写 Agent、Tool Loop、评测与微调数据集 | ⭐⭐⭐ |
| 06-多智能体 | 通信协议、协调机制、任务分配 | ⭐⭐⭐⭐ |
| 07-记忆与 RAG | Memory、RAG 管线、Embedding、GraphRAG | ⭐⭐⭐ |
| 08-工具使用 | Function Calling、MCP、工具编排、实时通信 | ⭐⭐⭐ |
| 09-评估与安全 | Benchmark、对齐、安全防护、审计 | ⭐⭐⭐⭐ |
| 10-真实案例 | 工业级 Agent 场景与落地挑战 | ⭐⭐⭐⭐⭐ |
| 面经模块 | 公开面经链接、真实问法与复习重点整理 | ⭐⭐⭐ |

## 如何使用

1. 按 `01 → 10` 顺序系统学习。
2. 根据薄弱模块针对性练习题目。
3. 结合 `solutions/` 练习面试回答。
4. 通过 `records/` 查看每次新增题目的批次记录。
5. 结合 `interview-experiences/` 了解真实面试问法和项目追问角度。

## 在 Codex 中使用 `collect-questions` Skill

`collect-questions` 是给 Codex 用的 skill，用来把一批新的 Agent / RAG 面试题自动归类到本仓库，并同步更新：

- `questions/XX-topic/README.md`
- `records/YYYY-MM.md`
- `solutions/XX-topic.md`

### 1. 需要复制哪些内容

不要只复制 `SKILL.md`，要把整个 `collect-questions` 目录完整复制过去，因为它还依赖：

- `collect-questions/SKILL.md`
- `collect-questions/agents/openai.yaml`
- `collect-questions/references/chapter-mapping.md`

### 2. 复制到哪里

优先使用 Codex 的 skill 目录：

- 如果设置了 `CODEX_HOME`：复制到 `$CODEX_HOME/skills/collect-questions`
- 如果没有设置：Windows 默认可放到 `%USERPROFILE%\\.codex\\skills\\collect-questions`

在 Windows PowerShell 下，可以直接执行：

```powershell
$targetBase = if ($env:CODEX_HOME) {
    Join-Path $env:CODEX_HOME 'skills'
} else {
    Join-Path $env:USERPROFILE '.codex\skills'
}

New-Item -ItemType Directory -Force -Path $targetBase | Out-Null
Copy-Item -Recurse -Force '.\collect-questions' (Join-Path $targetBase 'collect-questions')
```

如果你想手动复制，最终目录结构应当类似：

```text
C:\Users\你的用户名\.codex\skills\collect-questions\
├── SKILL.md
├── agents\
│   └── openai.yaml
└── references\
    └── chapter-mapping.md
```

### 3. 复制后做什么

1. 重新打开 Codex，或者新开一个会话，让它重新加载 skills。
2. 把当前工作目录切到本仓库根目录。
3. 在对话里明确提到要使用 `collect-questions`。

### 4. 怎么触发这个 Skill

最稳妥的方式，是在提示词里显式写出 skill 名：

```text
使用 $collect-questions，把下面这批 Agent / RAG 面试题整理进仓库。
```

也可以直接用下面这些表达：

- 收集题目
- 整理面试题
- 归类到题库
- 更新 `records/2026-03.md`
- 给某道题补参考答案

### 5. 推荐使用方式

示例一：新增一批题目

```text
使用 $collect-questions，把下面这批题加入仓库，并补充 solutions 和 records。
```

示例二：只补某一章的答案

```text
使用 $collect-questions，为 07-memory-and-rag 里这几道题补参考答案。
```

示例三：只更新月度记录

```text
使用 $collect-questions，把今天新增的题目追加到 records/2026-03.md。
```

### 6. 这个 Skill 会做什么

`collect-questions` 的默认工作流是：

1. 解析你提供的题目列表。
2. 按章节归类到对应的 `questions/XX-topic/README.md`。
3. 避免把高度相似的问题重复写入题库。
4. 为题目补或更新 `solutions/XX-topic.md`。
5. 在 `records/YYYY-MM.md` 里记录本次批次。

### 7. 使用时的注意事项

- 最好让 Codex 的当前工作区就是仓库根目录。
- 如果只给主题、不提供具体题目，skill 不应该凭空生成并落库。
- 如果仓库里已有高度相似的问题，skill 应该避免重复写入 `questions/`。
- 若目录结构和默认假设不一致，应先按实际仓库结构调整。

## 如何贡献

欢迎提交 PR。提交前建议先确认题目归类、编号连续性和 `records/` / `solutions/` 链接是否一致。

## License

[MIT](LICENSE)
