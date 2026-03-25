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

---

### Q: 用 Python 实现一个带有语义缓存（Semantic Cache）的工具调用层，相似请求复用缓存结果以降低 LLM 调用成本。

> 🤖 **Claude**

#### 1. 网络整合回答

语义缓存（Semantic Cache）与传统的精确匹配缓存（Exact Match Cache）有本质区别：传统缓存要求请求完全一致才能命中，而语义缓存通过将请求文本转换为 embedding 向量，利用余弦相似度（Cosine Similarity）衡量语义距离，只要两个请求的语义足够接近（超过设定阈值），就可以复用已缓存的结果。例如”北京今天天气怎么样”和”今日北京的天气情况”虽然字面不同，但语义几乎一致，语义缓存可以将它们识别为同一请求。

**核心组件与设计要点**：

1. **Embedding 模型选择**：通常使用 OpenAI 的 `text-embedding-3-small`（1536 维）或开源的 `sentence-transformers`（如 `all-MiniLM-L6-v2`，384 维）。开源模型无需 API 调用，适合对延迟敏感的场景；商业模型语义表达更精准，适合多语言场景。

2. **相似度阈值设定**：这是语义缓存最关键的超参数。根据 2025 年的实践经验，阈值设为 0.90 是大多数场景的推荐起点——在此阈值下缓存命中率可达 40-60%，同时几乎不会产生误匹配。阈值设为 0.85 会提高命中率但增加误匹配风险（如”商店几点开门”和”商店几点关门”可能被错误匹配）；阈值设为 0.95 则更安全但命中率显著下降。不同类型的查询（代码类、自然语言类）在 embedding 空间的分布密度不同，理想状态下应针对不同查询类别设置不同阈值。

3. **向量存储方案**：轻量级场景可用 FAISS（Facebook AI Similarity Search）的 `IndexFlatIP`（内积索引，对归一化向量等价于余弦相似度），支持精确搜索且无需外部依赖；生产环境推荐 Redis + RedisVL 向量搜索模块（支持持久化、分布式、TTL 原生支持），或 Qdrant/Milvus 等专用向量数据库。

4. **TTL 过期策略**：缓存条目必须设置生存时间（Time-To-Live），避免过时数据被持续返回。工具调用结果的时效性差异很大——天气查询可能 10 分钟过期，知识库问答可以缓存数小时。应支持按工具类型配置不同 TTL。

5. **缓存命中率优化**：据 Redis 官方基准测试，语义缓存在客户支持 FAQ、文档问答等语义重复度高的场景下，缓存命中率可达 60-85%，API 调用减少 68.8%，延迟从约 1.67 秒降至 0.052 秒（降低 96.9%）。优化手段包括：对请求文本做预处理（去除冗余词、标准化格式）、将工具名 + 参数序列化作为缓存 key 的一部分以区分不同工具的调用、以及定期分析缓存未命中的请求模式来调整阈值。

6. **开源生态**：GPTCache（Zilliz 出品）是目前最成熟的语义缓存开源方案，已与 LangChain 和 LlamaIndex 深度集成；LangChain 也提供了 `RedisSemanticCache` 开箱即用组件。

#### 2. 结合实际例子

