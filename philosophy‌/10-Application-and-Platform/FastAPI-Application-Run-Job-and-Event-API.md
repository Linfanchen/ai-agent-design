# FastAPI：Application、Run、Job 与 Event API

## 本文档在系统中的作用

FastAPI 层负责身份、资源、任务创建、状态查询、取消、Resume 和事件订阅。它不应在一个 HTTP Handler 内执行数分钟 Agent Graph。

## 1. API 资源

Agent：可用 Agent / Release；Conversation：用户可见对话容器；Thread：Graph 执行连续性；Run：一次任务；Job：后台调度单元；Approval：待审批动作；Artifact：生成文件或结果；Event：运行事件。

## 2. 推荐接口

- `POST /v1/agents/{agent_key}/runs`：创建 Run；
- `GET /v1/runs/{run_id}`：读取状态；
- `POST /v1/runs/{run_id}/cancel`：取消；
- `POST /v1/runs/{run_id}/resume`：提交 Interrupt 响应；
- `GET /v1/runs/{run_id}/events`：SSE；
- `GET /v1/threads/{thread_id}/runs`：历史 Run；
- `GET /v1/artifacts/{artifact_id}`：受控下载；
- `POST /v1/approvals/{approval_id}/decision`：审批。

## 3. 创建 Run

请求包含 input、thread_id（若继续对话）、idempotency_key、attachment_refs、client_metadata。Tenant、Principal、权限和 Model Profile 不允许由客户端任意指定。

返回 run_id、status、events_url、created_at。若 Run 已存在且幂等键相同，返回同一个 Run。

## 4. Worker 解耦

API 在事务中创建 Run + Job + Outbox，然后立即返回。Worker 使用 `FOR UPDATE SKIP LOCKED` 或消息队列领取 Job，使用 Lease / Heartbeat 防止僵尸任务。

## 5. SSE

SSE 事件需要 event_id / sequence、event_type、run_id、timestamp、payload。客户端断线后通过 Last-Event-ID 或 sequence 重连。事件必须有持久化读取路径，不能只靠进程内 asyncio Queue。

## 6. FastAPI Lifespan

在 lifespan 中初始化 Engine、HTTPX AsyncClient、Redis Pool、Telemetry、Registry Snapshot Cache。Shutdown 时先停止接收新 Run，再等待或取消 Worker 内任务，最后关闭资源。

## 7. API 安全

每个 Run、Thread、Artifact、Approval 的读取都必须做 Tenant + Resource ACL 检查。知道 UUID 不代表拥有资源。Resume API 只接受指定 Interrupt 的结构化响应，不接受任意 State。

## 8. 测试

使用 FastAPI TestClient / AsyncClient、Dependency Overrides、Lifespan 测试。注意 AsyncClient 本身不一定自动触发应用 Lifespan，测试需要显式管理 Lifespan。
