# 生产部署、SLO、容量模型与故障演练

## 本文档在系统中的作用

把 Agent 从“本地能运行”转化为“生产环境可用、可恢复、可扩展、成本可控”。它定义进程划分、容量估算、SLI、队列隔离和故障演练。

## 1. 推荐进程划分

- API：FastAPI，处理身份、Run 创建、查询、取消、Resume、SSE。
- Agent Worker：执行 LangGraph / Agent Run。
- Scheduler：Cron / Delay 任务创建。
- Index Worker：RAG / Catalog / Memory 索引。
- Sandbox Worker：隔离代码执行和文件转换。
- Evaluation Worker：离线评测与回归。
- Outbox Worker：事件投递和对账。

## 2. SLI

Invocation Availability、Run Completion Rate、Resume Success、Event Delivery、Tool Execution Reliability、Checkpoint Durability、Approval Integrity、Tenant Isolation。

租户隔离和未授权执行目标必须为零；不能用普通服务可用性 SLO 为安全违规设置“可接受比例”。

## 3. 延迟分解

Run Latency = Queue Wait + Runtime Build + Context Compile + Model Calls + Tool Calls + Checkpoint + Event Delivery + Final Validation。

交互任务关注 Time-to-First-Useful-Event；后台任务关注 End-to-End Completion。Approval Wait 单独统计，不混入计算耗时。

## 4. 容量估算

设并发 Run 为 C、每 Run 平均模型调用数为 M、平均模型响应时间为 L、Tool 并发为 T、Worker 槽为 W。模型并发近似取决于同时活跃的 Model Call 数；队列等待由到达率、服务时间分布和 Worker 并发共同决定。压测必须观察 P95 / P99，不能只看平均值。

## 5. 队列资源池

Interactive、Long Running、Sandbox、Indexing、Evaluation、Memory Consolidation、Scheduled Jobs 分离。一个耗时任务不能堵塞所有交互请求。

## 6. 背压与降级

队列、Provider、Sandbox 达到上限时，排队、拒绝或降低低优先级任务。不能通过跳过 Policy / Approval 来降级。可选 Memory / RAG 不可用时按任务风险决定是否降级。

## 7. 故障演练

Provider 429 / 5xx、PostgreSQL 主从切换、Redis 不可用、Worker Kill -9、Lease 过期、Checkpoint 写入失败、Outbox 堵塞、MCP Server 不可用、Sandbox 容器失联、权限服务不可用、SSE 断线恢复。

## 8. 容量报告

至少包含吞吐、P95 / P99、队列积压、DB 连接数、Provider 并发、Token 成本、Sandbox 峰值、恢复耗时、失败分布和容量瓶颈。