```python
“””
语义缓存工具调用层（Semantic Cache for Tool Calling）
功能：对语义相似的工具调用请求复用缓存结果，降低 LLM API 成本
依赖：pip install numpy sentence-transformers faiss-cpu
“””

import time
import hashlib
import json
from dataclasses import dataclass, field
from typing import Any, Callable, Optional
import numpy as np


# ============================================================
# 1. Embedding 提供者（支持替换为 OpenAI 等远程模型）
# ============================================================
class EmbeddingProvider:
    “””使用 sentence-transformers 生成本地 embedding，零 API 成本”””

    def __init__(self, model_name: str = “all-MiniLM-L6-v2”):
        from sentence_transformers import SentenceTransformer

        self.model = SentenceTransformer(model_name)
        self.dimension = self.model.get_sentence_embedding_dimension()

    def embed(self, text: str) -> np.ndarray:
        “””将文本转换为归一化的 embedding 向量”””
        vec = self.model.encode(text, normalize_embeddings=True)
        return vec.astype(np.float32)


# ============================================================
# 2. 缓存条目数据结构
# ============================================================
@dataclass
class CacheEntry:
    “””缓存条目：存储工具调用的请求、结果和元数据”””

    tool_name: str  # 工具名称
    request_text: str  # 原始请求文本
    result: Any  # 工具调用返回结果
    created_at: float  # 创建时间戳
    ttl: float  # 生存时间（秒）
    hit_count: int = 0  # 命中次数

    @property
    def is_expired(self) -> bool:
        return time.time() - self.created_at > self.ttl


# ============================================================
# 3. 语义缓存核心类
# ============================================================
class SemanticCache:
    “””
    基于 FAISS 的语义缓存
    - 使用 embedding 将请求向量化
    - 余弦相似度匹配（归一化向量的内积 = 余弦相似度）
    - 可配置相似度阈值和 TTL
    - 内置命中/未命中统计
    “””

    def __init__(
        self,
        embedding_provider: EmbeddingProvider,
        similarity_threshold: float = 0.90,
        default_ttl: float = 3600,
        max_cache_size: int = 10000,
    ):
        import faiss

        self.embedding_provider = embedding_provider
        self.similarity_threshold = similarity_threshold
        self.default_ttl = default_ttl
        self.max_cache_size = max_cache_size

        # FAISS 索引：IndexFlatIP 对归一化向量等价于余弦相似度搜索
        dim = embedding_provider.dimension
        self.index = faiss.IndexFlatIP(dim)

        # 缓存存储：index_id -> CacheEntry
        self.entries: dict[int, CacheEntry] = {}
        self.next_id: int = 0

        # 统计信息
        self.stats = {“hits”: 0, “misses”: 0, “evictions”: 0}

    def _build_cache_key_text(self, tool_name: str, request_text: str) -> str:
        “””
        将工具名和请求文本组合为缓存 key 文本
        确保不同工具的相似请求不会互相命中
        “””
        return f”[TOOL:{tool_name}] {request_text}”

    def lookup(self, tool_name: str, request_text: str) -> Optional[Any]:
        “””
        查询语义缓存
        返回值：命中时返回缓存结果，未命中返回 None
        “””
        if self.index.ntotal == 0:
            self.stats[“misses”] += 1
            return None

        # 生成查询 embedding
        key_text = self._build_cache_key_text(tool_name, request_text)
        query_vec = self.embedding_provider.embed(key_text)
        query_vec = query_vec.reshape(1, -1)

        # FAISS 搜索最相似的 1 个向量
        similarities, indices = self.index.search(query_vec, k=1)
        best_sim = similarities[0][0]
        best_idx = indices[0][0]

        # 判断是否超过相似度阈值
        if best_sim >= self.similarity_threshold and best_idx in self.entries:
            entry = self.entries[best_idx]

            # 检查是否过期
            if entry.is_expired:
                self._evict(best_idx)
                self.stats[“misses”] += 1
                return None

            # 缓存命中
            entry.hit_count += 1
            self.stats[“hits”] += 1
            return entry.result

        self.stats[“misses”] += 1
        return None

    def store(
        self,
        tool_name: str,
        request_text: str,
        result: Any,
        ttl: Optional[float] = None,
    ) -> None:
        “””将工具调用结果存入缓存”””
        # 容量管理：超出上限时清除过期条目
        if self.index.ntotal >= self.max_cache_size:
            self._cleanup_expired()

        # 如果清理后仍超上限，移除最早的条目
        if self.index.ntotal >= self.max_cache_size:
            oldest_id = min(self.entries.keys())
            self._evict(oldest_id)

        key_text = self._build_cache_key_text(tool_name, request_text)
        vec = self.embedding_provider.embed(key_text)
        vec = vec.reshape(1, -1)

        # 添加到 FAISS 索引
        self.index.add(vec)
        entry_id = self.next_id
        self.next_id += 1

        # 存储缓存条目
        self.entries[entry_id] = CacheEntry(
            tool_name=tool_name,
            request_text=request_text,
            result=result,
            created_at=time.time(),
            ttl=ttl or self.default_ttl,
        )

    def _evict(self, entry_id: int) -> None:
        “””逐出指定条目（逻辑删除，FAISS 不支持单条删除时标记为过期）”””
        if entry_id in self.entries:
            del self.entries[entry_id]
            self.stats[“evictions”] += 1

    def _cleanup_expired(self) -> None:
        “””批量清除所有过期条目”””
        expired_ids = [
            eid for eid, entry in self.entries.items() if entry.is_expired
        ]
        for eid in expired_ids:
            self._evict(eid)

    def get_stats(self) -> dict:
        “””返回缓存统计信息”””
        total = self.stats[“hits”] + self.stats[“misses”]
        hit_rate = self.stats[“hits”] / total if total > 0 else 0.0
        return {
            **self.stats,
            “total_requests”: total,
            “hit_rate”: f”{hit_rate:.1%}”,
            “cached_entries”: len(self.entries),
        }


# ============================================================
# 4. 带语义缓存的工具调用层
# ============================================================
class CachedToolExecutor:
    “””
    工具调用执行器，集成语义缓存
    - 注册工具函数
    - 调用时自动查询/写入缓存
    - 支持按工具配置不同 TTL
    “””

    def __init__(self, cache: SemanticCache):
        self.cache = cache
        self.tools: dict[str, Callable] = {}
        self.tool_ttls: dict[str, float] = {}  # 按工具自定义 TTL

    def register_tool(
        self, name: str, func: Callable, ttl: Optional[float] = None
    ) -> None:
        “””注册工具函数，可指定该工具专属 TTL”””
        self.tools[name] = func
        if ttl is not None:
            self.tool_ttls[name] = ttl

    def call_tool(self, tool_name: str, request_text: str, **kwargs) -> dict:
        “””
        执行工具调用（带语义缓存）
        返回：{“result”: ..., “cached”: bool, “similarity”: float|None}
        “””
        if tool_name not in self.tools:
            return {“error”: f”未知工具: {tool_name}”, “cached”: False}

        # Step 1: 查询缓存
        cached_result = self.cache.lookup(tool_name, request_text)
        if cached_result is not None:
            return {“result”: cached_result, “cached”: True}

        # Step 2: 缓存未命中，实际调用工具
        try:
            result = self.tools[tool_name](request_text, **kwargs)
        except Exception as e:
            return {“error”: str(e), “cached”: False}

        # Step 3: 将结果写入缓存
        ttl = self.tool_ttls.get(tool_name)
        self.cache.store(tool_name, request_text, result, ttl=ttl)

        return {“result”: result, “cached”: False}


# ============================================================
# 5. 模拟 LLM 工具调用场景的完整 Demo
# ============================================================
def demo():
    “””演示语义缓存在工具调用中的完整工作流程”””

    print(“=” * 60)
    print(“初始化语义缓存系统”)
    print(“=” * 60)

    # 初始化 embedding 提供者和缓存
    embedder = EmbeddingProvider(model_name=”all-MiniLM-L6-v2”)
    cache = SemanticCache(
        embedding_provider=embedder,
        similarity_threshold=0.90,  # 相似度 >= 0.90 视为命中
        default_ttl=3600,  # 默认 1 小时过期
    )
    executor = CachedToolExecutor(cache)

    # ----------------------------------------------------------
    # 注册模拟工具
    # ----------------------------------------------------------
    def weather_tool(query: str, **kwargs) -> dict:
        “””模拟天气查询 API（实际场景会调用真实 API）”””
        time.sleep(0.1)  # 模拟网络延迟
        return {“city”: “北京”, “temp”: “22°C”, “condition”: “晴”}

    def search_tool(query: str, **kwargs) -> dict:
        “””模拟搜索工具（实际场景会调用 LLM）”””
        time.sleep(0.5)  # 模拟 LLM 调用延迟
        return {“answer”: f”关于'{query}'的搜索结果...”, “sources”: 3}

    executor.register_tool(“weather”, weather_tool, ttl=600)  # 天气 10 分钟过期
    executor.register_tool(“search”, search_tool, ttl=7200)  # 搜索 2 小时过期

    # ----------------------------------------------------------
    # 测试 1: 首次调用（缓存未命中）
    # ----------------------------------------------------------
    print(“\n” + “=” * 60)
    print(“测试 1: 首次调用天气工具（应该缓存未命中）”)
    print(“=” * 60)
    t0 = time.time()
    result1 = executor.call_tool(“weather”, “北京今天天气怎么样”)
    t1 = time.time()
    print(f”结果: {result1}”)
    print(f”耗时: {t1 - t0:.3f}s”)

    # ----------------------------------------------------------
    # 测试 2: 语义相似请求（应该缓存命中）
    # ----------------------------------------------------------
    print(“\n” + “=” * 60)
    print(“测试 2: 语义相似的天气请求（应该缓存命中）”)
    print(“=” * 60)
    similar_queries = [
        “今天北京的天气情况”,
        “北京今日天气如何”,
        “查一下北京现在的天气”,
    ]
    for q in similar_queries:
        t0 = time.time()
        result = executor.call_tool(“weather”, q)
        t1 = time.time()
        print(f”  请求: '{q}' -> cached={result['cached']}, 耗时={t1 - t0:.4f}s”)

    # ----------------------------------------------------------
    # 测试 3: 不同工具的相似请求（不应互相命中）
    # ----------------------------------------------------------
    print(“\n” + “=” * 60)
    print(“测试 3: 不同工具的请求不会互相命中”)
    print(“=” * 60)
    result3 = executor.call_tool(“search”, “北京今天天气怎么样”)
    print(f”  search 工具结果: cached={result3['cached']}”)

    # ----------------------------------------------------------
    # 测试 4: 语义差异较大的请求（不应命中）
    # ----------------------------------------------------------
    print(“\n” + “=” * 60)
    print(“测试 4: 语义差异较大的请求（不应命中）”)
    print(“=” * 60)
    result4 = executor.call_tool(“weather”, “上海明天会下雨吗”)
    print(f”  '上海明天会下雨吗' -> cached={result4['cached']}”)

    # ----------------------------------------------------------
    # 查看缓存统计
    # ----------------------------------------------------------
    print(“\n” + “=” * 60)
    print(“缓存统计”)
    print(“=” * 60)
    stats = cache.get_stats()
    for k, v in stats.items():
        print(f”  {k}: {v}”)


if __name__ == “__main__”:
    demo()
```

