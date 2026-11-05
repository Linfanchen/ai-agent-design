# Tracing、Metrics、Logs 与 Trajectory

## 本文档在系统中的作用

可观测性用于解释 Agent 为什么慢、为什么失败、为什么调用某个工具、成本在哪里产生，以及质量是否发生变化。Agent Trace 需要跨越 API、Worker、Graph、Model、Tool、MCP、Sandbox 和数据库。

## 1. Trace 层级

`HTTP Request → Run → Graph Super-Step → Node → Model Call / Tool Execution → External Dependency`

每个 Span 保存 trace_id、run_id、thread_id、tenant_id（按脱敏策略）、agent_release_id、node_name、provider、tool_key、latency、status。

## 2. Metrics

Runs Started / Completed / Failed；Queue Wait；Time to First Token / First Useful Event；Model Latency；Tool Latency；Checkpoint Latency；Context Compile Latency；Approval Wait；Token Usage；Cost；Retry Count；UNKNOWN Execution Count；Recovery Success；Evaluation Score。

## 3. Logs

使用结构化日志和 Correlation ID。禁止记录密钥、Cookie、完整敏感 Prompt、原始个人数据。需要调试内容时使用受控、加密、短保留诊断存储。

## 4. Event 与 Trace 的关系

Event Log 是持久化业务执行事实；Trace 是性能和调用链视图；Metrics 是聚合指标；Trajectory 是可用于评测的任务级行为记录。四者通过 ID 关联，但不互相替代。

## 5. 采样

普通成功 Run 可采样 Trace；错误、高风险写操作、UNKNOWN 和审批事件应提高采样或全量记录关键摘要。采样策略不能影响 Audit 完整性。

## 6. 告警

Run Queue 持续积压；Provider 429 / 5xx 激增；Checkpoint 失败；UNKNOWN Tool 增加；Approval 未处理积压；Tenant 配额异常；Cross-tenant Deny 异常；Memory / Catalog Index Lag；Sandbox 孤儿容器；Cost per Successful Run 异常上升。

## 7. 关联 ID 规范

Run ID 是任务主关联键；Trace ID 关联跨服务调用；Tool Execution ID 关联副作用；Provider Request ID 关联模型请求；Event Sequence 用于有序回放；Delegation ID 关联父子 Agent。不要用单个 trace_id 替代持久业务 ID。

## 8. 采样与成本

成功短任务可采用低比例 Trace Sampling；失败、审批、UNKNOWN、越权拒绝和安全告警应提高采样。Metrics 和 Audit 不依赖 Trace Sampling。高基数标签（如完整用户输入、任意资源 ID）不要直接作为 Prometheus Label。

## 9. 日志分级

- INFO：Run 状态变化、步骤摘要；
- WARN：重试、降级、超时、部分失败；
- ERROR：不可恢复失败；
- SECURITY：授权拒绝、越权尝试、Injection 告警；
- AUDIT：结构化持久事实。

敏感信息脱敏在 Logger Handler 之前完成，不能仅依赖开发人员记得不打印。

## 10. Dashboard

至少按 Tenant、Agent Release、Task Type、Provider、Tool、Worker Queue 展示成功率、P95 延迟、成本、重试、UNKNOWN、审批等待、恢复率和评测分数。
