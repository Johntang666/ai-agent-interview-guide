# AI Agent Interview Guide

**A comprehensive collection of AI Agent interview questions, from fundamentals to system design.**

[中文](#中文介绍) | [English](#english-introduction)

---

## 中文介绍

### 项目简介

本仓库是一份系统化的 **AI Agent 面试题库与知识体系整理**，涵盖从基础概念到系统设计的完整面试准备资料。适用于准备 AI Agent 相关岗位面试的工程师、研究员，以及希望深入理解 Agent 技术栈的从业者。

### 目标受众

- 准备 AI/LLM Agent 方向面试的候选人
- 希望考察候选人 Agent 能力的面试官
- 对 Agent 技术体系感兴趣的开发者

### 项目架构

```
ai-agent-interview-guide/
│
├── README.md                          # 项目说明（中英双语）
├── CONTRIBUTING.md                    # 贡献指南
│
├── questions/                         # 面试题目（按主题分类）
│   ├── 01-fundamentals/              # 基础概念
│   │   └── README.md                 # Agent 定义、核心组件、与传统 AI 的区别
│   ├── 02-frameworks/                # 主流框架
│   │   └── README.md                 # LangChain, AutoGen, CrewAI, Dify 等
│   ├── 03-design-patterns/           # 设计模式
│   │   └── README.md                 # ReAct, Plan-and-Execute, Reflexion 等
│   ├── 04-system-design/             # 系统设计
│   │   └── README.md                 # Agent 系统架构设计题
│   ├── 05-coding/                    # 编码实战
│   │   └── README.md                 # 手写 Agent、Tool 调用、Prompt 工程
│   ├── 06-multi-agent/               # 多智能体协作
│   │   └── README.md                 # 多 Agent 通信、协调、冲突解决
│   ├── 07-memory-and-rag/            # 记忆与检索增强
│   │   └── README.md                 # 短期/长期记忆、RAG 架构、向量数据库
│   ├── 08-tool-use/                  # 工具使用
│   │   └── README.md                 # Function Calling、工具选择、API 编排
│   ├── 09-evaluation-and-safety/     # 评估与安全
│   │   └── README.md                 # Agent 评测、对齐、安全防护
│   └── 10-real-world-cases/          # 真实案例
│       └── README.md                 # 工业级 Agent 落地案例分析
│
├── solutions/                         # 参考答案与深度解析
│   └── README.md
│
├── resources/                         # 学习资源
│   └── README.md                     # 论文、博客、课程、开源项目推荐
│
└── docs/                              # 补充文档
    └── roadmap.md                    # 学习路线图
```

### 内容模块说明

| 模块 | 说明 | 难度 |
|------|------|------|
| 01-基础概念 | Agent 定义、感知-推理-行动循环、与 Chatbot 区别 | ⭐ |
| 02-主流框架 | LangChain / LangGraph / AutoGen / CrewAI / Dify 对比 | ⭐⭐ |
| 03-设计模式 | ReAct / Plan-and-Execute / Reflexion / LATS 等模式 | ⭐⭐⭐ |
| 04-系统设计 | 生产级 Agent 系统架构设计 | ⭐⭐⭐⭐ |
| 05-编码实战 | 手写 Agent 核心逻辑、工具调用链 | ⭐⭐⭐ |
| 06-多智能体 | 多 Agent 拓扑、通信协议、任务分配 | ⭐⭐⭐⭐ |
| 07-记忆与 RAG | 记忆机制、RAG 管线、向量检索优化 | ⭐⭐⭐ |
| 08-工具使用 | Function Calling、工具编排、错误恢复 | ⭐⭐⭐ |
| 09-评估与安全 | Benchmark、红队测试、Prompt 注入防御 | ⭐⭐⭐⭐ |
| 10-真实案例 | 客服 Agent、代码 Agent、数据分析 Agent 等 | ⭐⭐⭐⭐⭐ |

### 如何使用

1. **系统学习**：按 01 → 10 顺序逐模块学习
2. **查漏补缺**：根据自身薄弱环节针对性练习
3. **模拟面试**：随机抽取不同模块题目进行限时作答
4. **深度研究**：参考 `solutions/` 中的详细解析和 `resources/` 中的延伸阅读

### 如何贡献

欢迎提交 PR！请阅读 [CONTRIBUTING.md](CONTRIBUTING.md) 了解贡献流程。

---

## English Introduction

### Overview

This repository is a **systematic AI Agent interview question bank and knowledge base**, covering everything from foundational concepts to production-level system design. It is designed for engineers and researchers preparing for AI Agent roles, as well as practitioners looking to deepen their understanding of the Agent technology stack.

### Target Audience

- Candidates preparing for AI/LLM Agent positions
- Interviewers assessing candidates' Agent expertise
- Developers interested in the Agent ecosystem

### Architecture

```
ai-agent-interview-guide/
│
├── README.md                          # Project introduction (bilingual)
├── CONTRIBUTING.md                    # Contribution guidelines
│
├── questions/                         # Interview questions (by topic)
│   ├── 01-fundamentals/              # Core concepts
│   ├── 02-frameworks/                # Popular frameworks
│   ├── 03-design-patterns/           # Agent design patterns
│   ├── 04-system-design/             # System design problems
│   ├── 05-coding/                    # Hands-on coding
│   ├── 06-multi-agent/               # Multi-agent collaboration
│   ├── 07-memory-and-rag/            # Memory & RAG
│   ├── 08-tool-use/                  # Tool use & function calling
│   ├── 09-evaluation-and-safety/     # Evaluation & safety
│   └── 10-real-world-cases/          # Real-world case studies
│
├── solutions/                         # Reference answers & analysis
├── resources/                         # Learning resources
└── docs/                              # Supplementary documents
```

### Module Overview

| Module | Description | Difficulty |
|--------|-------------|------------|
| 01 Fundamentals | Agent definition, perception-reasoning-action loop, vs. chatbot | ⭐ |
| 02 Frameworks | LangChain / LangGraph / AutoGen / CrewAI / Dify comparison | ⭐⭐ |
| 03 Design Patterns | ReAct / Plan-and-Execute / Reflexion / LATS patterns | ⭐⭐⭐ |
| 04 System Design | Production-grade Agent system architecture | ⭐⭐⭐⭐ |
| 05 Coding | Implement Agent core logic, tool-calling chains | ⭐⭐⭐ |
| 06 Multi-Agent | Topologies, communication protocols, task allocation | ⭐⭐⭐⭐ |
| 07 Memory & RAG | Memory mechanisms, RAG pipelines, vector search | ⭐⭐⭐ |
| 08 Tool Use | Function calling, tool orchestration, error recovery | ⭐⭐⭐ |
| 09 Eval & Safety | Benchmarks, red-teaming, prompt injection defense | ⭐⭐⭐⭐ |
| 10 Real-World Cases | Customer service, coding, data analysis Agents | ⭐⭐⭐⭐⭐ |

### How to Use

1. **Systematic Study**: Follow modules 01 → 10 in order
2. **Targeted Practice**: Focus on your weak areas
3. **Mock Interviews**: Randomly pick questions across modules for timed practice
4. **Deep Dive**: Refer to `solutions/` for detailed analysis and `resources/` for further reading

### How to Contribute

PRs are welcome! Please read [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

---

## License

[MIT](LICENSE)
