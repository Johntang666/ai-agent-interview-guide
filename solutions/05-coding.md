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

---

### Q: 用 Python 实现一个简单的记忆检查点（Memory Checkpointer），支持保存和恢复 Agent 状态。

> 🤖 **Claude**

#### 1. 网络整合回答

在复杂的 Agent 系统中，任务执行往往是长时间、多步骤的过程。Agent 可能需要执行数十轮工具调用、经历多次规划-执行-反思循环，整个过程可能持续数分钟甚至数小时。在这种场景下，如果 Agent 因为网络异常、API 限流、服务重启或用户主动中断等原因被打断，没有检查点机制就意味着所有中间状态全部丢失，必须从头重跑，造成巨大的时间和 Token 成本浪费。

Memory Checkpointer（记忆检查点）的核心思想借鉴自操作系统和分布式计算中的 checkpoint/restart 机制：在任务执行的关键节点将当前完整状态持久化到磁盘或数据库，当需要恢复时，从最近的检查点加载状态并继续执行。对于 Agent 系统来说，需要保存的状态通常包括以下几个维度：(1) 对话历史（conversation history），这是 LLM 推理的上下文基础；(2) 当前计划及执行进度（plan steps），记录任务分解后哪些步骤已完成、哪些待执行；(3) 工具调用记录及结果（tool call history），避免重复调用已经成功的工具；(4) 自定义元数据（metadata），如当前使用的模型、温度参数、重试次数等运行时配置。

在序列化方案的选择上，JSON 具有可读性强、跨语言兼容的优势，适合大多数场景；pickle 则能处理更复杂的 Python 对象（如函数引用、自定义类实例），但存在安全风险且不跨语言。实际生产中通常优先使用 JSON，对于无法 JSON 序列化的对象采用自定义的序列化/反序列化钩子。多版本管理是另一个关键设计点：系统应该支持保留多个检查点，支持按时间戳回溯到任意历史状态，并提供自动清理过期检查点的能力，避免磁盘空间无限膨胀。LangGraph 的 Checkpointer 接口就是这类设计的典型参考，它将检查点抽象为 (thread_id, checkpoint_id) 二维索引，支持在同一会话中维护多个分支状态。

#### 2. 结合实际例子

