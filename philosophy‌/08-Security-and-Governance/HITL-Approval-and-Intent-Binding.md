# Human-in-the-Loop：审批与意图绑定

## 本文档在系统中的作用

HITL 将高风险 Agent 动作转化为“先提出意图、再由人确认、最后执行”的受控流程。它解决模型可以提出动作但不应自行承担最终授权的问题。

## 1. 哪些动作需要审批

按风险分类：Low（只读查询）；Medium（创建草稿、内部文档）；High（修改业务状态、发送外部通知、提交审批）；Critical（删除、资金、权限变更、不可逆外部动作）。具体分类由业务风险矩阵确定，不由模型决定。

## 2. Approval Request

approval_id、run_id、tool_execution_id、principal、action、resource_refs、before_snapshot、proposed_change、risk_level、intent_hash、expires_at、required_approver_policy、status。

## 3. Intent Binding

审批通过的是明确意图，不是宽泛“允许 Agent 操作”。`intent_hash` 覆盖 Tool Version、规范化参数、Resource Version、目标动作和关键约束。执行前重新计算并比较；不一致则重新审批。

## 4. 审批状态机

`requested → pending → approved / rejected / expired / cancelled / superseded`

审批只能由授权审批人操作。重复审批请求必须幂等。审批撤回后，Graph Resume 不得执行原动作。

## 5. Graph Interrupt

Graph 负责暂停；Approval Service 负责审批记录；Run Service 负责任务状态；Tool Executor 负责执行前再次校验。四者通过 approval_id 关联。

## 6. 双人复核与职责分离

Critical 动作可要求双人审批、申请人不能审批自己、审批人必须具备特定角色。审批策略按业务类型配置。

## 7. 审批内容展示

展示变更前后差异、影响对象、风险、来源和回滚方案。不能只展示“Agent 想执行某个 Tool”。

## 8. 测试

审批通过后参数变化、审批过期、审批人权限撤销、重复 Resume、审批请求重复创建、审批后资源版本变化、审批通过但 Tool 已撤销。