**运行效果示意**：

```
初始化语义缓存系统
============================================================

测试 1: 首次调用天气工具（应该缓存未命中）
============================================================
结果: {'result': {'city': '北京', 'temp': '22°C', 'condition': '晴'}, 'cached': False}
耗时: 0.127s

测试 2: 语义相似的天气请求（应该缓存命中）
============================================================
  请求: '今天北京的天气情况' -> cached=True, 耗时=0.0089s
  请求: '北京今日天气如何' -> cached=True, 耗时=0.0085s
  请求: '查一下北京现在的天气' -> cached=True, 耗时=0.0091s

测试 3: 不同工具的请求不会互相命中
============================================================
  search 工具结果: cached=False

测试 4: 语义差异较大的请求（不应命中）
============================================================
  '上海明天会下雨吗' -> cached=False

缓存统计
============================================================
  hits: 3
  misses: 4
  evictions: 0
  total_requests: 7
  hit_rate: 42.9%
  cached_entries: 4
```

#### 3. 面试核心回答

- **语义缓存 vs 精确缓存的本质区别**：精确缓存要求请求字符串完全一致才能命中，语义缓存通过 embedding 向量化 + 余弦相似度匹配，能识别”北京今天天气怎么样”和”今日北京天气如何”为同一请求，大幅提高缓存命中率（实测可达 60-85%），API 调用成本降低 60% 以上。
- **阈值是核心超参数**：相似度阈值推荐从 0.90 起步。过低（如 0.85）会导致语义不同的请求被错误匹配（”几点开门” vs “几点关门”），过高（如 0.95）则命中率过低失去缓存意义。理想方案是按工具类别、查询类型设置不同阈值。
- **向量存储选型**：开发/轻量场景用 FAISS 的 `IndexFlatIP`（归一化向量的内积 = 余弦相似度），零外部依赖；生产环境用 Redis + RedisVL（原生支持 TTL、持久化、分布式）或 Milvus/Qdrant 等专用向量数据库。
- **TTL 过期必须分工具配置**：天气类实时数据 TTL 设 10 分钟，知识库问答可设数小时，避免一刀切导致过时数据被返回或缓存利用率低。
- **缓存 key 要包含工具名**：将 `[TOOL:工具名] 请求文本` 作为 embedding 输入，确保不同工具的语义相似请求不会互相命中（同一句话对 weather 和 search 工具应分别缓存）。

一句话总结：语义缓存的核心是用 embedding + 余弦相似度替代精确字符串匹配，关键在于选好阈值（推荐 0.90）、按工具隔离缓存命名空间、按场景配置 TTL，就能在不牺牲结果质量的前提下将 LLM 调用成本降低 60% 以上。

---

### Q: 用 Python 实现一个带有语义缓存（Semantic Cache）的工具调用层，相似请求复用缓存结果以降低 LLM 调用成本。

> 🤖 **Claude**

#### 1. 网络整合回答

语义缓存（Semantic Cache）是一种 AI 原生的缓存策略，与传统精确匹配缓存（Exact Match Cache）在匹配机制上存在根本差异。传统缓存以请求字符串的哈希值作为 key，要求请求文本逐字一致才能命中；而语义缓存将请求通过 Embedding 模型映射到高维向量空间，再通过余弦相似度（Cosine Similarity）度量两个请求之间的语义距离——只要相似度超过预设阈值，即视为"同一语义请求"并返回缓存结果。例如 "Python 中如何读取 JSON 文件" 和 "怎样在 Python 里解析 JSON" 字面差异显著，但在 embedding 空间中距离极近，语义缓存能够将它们识别为等价请求。