```python
“””
Memory Checkpointer —— Agent 状态检查点管理器
支持保存/恢复 Agent 状态，多版本管理，自动清理过期检查点
“””

import json
import os
import time
import uuid
import copy
from dataclasses import dataclass, field, asdict
from typing import Any


# ──────────────────────────────────────────────
# 1. Agent 状态数据结构
# ──────────────────────────────────────────────
@dataclass
class ToolCallRecord:
    “””一次工具调用的完整记录”””
    tool_name: str
    arguments: dict
    result: Any
    timestamp: float
    success: bool

    def to_dict(self) -> dict:
        return asdict(self)

    @classmethod
    def from_dict(cls, data: dict) -> “ToolCallRecord”:
        return cls(**data)


@dataclass
class PlanStep:
    “””计划中的单个步骤”””
    step_id: int
    description: str
    status: str = “pending”  # pending / running / completed / failed
    result: str = “”

    def to_dict(self) -> dict:
        return asdict(self)

    @classmethod
    def from_dict(cls, data: dict) -> “PlanStep”:
        return cls(**data)


@dataclass
class AgentState:
    “””Agent 的完整运行状态”””
    conversation_history: list[dict] = field(default_factory=list)
    plan_steps: list[PlanStep] = field(default_factory=list)
    tool_call_history: list[ToolCallRecord] = field(default_factory=list)
    metadata: dict = field(default_factory=dict)

    def to_dict(self) -> dict:
        return {
            “conversation_history”: copy.deepcopy(self.conversation_history),
            “plan_steps”: [s.to_dict() for s in self.plan_steps],
            “tool_call_history”: [t.to_dict() for t in self.tool_call_history],
            “metadata”: copy.deepcopy(self.metadata),
        }

    @classmethod
    def from_dict(cls, data: dict) -> “AgentState”:
        return cls(
            conversation_history=data[“conversation_history”],
            plan_steps=[PlanStep.from_dict(s) for s in data[“plan_steps”]],
            tool_call_history=[ToolCallRecord.from_dict(t) for t in data[“tool_call_history”]],
            metadata=data.get(“metadata”, {}),
        )


# ──────────────────────────────────────────────
# 2. Checkpointer 核心实现
# ──────────────────────────────────────────────
class MemoryCheckpointer:
    “””
    Agent 状态检查点管理器
    - save_checkpoint: 保存当前状态
    - load_checkpoint: 恢复指定或最新的检查点
    - list_checkpoints: 列出所有检查点
    - delete_checkpoint: 删除指定检查点
    - auto_cleanup: 只保留最近 N 个检查点
    “””

    def __init__(self, storage_dir: str = “./checkpoints”, max_keep: int = 10):
        self.storage_dir = storage_dir
        self.max_keep = max_keep
        os.makedirs(storage_dir, exist_ok=True)

    def _checkpoint_path(self, checkpoint_id: str) -> str:
        return os.path.join(self.storage_dir, f”{checkpoint_id}.json”)

    def save_checkpoint(
        self,
        state: AgentState,
        tag: str = “”,
    ) -> str:
        “””
        保存 Agent 状态到检查点文件，返回 checkpoint_id。
        tag: 可选的人类可读标签，如 'before_tool_call'。
        “””
        checkpoint_id = f”{int(time.time())}_{uuid.uuid4().hex[:8]}”
        checkpoint_data = {
            “checkpoint_id”: checkpoint_id,
            “tag”: tag,
            “created_at”: time.time(),
            “created_at_human”: time.strftime(“%Y-%m-%d %H:%M:%S”),
            “state”: state.to_dict(),
        }
        path = self._checkpoint_path(checkpoint_id)
        with open(path, “w”, encoding=”utf-8”) as f:
            json.dump(checkpoint_data, f, ensure_ascii=False, indent=2)

        # 自动清理旧检查点
        self._auto_cleanup()
        return checkpoint_id

    def load_checkpoint(self, checkpoint_id: str | None = None) -> AgentState:
        “””
        加载指定检查点；若 checkpoint_id 为 None 则加载最新的。
        “””
        if checkpoint_id is None:
            checkpoints = self.list_checkpoints()
            if not checkpoints:
                raise FileNotFoundError(“没有可用的检查点”)
            checkpoint_id = checkpoints[0][“checkpoint_id”]  # 最新

        path = self._checkpoint_path(checkpoint_id)
        with open(path, “r”, encoding=”utf-8”) as f:
            data = json.load(f)
        return AgentState.from_dict(data[“state”])

    def list_checkpoints(self) -> list[dict]:
        “””列出所有检查点，按时间倒序排列”””
        results = []
        for fname in os.listdir(self.storage_dir):
            if fname.endswith(“.json”):
                path = os.path.join(self.storage_dir, fname)
                with open(path, “r”, encoding=”utf-8”) as f:
                    data = json.load(f)
                results.append({
                    “checkpoint_id”: data[“checkpoint_id”],
                    “tag”: data.get(“tag”, “”),
                    “created_at_human”: data[“created_at_human”],
                    “created_at”: data[“created_at”],
                })
        results.sort(key=lambda x: x[“created_at”], reverse=True)
        return results

    def delete_checkpoint(self, checkpoint_id: str) -> bool:
        path = self._checkpoint_path(checkpoint_id)
        if os.path.exists(path):
            os.remove(path)
            return True
        return False

    def _auto_cleanup(self):
        “””保留最近 max_keep 个检查点，删除其余”””
        checkpoints = self.list_checkpoints()
        if len(checkpoints) > self.max_keep:
            for old in checkpoints[self.max_keep:]:
                self.delete_checkpoint(old[“checkpoint_id”])


# ──────────────────────────────────────────────
# 3. 演示：模拟一个 Agent 运行并使用检查点
# ──────────────────────────────────────────────
def demo():
    checkpointer = MemoryCheckpointer(storage_dir=”./demo_checkpoints”, max_keep=5)
    state = AgentState(
        metadata={“model”: “gpt-4”, “temperature”: 0.3, “task”: “分析销售数据”}
    )

    # 模拟第 1 步：用户输入 + 计划制定
    state.conversation_history.append({“role”: “user”, “content”: “帮我分析上个月的销售数据”})
    state.plan_steps = [
        PlanStep(step_id=1, description=”读取数据库中的销售数据”),
        PlanStep(step_id=2, description=”按区域汇总计算”),
        PlanStep(step_id=3, description=”生成可视化图表”),
    ]
    cp1 = checkpointer.save_checkpoint(state, tag=”plan_created”)
    print(f”[检查点 1] 计划制定完成 -> {cp1}”)

    # 模拟第 2 步：执行工具调用
    state.plan_steps[0].status = “completed”
    state.tool_call_history.append(ToolCallRecord(
        tool_name=”query_database”,
        arguments={“sql”: “SELECT * FROM sales WHERE month='2025-02'”},
        result={“rows”: 1500, “total_revenue”: 2800000},
        timestamp=time.time(),
        success=True,
    ))
    state.conversation_history.append({
        “role”: “assistant”,
        “content”: “已查询到 1500 条销售记录，总营收 280 万。”
    })
    cp2 = checkpointer.save_checkpoint(state, tag=”step1_done”)
    print(f”[检查点 2] 步骤 1 完成 -> {cp2}”)

    # 模拟中断恢复
    print(“\n--- 模拟 Agent 崩溃，从最近检查点恢复 ---”)
    restored_state = checkpointer.load_checkpoint()  # 加载最新
    print(f”恢复的对话轮数: {len(restored_state.conversation_history)}”)
    print(f”恢复的计划步骤: {[s.description for s in restored_state.plan_steps]}”)
    print(f”已完成的步骤: {[s.description for s in restored_state.plan_steps if s.status == 'completed']}”)
    print(f”待执行的步骤: {[s.description for s in restored_state.plan_steps if s.status == 'pending']}”)
    print(f”已调用的工具: {[t.tool_name for t in restored_state.tool_call_history]}”)

    # 列出所有检查点
    print(“\n--- 所有检查点 ---”)
    for cp in checkpointer.list_checkpoints():
        print(f”  {cp['checkpoint_id']} | {cp['tag']} | {cp['created_at_human']}”)

    # 也可以回退到更早的检查点
    print(“\n--- 回退到检查点 1（计划刚制定时） ---”)
    old_state = checkpointer.load_checkpoint(cp1)
    print(f”回退后的工具调用数: {len(old_state.tool_call_history)}”)  # 应该是 0
    print(f”回退后的步骤状态: {[f'{s.description}={s.status}' for s in old_state.plan_steps]}”)

    # 清理演示目录
    import shutil
    shutil.rmtree(“./demo_checkpoints”, ignore_errors=True)


if __name__ == “__main__”:
    demo()
```

