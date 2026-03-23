# 05 - 编码实战 / Coding

本文件收录编码实战章节的参考答案与深度解析。

---

### Q: 有微调过 Agent 能力吗？数据集如何收集？

#### 1. 网络整合回答 / Comprehensive Answer

可以微调的通常不是“整个 Agent 系统”，而是其中某些稳定子能力，比如工具选择、参数填充、任务拆解、格式化输出、代码修复、反思打分等。对于变化快、依赖外部环境的部分，纯微调往往不如“提示词 + 工作流 + 评测闭环”有效。  
数据集收集通常来自四类：真实用户任务日志、人工标注的高质量示范、系统运行 trace 回放、以及用强模型生成的合成数据。关键不是数据量本身，而是样本是否覆盖失败案例、边界条件和高价值任务。实践里通常会先做数据清洗、去重、脱敏、按任务类型分桶，再构建 train / eval / hard-case 集合。

#### 2. 结合实际例子 / Practical Example

如果要微调“工具调用能力”，一套常见做法是：

- 收集真实对话和工具调用日志。
- 标注正确工具名、参数、调用时机和是否需要澄清。
- 把失败案例单独整理，例如误选工具、漏参数、重复调用。
- 用这些样本训练或蒸馏一个更稳的 tool-routing 模型，再在线上 A/B 对比。

#### 3. 面试核心回答 / Core Interview Answer

- Agent 微调要聚焦稳定子任务，不要笼统说“把整个 Agent 微调一下”。
- 数据来源优先是真实日志、人工示范、trace 回放和合成 hard cases。
- 没有高质量 eval set，微调收益往往不可验证。

一句话总结：微调 Agent 能力的关键不是”训没训”，而是你是否找到了稳定、可标注、可评测的能力切片。

> 🤖 **Claude**

#### 1. 网络整合回答 / Comprehensive Answer

微调 Agent 能力是一个需要精准定位的工程决策，而不是”把所有 Agent 行为都训进模型”。Agent 系统通常由多个环节组成——意图识别、任务分解、工具选择、参数填充、结果判断、错误恢复等——其中只有部分环节适合通过微调来优化。

适合微调的能力切片通常满足三个条件：(1) 输入输出格式稳定；(2) 有足够的标注数据或可生成合成数据；(3) 对延迟或成本敏感，需要用小模型替代大模型。典型场景包括：工具路由（给定用户意图选择正确工具）、参数提取（从自然语言中抽取结构化参数）、质量打分（判断 Agent 输出是否合格）、意图分类等。

数据集收集的核心策略有五种：
1. **生产日志挖掘**：从线上 Agent 的 trace 中提取成功案例，按 (input, expected_output) 格式整理
2. **人工标注**：对失败案例和边界案例做专家标注，这部分数据质量最高但成本也最高
3. **强模型蒸馏**：用 GPT-4 / Claude 等强模型生成标注数据，再用这些数据训练小模型
4. **对抗样本生成**：针对已知弱点构造 hard cases，如工具名相近时的区分、参数缺失时的澄清
5. **用户反馈闭环**：将用户的 thumbs up/down、人工修正结果回流为训练数据

数据质量远比数量重要。100 条精标的失败案例，比 10000 条自动采集的成功案例更有微调价值。

#### 2. 结合实际例子 / Practical Example

```python
# 示例：构建工具路由微调数据集
import json
from openai import OpenAI

client = OpenAI()

# 第一步：从线上 trace 提取训练样本
def extract_training_data(traces: list[dict]) -> list[dict]:
    “””从 Agent 运行 trace 中提取工具调用的训练数据”””
    samples = []
    for trace in traces:
        if trace[“status”] == “success” and trace.get(“tool_calls”):
            for call in trace[“tool_calls”]:
                samples.append({
                    “messages”: [
                        {“role”: “system”, “content”: “你是工具路由助手，根据用户意图选择合适的工具。”},
                        {“role”: “user”, “content”: trace[“user_input”]},
                    ],
                    “expected_tool”: call[“tool_name”],
                    “expected_params”: call[“parameters”],
                })
    return samples

# 第二步：用强模型生成合成 hard cases
def generate_hard_cases(tools_schema: list, n: int = 200):
    “””让强模型生成容易混淆的边界案例”””
    prompt = f”””
    已知工具列表：{json.dumps(tools_schema, ensure_ascii=False)}
    请生成 {n} 个容易让模型选错工具的用户输入，格式如下：
    {{“user_input”: “...”, “correct_tool”: “...”, “confusing_tool”: “...”, “reason”: “...”}}
    重点覆盖：工具名相似、功能重叠、需要多工具配合的场景。
    “””
    response = client.chat.completions.create(
        model=”claude-sonnet-4-6”,
        messages=[{“role”: “user”, “content”: prompt}]
    )
    return json.loads(response.choices[0].message.content)

# 第三步：构建评测集
def build_eval_set(samples: list, ratio: float = 0.2):
    “””按任务类型分层抽样构建评测集”””
    from collections import defaultdict
    import random
    by_tool = defaultdict(list)
    for s in samples:
        by_tool[s[“expected_tool”]].append(s)
    eval_set, train_set = [], []
    for tool, items in by_tool.items():
        random.shuffle(items)
        split = int(len(items) * ratio)
        eval_set.extend(items[:split])
        train_set.extend(items[split:])
    return train_set, eval_set
```

#### 3. 面试核心回答 / Core Interview Answer

- 微调要聚焦”稳定、可标注、可评测”的能力切片，如工具路由、参数提取、质量打分。
- 数据收集五条路：生产日志、人工标注、强模型蒸馏、对抗样本、用户反馈闭环。
- 数据质量 > 数量：100 条精标失败案例比 10000 条自动成功日志更有价值。
- 必须先有评测集再微调，否则无法验证收益。
- 微调不是万能药：变化快的部分（如 prompt 策略）用提示词迭代更灵活。

一句话总结：微调 Agent 的正确姿势是”先找到值得训的能力切片，再用高质量数据训出可评测的提升”。