核心工作流程分为四步：(1) **请求 Embedding**——将用户的工具调用请求文本通过 embedding 模型（如 OpenAI `text-embedding-3-small` 或本地 `all-MiniLM-L6-v2`）转换为固定维度的浮点向量；(2) **向量相似度搜索**——在缓存的向量索引中检索与当前请求最相似的历史向量，计算余弦相似度；(3) **阈值判定与返回**——若最高相似度 >= 阈值且缓存未过期，直接返回缓存结果（cache hit）；(4) **未命中写入**——若低于阈值或缓存过期，执行实际的工具调用或 LLM 推理，将结果连同 embedding 一并写入缓存。

相似度阈值的选择直接决定缓存质量。根据 2025 年的大量实践数据，通用场景推荐 **0.92-0.95** 区间：低于 0.90 时误匹配率显著上升（如 "取消订单" 和 "查询订单" 可能被错误视为等价），高于 0.97 时命中率过低，缓存价值有限。最佳实践是按业务场景分层设定——FAQ 类查询可用较低阈值（0.90），涉及数值参数的精确查询应使用较高阈值（0.95+）。

TTL（Time-To-Live）过期策略同样不可忽视。不同工具返回结果的时效性差异悬殊：实时数据（股票、天气）TTL 可能只有 5-10 分钟，而知识库问答的 TTL 可以设为数小时甚至数天。缺少 TTL 机制的缓存会导致过时结果被持续返回，严重影响用户体验。

在向量存储方面，FAISS（Facebook AI Similarity Search）是本地轻量级首选，其 `IndexFlatIP` 索引对归一化向量的内积搜索等价于余弦相似度，纯内存运行无外部依赖；分布式场景推荐 Redis Stack 的向量搜索模块（原生 TTL + 持久化 + 集群），或 Qdrant、Milvus 等专用向量数据库。据 Redis 官方测试报告和 Asteria 论文的评估，语义缓存在重复度高的 Agent 场景中可将 LLM 调用量减少 50-90%，响应延迟从 1-5 秒降至毫秒级，实现 10x-100x 的速度提升。在 Agentic 系统中，缓存命中不仅省去一次 LLM 调用——它还能短路（short-circuit）整条推理链，避免后续的检索、推理和工具调用级联开销，放大效果远超单次调用节省。

#### 2. 结合实际例子