#### 3. 面试核心回答

- **为什么需要 Checkpointer**：长任务 Agent 随时可能中断（网络、限流、重启），没有检查点就得从头重跑，浪费时间和 Token。
- **状态四要素**：对话历史（LLM 上下文）、计划步骤及进度、工具调用记录与结果、运行时元数据，四者缺一则恢复后行为不一致。
- **多版本管理**：保留多个检查点支持按时间回溯，搭配自动清理策略防止磁盘膨胀；用 (timestamp + uuid) 做版本标识既有序又唯一。
- **序列化选型**：优先 JSON（可读、跨语言、安全），对不可序列化对象提供自定义钩子；避免 pickle 在生产环境中的安全风险。
- **生产级考量**：检查点写入应该是原子操作（先写临时文件再 rename），支持持久化到远端存储（S3/Redis），并与 Agent 的重试/恢复逻辑解耦。

一句话总结：Memory Checkpointer 的本质是把 Agent 的”运行现场”完整快照到持久层，让长任务具备中断恢复和状态回溯的能力。

---

### Q: 如何设计一个对 LLM 幻觉参数名或参数值具有鲁棒性的工具调用函数？

> 🤖 **Claude**

#### 1. 网络整合回答

在 Agent 系统中，LLM 负责根据用户意图决定调用哪个工具、传什么参数。但 LLM 的输出本质上是概率采样，即使在 function calling 模式下也会出现各种”幻觉”：编造不存在的参数名（如把 `file_path` 写成 `filepath` 或 `path_to_file`）、生成错误的参数类型（如该传整数却传了字符串 `”42”`）、给出不在合法范围内的参数值（如枚举字段传了不存在的选项）。这些幻觉在工具数量多、参数复杂时尤为严重，如果没有鲁棒性设计，轻则工具调用失败，重则产生错误结果且难以排查。

