# Interrupt、人工介入、Resume 与恢复

## 本文档在系统中的作用

Interrupt 允许 Graph 在特定节点暂停，等待用户澄清、人工审批或外部事件，再从持久化状态继续。它把 Agent 自主执行转为受控自动化。

## 1. 三类中断

- Clarification：缺少业务信息，例如项目范围、目标学员或时间区间。
- Approval：即将执行有副作用或高风险动作，需要审批。
- External Wait：等待第三方任务、Webhook 或长时间计算结果。

三类中断需要不同 API、状态和 SLA，不应全部表现为一个泛化 `waiting`。

## 2. Interrupt Payload

只包含 UI 和 Resume 必须的数据：interrupt_type、interrupt_id、run_id、summary、resource_refs、required_fields / approval_options、expires_at、schema_version。不包含密钥、内部 Prompt、完整 Principal、策略源码和未脱敏业务对象。

## 3. 审批与 Graph Interrupt 分离

Approval Service 是审批事实来源；LangGraph Interrupt 只是暂停执行机制。审批记录包含申请内容、资源、操作、风险、审批人、决定、时间和 intent hash。Graph 恢复前必须读取 Approval Service 并验证状态。

## 4. Resume API

Resume API 不接收任意 State。它接收 `interrupt_id` 和符合该中断 Schema 的 `response`。服务端验证当前用户是否有权回答、Interrupt 是否属于当前 Run、是否过期、是否已被消费、响应是否满足 Schema、Resource Version 是否变化。

## 5. 恢复时重新检查

中断期间权限可能被撤销、资源可能被修改、Tool 可能被禁用、Agent Release 可能被撤销。恢复后对下一次副作用执行必须重新授权和检查业务状态。

## 6. 外部 Webhook 恢复

Webhook 先验签、去重、写入 Inbox，再把 `external_event_id` 与 Run / Job 关联。不要在 Webhook HTTP Handler 中直接恢复 Graph 长任务。由 Worker 从持久化 Inbox 领取恢复任务。

## 7. 超时与过期

Clarification / Approval 需要 TTL。过期策略可自动取消、转人工队列、要求重新发起；默认不能在审批过期后自动执行。

## 8. 测试

重复 Resume、错误用户 Resume、过期 Resume、审批撤销后 Resume、Webhook 重放、Worker 重启后 Resume、Graph State 迁移失败。