```python
"""
语义缓存工具调用层（Semantic Cache Tool Layer）——装饰器模式实现
核心特性：
  - 纯 numpy 余弦相似度，无需 FAISS 依赖
  - 装饰器一行代码为任意工具函数启用语义缓存
  - 可配置相似度阈值、TTL、最大容量
  - 内置命中率/节省成本统计面板
依赖：pip install numpy openai
"""

import time
import json
import functools
from dataclasses import dataclass, field
from typing import Any, Callable, Optional
import numpy as np

# ─────────────────────────────────────────────────
# 1. Embedding 接口（支持 OpenAI 和纯本地两种模式）
# ─────────────────────────────────────────────────
class OpenAIEmbedder:
    """通过 OpenAI API 生成 embedding"""

    def __init__(self, model: str = "text-embedding-3-small"):
        from openai import OpenAI
        self.client = OpenAI()
        self.model = model
        self._dim: Optional[int] = None

    def embed(self, text: str) -> np.ndarray:
        resp = self.client.embeddings.create(input=[text], model=self.model)
        vec = np.array(resp.data[0].embedding, dtype=np.float32)
        # L2 归一化，使内积 = 余弦相似度
        norm = np.linalg.norm(vec)
        if norm > 0:
            vec /= norm
        if self._dim is None:
            self._dim = len(vec)
        return vec

    @property
    def dimension(self) -> int:
        if self._dim is None:
            # 触发一次调用以获取维度
            self.embed("dim probe")
        return self._dim


class NumpyLocalEmbedder:
    """基于 sentence-transformers 的本地 embedding，零 API 开销"""

    def __init__(self, model_name: str = "all-MiniLM-L6-v2"):
        from sentence_transformers import SentenceTransformer
        self.model = SentenceTransformer(model_name)

    def embed(self, text: str) -> np.ndarray:
        vec = self.model.encode(text, normalize_embeddings=True)
        return vec.astype(np.float32)

    @property
    def dimension(self) -> int:
        return self.model.get_sentence_embedding_dimension()


# ─────────────────────────────────────────────────
# 2. 缓存条目
# ─────────────────────────────────────────────────
@dataclass
class CacheItem:
    key_text: str                # 原始请求文本
    embedding: np.ndarray        # 请求的 embedding 向量
    tool_name: str               # 所属工具
    result: Any                  # 缓存的返回值
    created_at: float            # 写入时间戳
    ttl: float                   # 生存时间（秒）
    hit_count: int = 0           # 被命中次数

    @property
    def is_expired(self) -> bool:
        return time.time() - self.created_at > self.ttl


# ─────────────────────────────────────────────────
# 3. SemanticToolCache 核心类
# ─────────────────────────────────────────────────
class SemanticToolCache:
    """
    纯 numpy 实现的语义缓存，无外部向量数据库依赖。
    适用场景：缓存条目 < 100k 的中小规模 Agent 系统。
    """

    def __init__(
        self,
        embedder,
        default_threshold: float = 0.92,
        default_ttl: float = 3600.0,
        max_entries: int = 50000,
    ):
        self.embedder = embedder
        self.default_threshold = default_threshold
        self.default_ttl = default_ttl
        self.max_entries = max_entries

        # 按工具名分桶存储（避免跨工具误匹配）
        self._buckets: dict[str, list[CacheItem]] = {}

        # 每个工具可单独配置阈值和 TTL
        self._tool_thresholds: dict[str, float] = {}
        self._tool_ttls: dict[str, float] = {}

        # 统计面板
        self.stats = {
            "total_lookups": 0,
            "hits": 0,
            "misses": 0,
            "expired_hits": 0,   # 命中但已过期
            "writes": 0,
            "evictions": 0,
            "estimated_cost_saved": 0.0,  # 估算节省的美元数
        }

    def configure_tool(
        self, tool_name: str, threshold: float = None, ttl: float = None
    ) -> None:
        """为特定工具设置独立的相似度阈值和 TTL"""
        if threshold is not None:
            self._tool_thresholds[tool_name] = threshold
        if ttl is not None:
            self._tool_ttls[tool_name] = ttl

    def _cosine_similarity(self, a: np.ndarray, b: np.ndarray) -> float:
        """计算两个已归一化向量的余弦相似度（等价于点积）"""
        return float(np.dot(a, b))

    def _cosine_similarity_batch(
        self, query: np.ndarray, matrix: np.ndarray
    ) -> np.ndarray:
        """批量余弦相似度：query (d,) vs matrix (n, d) -> (n,)"""
        return matrix @ query

    def lookup(
        self, tool_name: str, request_text: str, cost_per_call: float = 0.003
    ) -> Optional[Any]:
        """
        在缓存中查找语义相似的历史请求。
        cost_per_call: 估算单次 LLM 调用成本（用于统计面板）。
        """
        self.stats["total_lookups"] += 1
        bucket = self._buckets.get(tool_name)
        if not bucket:
            self.stats["misses"] += 1
            return None

        # 生成查询 embedding
        query_vec = self.embedder.embed(request_text)

        # 批量计算相似度（numpy 向量化，比逐条循环快 10-50x）
        embeddings_matrix = np.stack([item.embedding for item in bucket])
        similarities = self._cosine_similarity_batch(query_vec, embeddings_matrix)

        # 找到最大相似度
        best_idx = int(np.argmax(similarities))
        best_sim = float(similarities[best_idx])

        threshold = self._tool_thresholds.get(tool_name, self.default_threshold)

        if best_sim >= threshold:
            item = bucket[best_idx]
            # 检查 TTL 过期
            if item.is_expired:
                self.stats["expired_hits"] += 1
                self.stats["misses"] += 1
                bucket.pop(best_idx)
                self.stats["evictions"] += 1
                return None

            # 缓存命中
            item.hit_count += 1
            self.stats["hits"] += 1
            self.stats["estimated_cost_saved"] += cost_per_call
            return item.result

        self.stats["misses"] += 1
        return None

    def store(self, tool_name: str, request_text: str, result: Any) -> None:
        """将工具调用结果写入缓存"""
        if tool_name not in self._buckets:
            self._buckets[tool_name] = []

        bucket = self._buckets[tool_name]

        # 容量管理：先清理过期条目
        before = len(bucket)
        self._buckets[tool_name] = [item for item in bucket if not item.is_expired]
        self.stats["evictions"] += before - len(self._buckets[tool_name])
        bucket = self._buckets[tool_name]

        # 若仍超上限，移除最久未被命中的条目（LFU 策略）
        total_entries = sum(len(b) for b in self._buckets.values())
        while total_entries >= self.max_entries and bucket:
            bucket.sort(key=lambda x: (x.hit_count, x.created_at))
            bucket.pop(0)
            self.stats["evictions"] += 1
            total_entries -= 1

        ttl = self._tool_ttls.get(tool_name, self.default_ttl)
        embedding = self.embedder.embed(request_text)
        bucket.append(CacheItem(
            key_text=request_text,
            embedding=embedding,
            tool_name=tool_name,
            result=result,
            created_at=time.time(),
            ttl=ttl,
        ))
        self.stats["writes"] += 1

    def report(self) -> str:
        """生成可读的缓存统计报告"""
        s = self.stats
        total = s["total_lookups"] or 1
        hit_rate = s["hits"] / total
        lines = [
            "── Semantic Cache Report ──",
            f"  总查询数: {s['total_lookups']}",
            f"  命中: {s['hits']}  |  未命中: {s['misses']}  |  过期命中: {s['expired_hits']}",
            f"  命中率: {hit_rate:.1%}",
            f"  写入: {s['writes']}  |  淘汰: {s['evictions']}",
            f"  估算节省: ${s['estimated_cost_saved']:.4f}",
            f"  缓存条目: {sum(len(b) for b in self._buckets.values())}",
        ]
        return "\n".join(lines)


# ─────────────────────────────────────────────────
# 4. 装饰器：一行代码为工具函数启用语义缓存
# ─────────────────────────────────────────────────
def semantic_cached(
    cache: SemanticToolCache,
    tool_name: str = None,
    cost_per_call: float = 0.003,
):
    """
    装饰器工厂：为工具函数添加语义缓存能力。

    用法：
        @semantic_cached(cache, tool_name="weather")
        def get_weather(query: str) -> dict:
            return call_weather_api(query)
    """
    def decorator(func: Callable) -> Callable:
        name = tool_name or func.__name__

        @functools.wraps(func)
        def wrapper(request_text: str, *args, **kwargs) -> dict:
            # 查缓存
            cached = cache.lookup(name, request_text, cost_per_call)
            if cached is not None:
                return {"result": cached, "cached": True, "tool": name}

            # 未命中，调用实际工具
            result = func(request_text, *args, **kwargs)

            # 写入缓存
            cache.store(name, request_text, result)
            return {"result": result, "cached": False, "tool": name}

        # 暴露缓存引用，便于外部检查
        wrapper._cache = cache
        wrapper._tool_name = name
        return wrapper

    return decorator


# ─────────────────────────────────────────────────
# 5. 完整 Demo：模拟 Agent 多工具调用场景
# ─────────────────────────────────────────────────
def demo():
    """
    演示装饰器模式下语义缓存的完整工作流程：
    - 注册多个工具
    - 模拟语义相似 / 不同的请求
    - 观察命中率和性能差异
    """
    # ---------- 初始化 ----------
    # 生产环境替换为 OpenAIEmbedder()
    # 这里用 NumpyLocalEmbedder 做演示（无需 API key）
    embedder = NumpyLocalEmbedder("all-MiniLM-L6-v2")
    cache = SemanticToolCache(
        embedder=embedder,
        default_threshold=0.92,
        default_ttl=3600,
    )
    # 天气工具：低阈值（天气查询表述变化大）、短 TTL
    cache.configure_tool("weather", threshold=0.88, ttl=600)
    # 知识问答工具：高阈值（精确性要求高）、长 TTL
    cache.configure_tool("knowledge_qa", threshold=0.94, ttl=7200)

    # ---------- 用装饰器注册工具 ----------
    @semantic_cached(cache, tool_name="weather", cost_per_call=0.002)
    def get_weather(query: str) -> dict:
        time.sleep(0.05)  # 模拟 API 延迟
        return {"city": "上海", "temp": "26°C", "humidity": "72%"}

    @semantic_cached(cache, tool_name="knowledge_qa", cost_per_call=0.005)
    def ask_knowledge_base(query: str) -> dict:
        time.sleep(0.2)  # 模拟 LLM 推理延迟
        return {"answer": f"关于 '{query}' 的知识库回答", "confidence": 0.95}

    # ---------- 模拟请求序列 ----------
    print("=" * 55)
    print("  Semantic Cache Demo — 装饰器模式")
    print("=" * 55)

    requests = [
        ("weather",      "上海今天天气怎么样"),         # miss
        ("weather",      "今天上海天气如何"),           # hit（语义相似）
        ("weather",      "上海今日的气温和湿度"),       # hit（语义相似）
        ("weather",      "北京明天会下雪吗"),           # miss（不同城市）
        ("knowledge_qa", "Python 中如何读取 JSON 文件"), # miss
        ("knowledge_qa", "怎样在 Python 里解析 JSON"),   # hit（语义相似）
        ("knowledge_qa", "Java 读取 JSON 的方法"),       # miss（不同语言）
    ]

    for tool, query in requests:
        t0 = time.time()
        if tool == "weather":
            res = get_weather(query)
        else:
            res = ask_knowledge_base(query)
        elapsed = time.time() - t0
        tag = "HIT " if res["cached"] else "MISS"
        print(f"  [{tag}] {tool:15s} | {query:30s} | {elapsed:.4f}s")

    # ---------- 输出统计报告 ----------
    print()
    print(cache.report())


if __name__ == "__main__":
    demo()
```

