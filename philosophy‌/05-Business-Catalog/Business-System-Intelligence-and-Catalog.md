# Business Catalog：让 Agent 理解业务系统

## 本文档在系统中的作用

Business Catalog 是业务系统的机器可读语义目录。它把 API、Schema、实体、关系、操作、权限、业务术语和用户任务连接起来，让 Agent 理解“我要完成什么业务目标”，而不是要求模型记住成百上千个原始 API。

## 1. 为什么不能把所有 API 都做成 Tool

假设系统有 800 个 API：全量暴露会占用大量 Tool Schema Token；模型容易选错相似 API；原始 API 语义通常偏技术而非业务；参数可能需要多个前置查询；权限、审批和幂等策略分散；API 更新后 Tool Schema 容易过期。

因此需要两层抽象：
1. Business Capability：业务动作，例如“查询带教项目”“查询学员状态”“更新学员状态”。
2. API Binding：Capability 背后具体调用哪个 API / Service / Query。

模型主要理解 Capability；系统执行时解析 Binding。

## 2. Catalog 核心对象

- Domain：业务域，如 mentorship、billing、crm。
- Entity：项目、学员、导师、订单、客户。
- Attribute：状态、负责人、开始时间。
- Relationship：项目包含学员、学员分配导师。
- Capability：search / get / create / update / transition / approve。
- API Asset：HTTP Method、Path、OperationId、Schema。
- Data Asset：Table、Column、Constraint、Foreign Key。
- Permission：谁可以执行什么动作。
- Business Rule：状态迁移、必填条件、限制。
- Evidence：来源代码、文档、审核记录。
- Snapshot：已发布目录版本。

## 3. Entity Catalog 示例

`Learner`：
- primary identifiers：learner_id；
- display attributes：name、employee_no；
- state attributes：learning_status；
- relationships：mentorship_project、mentor；
- read capabilities：get、search、list_history；
- write capabilities：request_status_transition；
- sensitive attributes：contact、salary 等按策略限制。

Agent 不需要直接访问全部 Learner 表字段。Catalog 只暴露经过审核的字段和 Capability。

## 4. Capability Contract

```python
from pydantic import BaseModel
from typing import Literal

class CapabilityDefinition(BaseModel):
    capability_id: str
    domain: str
    operation: Literal["search", "read", "create", "update", "transition", "delete"]
    target_entity: str
    description: str
    input_schema: dict
    output_schema: dict
    required_permissions: list[str]
    risk_level: Literal["low", "medium", "high", "critical"]
    approval_policy_ref: str | None = None
    api_binding_ref: str
    version: str
```

## 5. Capability 与原始 API 的关系

一个业务 Capability 可以由多个 API 组成：先搜索项目，再读取项目详情，再按项目 ID 查询学员，最后读取学员状态历史。这些步骤可以由高层 Capability / Workflow 封装，而不是让模型猜 API 调用顺序。

一个 API 也可能被多个 Capability 复用，但每个 Binding 的参数映射和权限必须明确。

## 6. Catalog 查询

Agent Runtime 暴露内部 Catalog Port：
- `search_domains(query, principal_scope)`；
- `search_entities(query, domain)`；
- `search_capabilities(intent, entity, operation)`；
- `describe_capability(capability_id, version)`；
- `resolve_binding(capability_id, version)`。

Search 返回精简摘要；Describe 才返回完整 Schema；Invoke 再次校验当前版本和权限。

## 7. 业务规则建模

关键规则不能只写在自然语言描述中。状态机允许的转换、字段条件、角色限制、生效时间、审批要求、幂等要求和业务校验器都应结构化。自然语言规则辅助理解，执行时由 Rule Validator / Domain Service 验证。

## 8. 数据来源与可信度

Catalog 条目记录 Source：OpenAPI、Pydantic Schema、Python Route / Service、SQL DDL、业务文档、人工审核。自动扫描只能产生候选；权限和写操作规则必须人工审核，不能仅凭注释推断。

## 9. 版本与运行时

每个 Run 固定 `catalog_snapshot_id`。新 Run 使用新 Snapshot；旧 Run 使用兼容版本，或在安全变更时被撤销。Tool Invoke 检查 Binding 仍可执行。

## 10. 质量指标

API Coverage、Entity Mapping Coverage、Capability Coverage、Human Review Coverage、Schema Drift Detection Recall、Broken Binding Count、Tool Selection Accuracy、Business Task Success。

## 11. API 数量大时的三层目录

当系统拥有数百或数千 API 时，建议将目录拆成三层：

1. Asset Layer：原始 API、Pydantic Schema、数据库表、Python 函数、权限注解。
2. Semantic Layer：业务 Domain、Entity、Relationship、Attribute、Business Rule。
3. Capability Layer：模型可理解的业务动作及其 API Binding。

LLM 主要使用 Semantic / Capability Layer。Asset Layer 只供 Compiler、管理员和执行器使用。

## 12. Entity Resolution

自然语言名称不一定唯一。实体解析流程：
- 识别 Entity Type；
- 检索候选；
- 应用 Tenant / ACL；
- 计算名称、别名、属性和关系匹配；
- 根据置信度决定自动选取或澄清；
- 将确认后的稳定 ID 保存为 Entity Reference。

高风险写操作不能只用模糊名称定位资源。

## 13. Capability 组合规则

复合 Capability 需定义：
- 输入依赖；
- 子 Capability 列表；
- 执行顺序；
- 是否允许并行；
- 中间结果 Schema；
- 错误传播策略；
- 补偿策略；
- 输出证据要求。

不要让 LLM 临时发明一个不存在的组合 API。

## 14. 字段级语义

每个 Attribute 应声明：
- data_type；
- display_name / aliases；
- description；
- classification；
- readable_by；
- writable_by；
- searchable；
- filterable；
- mask_policy；
- freshness_requirement。

这样 Tool Search、Context Compiler 和 Policy Engine 可以共用同一套语义元数据。

## 15. Catalog 与业务代码的关系

Catalog 是代码和业务规则的语义索引，不是业务规则最终执行点。真正的状态转换和约束仍由业务 Domain Service / Validator 执行。Catalog 发现规则变化后应触发 Drift Review，而不是直接修改生产业务逻辑。