防御 LLM 幻觉参数的策略可以分为三个层次。第一层是**严格的 Schema 验证**：使用 Pydantic 等库定义每个工具的参数模型，在工具执行前自动校验参数名、类型、取值范围，不合法就拒绝执行。第二层是**容错与自动修复**：对常见的幻觉模式做预处理，包括参数名模糊匹配（用编辑距离或别名映射将错误参数名纠正为正确名称）、类型强制转换（字符串 `”42”` 自动转为 int `42`）、默认值回退（缺失的可选参数自动填充默认值）、枚举值模糊匹配（将 `”cancell”` 纠正为 `”cancel”`）。第三层是**闭环重试机制**：当参数验证失败且自动修复无法处理时，将结构化的错误信息（哪个参数出了什么问题、合法的值是什么）反馈给 LLM，让它重新生成参数。通常 1-2 轮重试就能修正大多数幻觉。

这三个层次层层递进：先尝试自动修复，修复不了再拒绝并要求重试，形成”宽进严出”的防御体系。Pydantic V2 的 validator 和 model_validator 是实现前两层的最佳工具，而重试逻辑则需要在 Agent 的工具调用编排层实现。关键设计原则是：错误信息要具体且对 LLM 友好，不能只返回 “invalid parameter”，而要明确指出 “参数 `statuss` 不存在，你是否想用 `status`？合法值为 ['active', 'inactive', 'pending']”。

#### 2. 结合实际例子