**运行效果示意**：

```
=======================================================
  Semantic Cache Demo — 装饰器模式
=======================================================
  [MISS] weather         | 上海今天天气怎么样              | 0.0732s
  [HIT ] weather         | 今天上海天气如何                | 0.0214s
  [HIT ] weather         | 上海今日的气温和湿度            | 0.0198s
  [MISS] weather         | 北京明天会下雪吗                | 0.0715s
  [MISS] knowledge_qa    | Python 中如何读取 JSON 文件     | 0.2306s
  [HIT ] knowledge_qa    | 怎样在 Python 里解析 JSON       | 0.0187s
  [MISS] knowledge_qa    | Java 读取 JSON 的方法           | 0.2289s

── Semantic Cache Report ──
  总查询数: 7
  命中: 3  |  未命中: 4  |  过期命中: 0
  命中率: 42.9%
  写入: 4  |  淘汰: 0
  估算节省: $0.0090
  缓存条目: 4
```

#### 3. 面试核心回答

- **语义缓存的本质**：用 embedding 向量空间中的余弦相似度替代字符串精确匹配，使 "上海今天天气" 和 "今天上海天气如何" 这类语义等价但字面不同的请求能够共享缓存，命中率比精确缓存高出数倍。
- **阈值是成败关键**：推荐从 0.92 起步，按工具类型分层调整——表述灵活的查询（天气、闲聊）适当下调至 0.88-0.90，精确性要求高的查询（代码生成、数据库操作）上调至 0.94-0.96，避免 "取消订单" 与 "查询订单" 被误匹配。
- **工具级隔离与 TTL 分治**：缓存按工具名分桶存储，杜绝跨工具误命中；TTL 按数据时效性分配——实时数据 5-10 分钟，知识问答数小时，静态配置可达数天。
- **装饰器模式降低接入成本**：通过 `@semantic_cached(cache, tool_name="xxx")` 装饰器，一行代码为任意工具函数启用语义缓存，无需修改工具内部逻辑，符合开放-封闭原则。
- **Agentic 场景的放大效应**：缓存命中不仅节省一次 LLM 调用，还能短路整条推理链——跳过后续的检索、推理和级联工具调用，实际节省可达标称成本的 3-5 倍。

一句话总结：语义缓存通过 embedding + 余弦相似度将缓存粒度从"字面一致"提升到"语义等价"，配合按工具分桶隔离、分层阈值和 TTL 策略，能以极低的接入成本（装饰器一行）实现 LLM 调用量 50-90% 的缩减。

---

### Q: 用 Python 实现一个最简单的 ReAct Agent（不使用任何框架）。

#### 1. 网络整合回答 / Comprehensive Answer

最小 ReAct Agent 的核心就是一个 while loop：把用户问题、已有 scratchpad 和工具描述发给模型；模型返回 Thought 和 Action；如果是工具调用，就执行并把 Observation 拼回上下文；如果是 Final Answer，就结束。  
实现时最重要的不是功能多，而是协议清晰。你要明确模型输出格式、最大步数、工具错误处理和终止条件。否则很容易进入死循环或解析失败。

#### 2. 结合实际例子 / Practical Example

```python
while step < max_steps:
    response = llm(prompt + scratchpad)
    thought, action = parse(response)
    if action.type == "final":
        return action.answer
    obs = tools[action.name](**action.args)
    scratchpad += f"\nThought: {thought}\nAction: {action}\nObservation: {obs}"
```

#### 3. 面试核心回答 / Core Interview Answer

- 最简 ReAct = 模型推理 + 工具执行 + 观察回填循环。
- 必须限制步数、约束输出格式、处理工具异常。
- 重点不是复杂框架，而是闭环是否稳定。

一句话总结：手写 ReAct 的本质是把"想一步、做一步、看结果"写成一个可控循环。

---

### Q: 实现一个支持 Function Calling 的 Agent 循环。

#### 1. 网络整合回答 / Comprehensive Answer

Function Calling 循环通常是：先向模型提供工具 schema，模型返回要调用的函数名和参数，执行器根据 schema 校验参数并调用真实函数，再把结果作为 tool message 回传给模型，直到模型返回最终自然语言答案。  
生产实现里要特别注意参数校验和工具结果结构化。模型可能选对工具但传错参数，所以执行前一定要做 schema validate；工具返回也最好是 JSON，便于后续继续推理。

#### 2. 结合实际例子 / Practical Example

