# Event Streaming 与客户端协议

## 本文档在系统中的作用

Event Protocol 将内部 Graph / Tool / Approval / Run 事件统一投影给 Web、CLI、IM 或其他客户端。不同入口看到同一任务事实，不需要理解 LangGraph 内部事件格式。

## 1. Event Envelope

event_id、run_id、thread_id、sequence、type、occurred_at、schema_version、visibility、payload、correlation_id。

## 2. 事件类型

run.queued / started / completed / failed；message.delta / message.completed；plan.updated；tool.started / completed / failed；approval.required / updated；clarification.required；artifact.created；usage.updated；run.cancelled。

## 3. 内部事件与用户事件

内部事件可包含 Node 名称、错误分类和调试信息；用户事件只暴露必要状态和安全摘要。不要直接把 Python Exception 或完整 Tool 参数发给客户端。

## 4. SSE 重连

服务端按 `(run_id, sequence)` 持久化并分页读取。客户端保存 Last Event ID；断线重连后补齐缺失事件。若事件已过保留期，返回 snapshot + latest_sequence。

## 5. 顺序与重复

sequence 在 Run 内单调递增。客户端支持重复事件幂等消费；不能仅按 timestamp 排序。事件发布通过 Outbox 避免 DB Commit 与消息投递之间丢失。

## 6. 事件 Schema 版本

事件 payload 向后兼容优先；客户端忽略未知 event type；删除字段需要弃用周期。Graph 内部事件变化不能直接导致外部协议不兼容。

## 7. Event Persistence 与 Outbox

Run 状态更新和 Event 写入应尽可能在同一数据库事务提交。若需要推送到 Redis Pub/Sub 或 WebSocket Hub，通过 Outbox Worker 投递。Pub/Sub 只用于唤醒订阅者，历史回放仍以持久 Event Store 为准。

## 8. Snapshot 与事件压缩

事件数量过多时可生成 Run Snapshot，包含当前状态、latest_sequence 和关键结果引用。客户端先获取 Snapshot，再从其 sequence 之后继续订阅。Snapshot 不删除 Audit，也不代表 Checkpoint。

## 9. SSE 安全

订阅连接建立时校验 Run 归属和权限；每次重连重新校验；不允许只凭 run_id 订阅。事件 Payload 按用户可见级别过滤，内部错误和敏感参数不发送给普通客户端。

## 10. 背压

慢客户端不能拖垮 Worker。SSE 推送应有有限缓冲区；溢出时关闭连接并要求客户端按 Last-Event-ID 回放。不能无限缓存 Token Delta。