```python
“””
鲁棒性工具调用框架 —— 防御 LLM 幻觉参数名和参数值
三层防御：Schema 验证 -> 自动修复 -> 反馈重试
“””

from pydantic import BaseModel, field_validator, model_validator, ValidationError, Field
from typing import Any, Callable, Literal
from difflib import get_close_matches
import json


# ──────────────────────────────────────────────
# 1. 参数名模糊匹配：修复幻觉参数名
# ──────────────────────────────────────────────
def fuzzy_fix_param_names(
    raw_params: dict,
    valid_names: list[str],
    cutoff: float = 0.6,
) -> tuple[dict, list[str]]:
    “””
    将 LLM 幻觉出的错误参数名模糊匹配到合法参数名。
    返回 (修复后的参数 dict, 修复日志列表)。
    “””
    fixed = {}
    logs = []
    for key, value in raw_params.items():
        if key in valid_names:
            fixed[key] = value
        else:
            matches = get_close_matches(key, valid_names, n=1, cutoff=cutoff)
            if matches:
                fixed[matches[0]] = value
                logs.append(f”参数名 '{key}' 不存在，已自动纠正为 '{matches[0]}'”)
            else:
                logs.append(f”参数名 '{key}' 无法识别，已忽略（合法参数: {valid_names}）”)
    return fixed, logs


# ──────────────────────────────────────────────
# 2. Pydantic 模型：严格验证 + 类型强制转换
# ──────────────────────────────────────────────
class SearchToolParams(BaseModel):
    “””搜索工具的参数定义 —— 演示各种防御策略”””

    query: str = Field(..., min_length=1, max_length=500, description=”搜索关键词”)
    max_results: int = Field(default=10, ge=1, le=100, description=”最大返回数量”)
    sort_by: Literal[“relevance”, “date”, “popularity”] = Field(
        default=”relevance”, description=”排序方式”
    )
    language: str = Field(default=”zh”, description=”语言代码，如 zh, en, ja”)

    @field_validator(“max_results”, mode=”before”)
    @classmethod
    def coerce_max_results(cls, v: Any) -> int:
        “””类型强制转换：字符串 '42' -> int 42”””
        if isinstance(v, str):
            try:
                return int(v)
            except ValueError:
                raise ValueError(f”max_results 应为整数，但收到 '{v}'”)
        return v

    @field_validator(“sort_by”, mode=”before”)
    @classmethod
    def fuzzy_match_sort_by(cls, v: Any) -> str:
        “””枚举值模糊匹配：'relevanc' -> 'relevance'”””
        valid = [“relevance”, “date”, “popularity”]
        if isinstance(v, str) and v not in valid:
            matches = get_close_matches(v, valid, n=1, cutoff=0.5)
            if matches:
                return matches[0]
            raise ValueError(f”sort_by 的值 '{v}' 无效，合法值: {valid}”)
        return v

    @field_validator(“language”, mode=”before”)
    @classmethod
    def normalize_language(cls, v: Any) -> str:
        “””语言代码归一化”””
        if isinstance(v, str):
            v = v.lower().strip()
            lang_aliases = {
                “chinese”: “zh”, “english”: “en”, “japanese”: “ja”,
                “中文”: “zh”, “英文”: “en”, “日文”: “ja”,
            }
            return lang_aliases.get(v, v)
        return v


# ──────────────────────────────────────────────
# 3. 鲁棒性工具调用框架
# ──────────────────────────────────────────────
class RobustToolExecutor:
    “””
    具有鲁棒性的工具调用执行器。
    三层防御：参数名修复 -> Pydantic 验证 -> 错误反馈重试。
    “””

    def __init__(self):
        self.tools: dict[str, dict] = {}

    def register_tool(
        self,
        name: str,
        func: Callable,
        param_model: type[BaseModel],
        description: str = “”,
    ):
        self.tools[name] = {
            “func”: func,
            “param_model”: param_model,
            “description”: description,
        }

    def execute(
        self,
        tool_name: str,
        raw_params: dict,
        max_retries: int = 2,
        llm_retry_fn: Callable | None = None,
    ) -> dict:
        “””
        执行工具调用，包含完整的防御逻辑。

        参数:
            tool_name: 工具名称
            raw_params: LLM 生成的原始参数（可能包含幻觉）
            max_retries: 最大重试次数
            llm_retry_fn: 回调函数，接收错误信息返回修正后的参数
                          签名: (error_message: str) -> dict
        “””
        if tool_name not in self.tools:
            # 工具名也可能是幻觉，尝试模糊匹配
            matches = get_close_matches(tool_name, list(self.tools.keys()), n=1, cutoff=0.6)
            if matches:
                tool_name = matches[0]
            else:
                return {“success”: False, “error”: f”工具 '{tool_name}' 不存在，可用工具: {list(self.tools.keys())}”}

        tool = self.tools[tool_name]
        param_model = tool[“param_model”]
        valid_names = list(param_model.model_fields.keys())
        current_params = raw_params

        for attempt in range(max_retries + 1):
            # 第 1 层：参数名模糊修复
            fixed_params, fix_logs = fuzzy_fix_param_names(current_params, valid_names)
            if fix_logs:
                print(f”  [自动修复] {'; '.join(fix_logs)}”)

            # 第 2 层：Pydantic 严格验证（含类型转换和值修复）
            try:
                validated = param_model(**fixed_params)
                # 验证通过，执行工具
                result = tool[“func”](**validated.model_dump())
                return {
                    “success”: True,
                    “result”: result,
                    “auto_fixes”: fix_logs,
                    “attempts”: attempt + 1,
                }
            except ValidationError as e:
                error_msg = self._format_error_for_llm(e, param_model)
                print(f”  [验证失败 #{attempt + 1}] {error_msg}”)

                # 第 3 层：反馈 LLM 重试
                if attempt < max_retries and llm_retry_fn:
                    print(f”  [重试] 将错误反馈给 LLM 进行自纠正...”)
                    current_params = llm_retry_fn(error_msg)
                else:
                    return {
                        “success”: False,
                        “error”: error_msg,
                        “attempts”: attempt + 1,
                    }

        return {“success”: False, “error”: “超过最大重试次数”}

    def _format_error_for_llm(self, error: ValidationError, model: type[BaseModel]) -> str:
        “””将 Pydantic 验证错误格式化为 LLM 友好的反馈信息”””
        lines = [“参数验证失败，请修正以下问题：”]
        for err in error.errors():
            field_name = “.”.join(str(x) for x in err[“loc”])
            lines.append(f”  - 字段 '{field_name}': {err['msg']}”)
            # 附加该字段的合法信息
            if field_name in model.model_fields:
                field_info = model.model_fields[field_name]
                if field_info.description:
                    lines.append(f”    说明: {field_info.description}”)
        lines.append(f”完整参数 schema: {json.dumps(model.model_json_schema(), ensure_ascii=False, indent=2)}”)
        return “\n”.join(lines)


# ──────────────────────────────────────────────
# 4. 演示
# ──────────────────────────────────────────────
def mock_search(query: str, max_results: int = 10, sort_by: str = “relevance”, language: str = “zh”):
    “””模拟搜索工具”””
    return {“query”: query, “results_count”: max_results, “sort”: sort_by, “lang”: language}


def demo():
    executor = RobustToolExecutor()
    executor.register_tool(
        name=”search”,
        func=mock_search,
        param_model=SearchToolParams,
        description=”搜索工具”,
    )

    print(“=” * 60)
    print(“测试 1: 正常参数”)
    print(“=” * 60)
    result = executor.execute(“search”, {“query”: “AI Agent”, “max_results”: 5})
    print(f”结果: {result}\n”)

    print(“=” * 60)
    print(“测试 2: 幻觉参数名 (search_query -> query, num_results -> max_results)”)
    print(“=” * 60)
    result = executor.execute(“search”, {
        “search_query”: “AI Agent”,  # 错误参数名
        “num_results”: “20”,         # 错误参数名 + 字符串类型
        “sort_by”: “relevanc”,       # 拼写错误的枚举值
        “lang”: “Chinese”,           # 错误参数名 + 非标准值
    })
    print(f”结果: {result}\n”)

    print(“=” * 60)
    print(“测试 3: 严重幻觉 + LLM 重试”)
    print(“=” * 60)
    retry_count = 0

    def mock_llm_retry(error_message: str) -> dict:
        “””模拟 LLM 收到错误反馈后重新生成参数”””
        nonlocal retry_count
        retry_count += 1
        print(f”  [LLM 收到反馈] {error_message[:80]}...”)
        # 模拟 LLM 在第一次重试时修正参数
        return {“query”: “AI Agent 面试”, “max_results”: 10, “sort_by”: “date”}

    result = executor.execute(
        “search”,
        {“q”: “AI”, “limit”: “999”, “order”: “newest”},  # 全是幻觉参数
        max_retries=2,
        llm_retry_fn=mock_llm_retry,
    )
    print(f”结果: {result}\n”)

    print(“=” * 60)
    print(“测试 4: 工具名也是幻觉 (serach -> search)”)
    print(“=” * 60)
    result = executor.execute(“serach”, {“query”: “test”})
    print(f”结果: {result}”)


if __name__ == “__main__”:
    demo()
```

#### 3. 面试核心回答

- **LLM 幻觉的三种形态**：参数名幻觉（编造不存在的字段名）、类型幻觉（字符串传给整数字段）、值幻觉（枚举值拼写错误或不在合法范围内），工具越多、参数越复杂，幻觉概率越高。
- **三层防御体系**：第一层参数名模糊匹配（编辑距离 + 别名映射），第二层 Pydantic 严格验证 + 类型强制转换 + 默认值回退，第三层将结构化错误信息反馈给 LLM 重试自纠正。
- **错误信息要对 LLM 友好**：不能只返回 “validation error”，必须具体指出哪个参数、什么问题、合法值有哪些，LLM 才能有效自纠正。
- **工具 Schema 设计也是防线**：参数名要语义明确避免歧义，尽量用枚举而非自由文本，可选参数都给默认值，减少 LLM 犯错的空间。
- **重试要有上限**：通常 1-2 次重试足够，超过 3 次大概率是工具定义本身有问题或用户意图不匹配，应该升级处理而非无限循环。

一句话总结：鲁棒性工具调用的核心是”宽进严出”——用模糊匹配和类型转换容忍 LLM 的小错误，用 Schema 验证拦住大错误，再用错误反馈闭环让 LLM 自己改正。
