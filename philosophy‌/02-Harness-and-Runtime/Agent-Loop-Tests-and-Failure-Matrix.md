# Agent Loop 测试矩阵与故障分析

## 本文档在系统中的作用

把 Agent Loop 可靠性转化为可自动化验证的场景。测试不能只验证“问一句话能回答”，必须覆盖多轮工具调用、部分失败、取消、预算和恢复。

## 1. 测试类别

Happy Path：无工具、单工具、多工具、多轮工具。
Routing：Final、Tool、Clarification、Approval、Retry、END。
Budget：Step、Token、Cost、Runtime、Tool Count。
Concurrency：并行读、写冲突、限流、父任务取消。
Recovery：Checkpoint 后进程退出、恢复、重复事件。
Security：越权工具、伪造 Principal、Prompt Injection。
Idempotency：重复 Run 请求、重复 Tool Call、超时对账。
Provider：429、5xx、无效流、流中断、Usage 缺失。

## 2. 典型故障矩阵

| 故障点 | 期望行为 |
|---|---|
| 模型响应缺失 | 记录 Provider Error，按有限策略重试 |
| Stream 中断 | 按 Provider 语义续传或重新请求 |
| Tool 参数不合法 | 不执行，返回 Schema Error |
| Tool 权限拒绝 | 不执行，返回安全拒绝 |
| Tool 超时但外部已提交 | 标记 UNKNOWN 并对账 |
| Graph Checkpoint 失败 | 不宣称任务已持久化 |
| Worker 重启 | 由 Lease 超时和 Checkpoint 恢复 |
| 用户取消 | 停止新步骤并清理资源 |
| 多次相同调用 | Loop Guard 阻止无限重复 |
| 超出预算 | 终止或降级，不继续消耗 |

## 3. Golden Scenario

只读查询；先澄清再查询；多个只读 Tool 并行；写操作需审批；外部写操作成功但响应丢失；子 Agent 部分失败；Context 超长压缩；Memory 冲突；用户权限中途撤销；Run 在 Checkpoint 后重启恢复。

## 4. 验收指标

Task Completion Rate、Tool Selection Accuracy、Tool Execution Success、Duplicate Side Effect Rate、Unauthorized Execution Rate、Mean / P95 Steps、Cost per Successful Run、Recovery Success Rate、No-progress Termination Rate。

## 5. Unit Test / Integration Test 边界

Unit Test 用 Fake Provider、Fake Tool Executor 和 Fake Clock，验证 Loop 状态机，不访问真实外部系统。Integration Test 验证 Provider Adapter、数据库持久化、Checkpointer 和 Event Store 的组合。Evaluation Test 衡量任务结果，不替代单元测试。

## 6. Property-Based Test

可对随机 Tool Call 序列验证：
- 任何 Deny 都不会进入 Handler；
- 超过 Budget 后不会再发起新调用；
- 相同幂等键和参数不会重复执行；
- 相同幂等键不同参数必定冲突；
- Cancel 后不再创建新的 Tool Execution；
- No-progress 达到阈值必定终止。

## 7. Chaos Test

在 Tool 调用前、请求发出后、外部执行成功后、结果写库前、Checkpoint 写入前后分别杀掉 Worker，观察恢复后的执行状态。测试重点是“有没有重复副作用”，不是只看任务能否继续。

## 8. 测试数据治理

测试日志不能记录生产密钥或敏感参数。失败样本脱敏后才能进入 Golden Dataset。对随机模型输出，固定 Fixture 和预算，使用多次采样区分模型波动和真实回归。
