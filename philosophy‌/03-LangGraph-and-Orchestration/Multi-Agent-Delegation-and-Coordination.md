# Multi-Agent：委派、协作、并行与恢复

## 本文档在系统中的作用

Multi-Agent 通过多个职责受限的 Agent 分担任务，适用于复杂任务拆分、并行检索、专业分析和结果汇总。它会增加 Token 成本、执行延迟、权限传播和故障面，因此不是默认架构。

## 1. 常见拓扑

- Supervisor：主 Agent 规划并委派子任务。
- Router：按任务类型路由到一个专家。
- Pipeline：多个 Agent 按固定阶段串行协作。
- Parallel Fan-out / Fan-in：多个子 Agent 并行完成独立任务。
- Hierarchical：多层委派，必须设置深度上限。

企业场景优先使用 Supervisor + 有界子 Agent；固定流程用 Pipeline，不要用自由对话模拟确定工作流。

## 2. 父子对象

Parent Run 与 Child Run 独立持久化。Child Run 具有 parent_run_id、delegation_id、assigned_agent_release_id、delegated_scope、input_ref、output_schema、budget_reservation、deadline、cancellation_policy、result_status。

子 Agent 不能直接读取父 Agent 全部消息历史，只接收任务所需的最小上下文包。

## 3. 委派协议

```python
from pydantic import BaseModel, Field
from typing import Literal

class DelegationRequest(BaseModel):
    task: str = Field(min_length=1, max_length=4000)
    objective: str
    input_refs: list[str] = Field(default_factory=list)
    allowed_capability_tags: list[str] = Field(default_factory=list)
    output_schema_ref: str
    max_steps: int = Field(default=8, ge=1, le=30)
    max_cost_usd: float = Field(default=0.25, gt=0)
    deadline_seconds: int = Field(default=120, ge=5, le=1800)
    mode: Literal["parallel", "sequential"] = "parallel"
```

所有 `input_refs` 必须重新做 Tenant / ACL 校验。

## 4. 权限委派

父 Agent 只能下放自己拥有且允许委派的权限，不能扩大权限。推荐 Capability Grant：child principal、tenant_id、resource scope、allowed actions、expires_at、parent delegation id、policy version。

子 Agent 每次 Tool Invoke 仍需 Policy Engine 验证。父任务权限撤销时，尚未完成子任务按策略取消或缩减权限。

## 5. 并发与预算

Fan-out 前预留子任务预算，限制最大 Fan-out 数、并发数、深度和总成本。对相同资源的写操作不允许并行。不同子任务使用独立 DB Session、独立 Tool Context 和独立 Trace Span。

## 6. 结果协议

子 Agent 返回结构化结果：status、summary、output_ref、evidence_refs、warnings、unresolved_questions、usage、error_code。主 Agent 验证 Schema、来源和证据，再合并。不要把子 Agent 自由文本直接作为可信指令注入主 Prompt。

## 7. 失败策略

一个子任务失败时按关键性决定继续或整体失败；超时则取消或标记部分成功；无法取消时保留 cancel_requested；输出 Schema 不合法时隔离结果并有限重试；子 Agent 请求扩大权限时拒绝并审计；预算耗尽返回 budget_exhausted，不自动追加预算。

## 8. 避免反模式

Agent 数量越多不代表能力越强；不让多个 Agent 对同一资源无协调写入；不复制全部聊天历史；不允许无限递归委派；子 Agent 不得自行选择未授权模型或 Tool；不能只靠主 Agent 总结判断结果正确。

## 9. 评测

比较单 Agent 与 Multi-Agent 在任务成功率、延迟、成本、事实一致性、工具错误率和人工修正率上的差异。只有质量收益超过额外成本，才保留多 Agent 设计。

## 10. Delegation 的完整状态机

`proposed → validated → authorized → queued → running → waiting_input / waiting_approval → succeeded / partially_succeeded / failed / cancelled / expired`

父 Run 不能只通过一个布尔值记录 Child 是否完成。每个 Delegation 有独立状态、Attempt、预算、权限 Grant 和 Result Reference。

## 11. Child Context Package

只传递：
- 目标和完成标准；
- 相关实体引用；
- 必要证据引用；
- 可用 Capability；
- 输出 Schema；
- 时间和成本预算；
- 明确禁止事项。

不要复制父 Agent 全部对话历史、所有 Memory 和所有 Tool Result。Child 若需要更多上下文，应通过受控 Query / Capability 请求。

## 12. Fan-in 汇总策略

汇总时按任务类型选择：
- Merge：结果互补；
- Vote：多个 Agent 独立判断；
- Evidence Aggregation：合并证据后由主 Agent 重新判断；
- Conflict Escalation：冲突时要求更多证据或人工决策。

不要简单采用多数票解决事实问题。对于真实业务状态，始终以权威业务 API 为准。

## 13. 防止递归爆炸

限制 `max_delegation_depth`、`max_child_runs`、`max_total_model_calls`、`max_total_cost` 和 `max_wall_time`。每次委派必须有父子关系和预算预留。子 Agent 不允许自动无限创建孙 Agent。

## 14. Multi-Agent 的选择门槛

只有当任务能明显拆分、子任务输入输出清晰、并行收益可测、结果可验证、预算足够时使用 Multi-Agent。若只是把一个复杂 Prompt 拆成多个角色轮流发言，通常不值得引入多个 Runtime。
