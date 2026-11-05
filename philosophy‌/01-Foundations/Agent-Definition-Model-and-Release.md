# Agent Definition、Model Profile 与 Release

## 本文档在系统中的作用

Agent Definition 描述 Agent 身份、职责和允许能力；Model Profile 描述调用模型、参数和路由策略；Release 将经过验证的配置固定为不可变版本。三者共同构成可复现运行基础。

## 1. 为什么不能只有一个 Agent JSON

把 system_prompt、model、tools、skills、memory、graph_config 全部放在一个大 JSON 中，短期开发快，但会导致字段无 Schema、权限与 Prompt 混杂、无法独立回滚、运行配置漂移、依赖版本不可复现。

## 2. Agent Definition 核心字段

`agent_key`、`display_name`、`description`、`agent_kind`、`graph_key`、`system_prompt_ref`、`model_profile_ref`、`context_profile_ref`、`toolset_ref`、`skill_refs`、`policy_profile_ref`、`memory_profile_ref`、`max_steps`、`max_runtime_seconds`、`max_cost`、`definition_status`。

Agent Definition 不保存运行时 Python 对象、数据库 Session 或 API Key。

## 3. Release Snapshot

Release 发布时解析所有引用，生成完整 Manifest：
- Agent Definition Version；
- Graph Builder Version；
- Prompt Version；
- Model Profile Version；
- Tool Version Set；
- Skill / Plugin Version Set；
- Context / Memory / Policy Profile Version；
- Python Package Lock Hash；
- Catalog Snapshot ID。

Manifest 不可变并保存 checksum。Run 创建时固定 `agent_release_id`，保证问题可以重现。

## 4. Pydantic V2 配置模型

```python
from typing import Literal
from pydantic import BaseModel, ConfigDict, Field

class ExecutionBudget(BaseModel):
    model_config = ConfigDict(extra="forbid", frozen=True)
    max_steps: int = Field(default=20, ge=1, le=200)
    max_runtime_seconds: int = Field(default=300, ge=1, le=7200)
    max_tool_calls: int = Field(default=30, ge=0, le=500)
    max_cost_usd: float = Field(default=2.0, gt=0)

class AgentDefinition(BaseModel):
    model_config = ConfigDict(extra="forbid", frozen=True)
    agent_key: str = Field(min_length=2, max_length=120)
    agent_kind: Literal["assistant", "workflow", "specialist", "supervisor"]
    graph_key: str
    prompt_ref: str
    model_profile_ref: str
    toolset_ref: str
    policy_profile_ref: str
    budget: ExecutionBudget
```

## 5. 发布流程

`Draft → Static Validation → Dependency Resolution → Security Review → Evaluation → Approval → Published`

检查所有引用存在、Schema 兼容、Tool Handler 可加载、Skill / Plugin 扫描通过、模型能力满足要求、Golden Dataset 达标、Policy 未放宽高风险权限、Manifest 可复现。

## 6. 配置热更新

新 Run 使用最新已发布 Release；已运行 Run 使用固定 Release。若 Tool / Plugin 出现安全问题，可通过全局 Revocation List 阻止新执行，即使旧 Release 仍引用它。撤销是安全覆盖，不是悄悄修改旧 Release 内容。

## 7. 验收

任意历史 Run 都可以查出其 Release、Model Profile、Prompt、Tool Versions、Catalog Snapshot 和 Policy Version；回滚通过重新发布旧版本实现。
