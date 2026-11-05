# 端到端业务 Agent 案例：查询项目、查询学员、更新状态

## 本文档在系统中的作用

将 V4 模块串联为一个业务案例，展示 Agent 如何理解任务、发现 Capability、读取实时数据、消歧、审批、执行状态变更并验证结果。

## 1. 用户任务

“查询当前有哪些导师带教项目，找到项目 A 中的学员小李，查看他的状态，如果还在进行中且符合规则，把状态更新为已完成。”

## 2. 任务拆解

识别 Domain mentorship；识别 Entity project / learner；识别 Operation search / read / transition；搜索项目列表 Capability；查询项目详情和学员列表；通过稳定业务 ID 消歧“小李”；实时读取学员状态和状态历史；检查状态机允许转换；根据风险策略判断审批；创建状态变更 Tool Execution；幂等执行；再次读取状态验证；返回证据。

## 3. 不应直接暴露的接口

不要把 `GET /api/v1/table/{table}`、`POST /sql/execute` 或任意 CRUD 暴露给模型。应定义语义明确的 Capability：
- `mentorship.project.search`
- `mentorship.project.get`
- `mentorship.learner.search`
- `mentorship.learner.get`
- `mentorship.learner.status.transition`
- `mentorship.learner.status.history`

## 4. Tool Input 示例

```python
from pydantic import BaseModel, Field
from typing import Literal

class ProjectSearchInput(BaseModel):
    keyword: str | None = None
    status: list[str] = Field(default_factory=list)
    page: int = Field(default=1, ge=1, le=1000)
    page_size: int = Field(default=20, ge=1, le=100)

class LearnerSearchInput(BaseModel):
    project_id: str
    keyword: str = Field(min_length=1, max_length=100)

class LearnerStatusTransitionInput(BaseModel):
    learner_id: str
    target_status: Literal["completed", "paused", "active"]
    expected_current_status: str
    reason: str = Field(min_length=3, max_length=500)
    expected_resource_version: int
```

## 5. 实体消歧

“小李”可能匹配多名学员。Agent 不应随意选第一条。返回候选的最小必要信息（项目、部门、员工编号后几位等，具体取决于权限），要求用户确认。确认结果保存为 Entity Reference，而不是只保存名字。

## 6. 状态变更规则

状态迁移由业务 Domain Service / API 执行。Agent 只能提出 `target_status`。执行前校验当前状态仍等于 expected_current_status、资源版本未变化、状态机允许转换、用户有操作权限、目标状态满足前置条件、审批有效（若需要）。

## 7. Graph 设计

```text
START
  → classify_request
  → search_capabilities
  → resolve_project
  → search_learner
  → disambiguate_learner
  → read_current_status
  → evaluate_transition
  → approval_interrupt (conditional)
  → execute_transition
  → verify_status
  → compose_answer
  → END
```

## 8. 写操作安全

Tool Executor 生成稳定 Idempotency Key。审批绑定参数 Hash。执行时使用 expected_resource_version 做乐观锁。若 API 超时，进入 UNKNOWN 并查询状态历史，不直接重复更新。

## 9. 最终回答必须有证据

回答找到哪个项目、学员身份如何确认、更新前状态、更新后状态、状态变更记录 ID、执行时间、是否经过审批。若验证失败，必须说明“请求已提交但尚无法确认”，不能说“已成功更新”。

## 10. 测试场景

同名学员、项目权限不足、状态已被他人更新、审批拒绝、审批过期、Tool 超时但业务成功、Tool 超时且业务失败、重复请求、Catalog 版本过期、用户中途取消。