```python
messages = [{"role": "user", "content": question}]
while True:
    resp = client.responses.create(model=model, tools=tool_schemas, input=messages)
    if not resp.tool_calls:
        return resp.output_text
    for call in resp.tool_calls:
        args = validate(call.arguments, schema_map[call.name])
        result = tool_map[call.name](**args)
        messages.append({"role": "tool", "name": call.name, "content": json.dumps(result)})
```

#### 3. 面试核心回答 / Core Interview Answer

- Function Calling 循环本质是"模型决定 -> 执行器校验执行 -> 结果回填"。
- 模型不直接执行工具，执行器才是真正的安全边界。
- schema 校验和结构化返回是稳定性的关键。

一句话总结：支持 Function Calling 的 Agent，本质上是一个围绕结构化工具调用搭起来的事件循环。

---

### Q: 编写一个具有短期记忆（对话历史）和长期记忆（向量存储）的 Agent。

#### 1. 网络整合回答 / Comprehensive Answer

这类 Agent 通常把短期记忆存在会话状态里，把长期记忆存在向量库或其他持久化存储里。运行时先读取短期对话上下文，再根据当前问题检索长期记忆片段，最后把两者一起送给模型。  
设计难点不在于"把历史都塞进去"，而在于摘要和筛选。短期历史过长要做压缩，长期记忆要控制召回质量和写入策略，避免把低质量噪声永久存进去。

#### 2. 结合实际例子 / Practical Example

```python
history = session_store.get(session_id)
memories = vector_store.search(embed(user_query), top_k=3)
prompt = build_prompt(history=history[-8:], memories=memories, query=user_query)
answer = llm(prompt)
session_store.append(session_id, {"user": user_query, "assistant": answer})
```

#### 3. 面试核心回答 / Core Interview Answer

- 短期记忆保当前会话，长期记忆保跨会话经验。
- 关键是召回和摘要，不是无限堆上下文。
- 写入长期记忆要有质量门槛和更新策略。

一句话总结：带记忆的 Agent，核心是让上下文既连续又不过载。

---

### Q: 实现一个 Plan-and-Execute Agent，先制定计划再逐步执行。

#### 1. 网络整合回答 / Comprehensive Answer

Plan-and-Execute 的实现通常分成两个阶段。规划器先把任务拆成若干有序步骤；执行器按步骤运行，每完成一步都更新状态；如果某步失败，可以局部重试或触发重规划。  
设计重点是计划要结构化，例如列表、状态和依赖，而不是一段自由文本。否则执行器很难准确消费计划。

#### 2. 结合实际例子 / Practical Example

```python
plan = planner_llm(f"请把任务拆成步骤: {task}")
for item in plan["steps"]:
    result = run_step(item, context)
    context["completed"].append({"step": item, "result": result})
    if result.get("need_replan"):
        plan = replanner(plan, context)
```

#### 3. 面试核心回答 / Core Interview Answer

- Plan-and-Execute 把"想怎么做"和"真正执行"分开。
- 计划必须结构化，执行过程必须可更新状态。
- 实用系统还要支持局部失败后的重规划。

一句话总结：Plan-and-Execute 的价值在于先把复杂任务拆开，再把执行过程收进状态机。

---

### Q: 编写一个让 LLM 稳定输出结构化 JSON 的 System Prompt。

#### 1. 网络整合回答 / Comprehensive Answer

让模型稳定输出 JSON，system prompt 需要同时约束格式和失败行为。通常要明确：只输出 JSON、不允许额外解释、字段名固定、字段类型固定、缺失信息如何处理、非法值如何置空。  
仅靠 prompt 还不够，最好再结合 schema 校验和重试。因为模型即便理解要求，也可能在边界 case 下多说一句或字段拼错。

#### 2. 结合实际例子 / Practical Example

```text
你必须只输出一个合法 JSON 对象。
禁止输出 markdown、解释文字、注释。
字段:
- intent: string
- confidence: number in [0,1]
- entities: array
若无法判断, 使用 null 或空数组。
```

#### 3. 面试核心回答 / Core Interview Answer

- prompt 要同时约束输出格式、字段类型和失败时行为。
- "只输出 JSON"、"字段固定"、"无法判断如何填"要写清楚。
- 生产里一定要配 schema 校验和重试。

一句话总结：稳定 JSON 不是靠一句"请返回 JSON"，而是靠格式约束加校验闭环。

---

### Q: 设计一个 Router Agent 的 Prompt，能根据用户意图路由到不同的子 Agent。

#### 1. 网络整合回答 / Comprehensive Answer

Router Prompt 的重点是分类边界清晰。你要明确有哪些子 Agent、各自负责什么、何时路由、何时拒绝或转人工。输出最好是结构化结果，例如 `target_agent`、`reason`、`confidence`。  
一个常见错误是把路由条件写得太抽象，导致模型什么都能归到多个 agent。更稳妥的写法是给每个子 Agent 明确正例、反例和冲突处理规则。

#### 2. 结合实际例子 / Practical Example

```text
你是路由器，只负责分类，不回答问题。
可选目标:
1. billing_agent: 发票、退款、支付异常
2. logistics_agent: 物流、签收、退货进度
3. policy_agent: 规则解释、政策咨询
若问题同时涉及敏感赔付，输出 human_review。
返回 JSON: {target_agent, confidence, reason}
```

#### 3. 面试核心回答 / Core Interview Answer

- Router Prompt 核心是边界清晰、输出结构化。
- 每个子 Agent 要有职责定义、正反例和冲突规则。
- 路由器负责分发，不负责回答。

一句话总结：好的 Router Prompt 不是让模型更会说，而是让它更少分错。

---

### Q: 如何通过 Prompt 实现 few-shot 工具选择？

#### 1. 网络整合回答 / Comprehensive Answer

few-shot 工具选择的关键是给模型展示"什么问题对应什么工具、为什么、参数长什么样"。示例最好覆盖相似工具之间的区分，例如查询订单和取消订单、读数据和写数据。  
如果只给工具描述不给例子，模型在边界 case 上容易混淆。few-shot 的价值就在于把抽象规则变成可模仿模式。

#### 2. 结合实际例子 / Practical Example

```text
示例1:
用户: 查询订单 123 的状态
应选工具: get_order_status(order_id="123")

示例2:
用户: 取消订单 123
应选工具: cancel_order(order_id="123")

示例3:
用户: 帮我看看订单大概到哪了
应选工具: get_order_status(...)
```

#### 3. 面试核心回答 / Core Interview Answer

- few-shot 要展示工具选择逻辑，而不只是列工具名。
- 示例应覆盖相似工具的边界差异。
- 最佳实践是示例 + schema + 规则联合使用。

一句话总结：few-shot 的本质是把"怎么选工具"示范给模型看。

---

### Q: 实现一个通用的工具注册和调用机制。

#### 1. 网络整合回答 / Comprehensive Answer

通用工具机制至少要有三块：注册表、schema、执行器。注册表保存工具名、描述、参数定义和权限要求；schema 用来校验输入；执行器负责超时、重试、审计和结果封装。  
这样设计的好处是新增工具时只需要注册，不必改动主流程。同时，工具调用可以统一做观测和安全控制。

#### 2. 结合实际例子 / Practical Example

```python
registry.register(
    name="get_weather",
    schema=WeatherArgs,
    handler=get_weather,
    permission="read_weather"
)
result = executor.call("get_weather", {"city": "Shanghai"})
```

#### 3. 面试核心回答 / Core Interview Answer

- 工具机制核心是注册表、参数 schema 和统一执行器。
- 新工具应通过配置接入，而不是在主逻辑里硬编码。
- 统一执行器可以集中处理权限、超时和审计。

一句话总结：好的工具机制让能力扩展像注册插件，而不是改主程序。

---

### Q: 编写一个带有错误重试和降级策略的工具调用包装器。

#### 1. 网络整合回答 / Comprehensive Answer

工具包装器通常要处理四类情况：瞬时失败、永久失败、超时和返回异常。对瞬时失败可做指数退避重试；对永久失败应立即停止；超时要可取消；返回异常要结构化上报。降级策略则取决于业务，比如换备用接口、返回缓存结果或转人工。  
重点不是"永远重试直到成功"，而是根据错误类型有节制地恢复。

#### 2. 结合实际例子 / Practical Example

```python
for attempt in range(3):
    try:
        return primary_tool(**args)
    except TimeoutError:
        sleep(backoff(attempt))
if fallback_tool:
    return fallback_tool(**args)
return {"status": "degraded", "reason": "tool_unavailable"}
```

#### 3. 面试核心回答 / Core Interview Answer

- 重试要按错误类型区分，不能盲目无限重试。
- 降级可以是备用接口、缓存结果或人工接管。
- 包装器要统一返回结构化错误，便于 Agent 决策下一步。

一句话总结：健壮的工具包装器不是保证不报错，而是保证出错时系统仍然可控。

---

### Q: 实现一个 Agent 的流式输出（Streaming），同时支持中间步骤展示。

#### 1. 网络整合回答 / Comprehensive Answer

流式输出通常分两类事件：最终内容流和中间过程流。最终内容流给用户连续反馈，中间过程流用于展示当前正在检索、调用工具还是等待审批。实现时可以用事件流模型，把不同阶段统一包装成事件并逐步推给前端。  
关键是不要把思维链原样暴露，而是展示适合用户看的摘要化步骤，例如"正在查询订单系统"、"正在生成报告"。

#### 2. 结合实际例子 / Practical Example

```python
yield {"type": "status", "message": "正在检索知识库"}
yield {"type": "tool", "name": "search_docs"}
for token in llm.stream(prompt):
    yield {"type": "token", "content": token}
yield {"type": "done"}
```

#### 3. 面试核心回答 / Core Interview Answer

- Streaming 不只是 token 流，还包括状态事件流。
- 中间步骤应摘要化展示，避免泄露不适合暴露的内部内容。
- 事件模型能让前端同时处理状态和结果。

一句话总结：好的流式输出让用户感觉 Agent 在工作，而不是在卡住。

---

### Q: 如何为 Agent 编写单元测试？Mock LLM 调用的最佳实践是什么？

#### 1. 网络整合回答 / Comprehensive Answer

Agent 单测的核心是把不稳定的外部依赖替换掉。LLM、向量库、外部 API、时间和随机数都应该 mock 或 stub，这样测试才能稳定复现。  
最佳实践通常是：为路由、工具选择、错误恢复、输出解析等关键节点单独写测试；对 LLM 返回做固定夹具；对工具调用顺序和参数做断言；只在少量集成测试中接真实模型。不要把所有质量判断都塞进慢而脆弱的端到端测试。

#### 2. 结合实际例子 / Practical Example

```python
mock_llm.return_value = {"tool": "get_order_status", "args": {"order_id": "123"}}
result = agent.run("查一下订单 123")
assert result["tool_called"] == "get_order_status"
assert tool_mock.called_once_with(order_id="123")
```

#### 3. 面试核心回答 / Core Interview Answer

- 单测要 mock 掉 LLM 和外部依赖，保证可重复。
- 重点测路由、解析、工具调用、失败恢复等稳定逻辑。
- 真实模型留给少量集成或回归测试，不适合当主力单测。

一句话总结：Agent 单测的核心，不是测模型有多聪明，而是测系统在固定输出下是否按预期工作。

---

### Q: 实现一个简单的 Agent 评估框架，支持多维度打分。

#### 1. 网络整合回答 / Comprehensive Answer

简单 eval 框架一般包括测试集、执行器、评分器和报告器。测试集定义输入和期望；执行器批量跑 agent；评分器按多个维度打分，比如任务成功、格式正确、工具使用是否合理、延迟和成本；报告器汇总均分和失败原因。  
设计重点是评分维度要可扩展，不能只留一个总分。多维分数才有助于定位问题。

#### 2. 结合实际例子 / Practical Example

```python
scores = {
    "task_success": judge_task(output, expected),
    "format_ok": judge_format(output),
    "tool_ok": judge_tool_trace(trace),
    "latency_ms": trace["latency_ms"],
}
report.append(scores)
```

#### 3. 面试核心回答 / Core Interview Answer

- Eval 框架最小四件套：样本、执行、评分、报告。
- 打分要多维，不要只看一个总分。
- 失败原因标签化，才能支撑版本迭代。

一句话总结：一个能用的评估框架，不是只会打分，而是能告诉你为什么扣分。
