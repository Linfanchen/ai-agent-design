# Agent 系统总体架构蓝图

## 本文档在系统中的作用

定义整个平台的总边界：Agent 产品由哪些层组成、请求如何经过系统、哪些状态由谁负责，以及为什么不能把所有功能写进一个 Agent 类或一张 Graph。

## 1. 先区分三个系统

### Agent Application
面向用户的应用服务，负责身份认证、租户、Conversation、Thread、Run、Job、Artifact、审批入口、事件订阅和资源访问。

### Agent Harness
面向执行的运行时外壳，负责把模型、Prompt、Context、Memory、Tools、Policy、Skills、Plugins、Graph 和 Observability 组合成一个可控执行环境。

### Orchestration Runtime
负责流程推进、节点执行、条件路由、并行任务、暂停恢复和 Checkpoint。V4 选择 LangGraph Python 作为主要编排引擎，但不把其 State 当作全部业务数据库。

## 2. 七层架构

```text
Experience: Web / API / CLI / IM / Scheduled Trigger
Application: Identity / Tenant / Conversation / Run / Job
Harness: Factory / Container / Middleware / Runtime Scope
Orchestration: LangGraph / Loop / Subgraph / Planner
Knowledge: Prompt / Context / Memory / RAG / Business Catalog
Capabilities: Tool / Skill / Plugin / MCP / Sandbox
Governance & Ops: Policy / Approval / Audit / Eval / Trace
       PostgreSQL / Redis / Object Storage / Vector Store
```

## 3. 一次 Agent Run 的完整路径

1. API 收到请求并验证认证凭证。
2. Identity Resolver 构造可信 Principal；Tenant Resolver 确定租户范围。
3. Application Service 检查 Agent Release、配额和 Thread 并发策略。
4. 创建 Run 与 Job，写入幂等键和请求摘要。
5. Worker Claim Job，创建 Run Scope。
6. Runtime Factory 装配 Provider、Context Compiler、Tool Registry、Policy、Memory、Graph。
7. Context Compiler 读取历史、用户输入、记忆候选、业务 Catalog 和实时业务数据引用。
8. Graph / Agent Loop 调用模型。
9. 模型请求工具时，Tool Discovery / Registry 提供候选和 Schema。
10. Tool Executor 在执行前重新鉴权、策略判断、参数校验、审批检查和幂等检查。
11. Tool 结果以结构化 Observation 回到 Graph State，并写入事件和审计。
12. Graph 继续执行，必要时 Interrupt 等待澄清或审批。
13. 最终结果通过 Output Validator 校验，保存 Artifact / Run Result。
14. Run 状态变为完成；事件通过 SSE / WebSocket / Channel 投影给客户端。
15. Trace、Usage、Trajectory 和 Evaluation 样本被记录或异步处理。

## 4. 关键所有权边界

| 对象 | 权威所有者 | 不应由谁代替 |
|---|---|---|
| 用户身份 | Identity Provider / Application | LLM |
| 租户与权限 | Policy / IAM | Prompt |
| 业务实体状态 | 业务 API / 业务数据库 | Memory / Vector DB |
| Agent 定义 | Control Plane | Graph State |
| Run / Job 状态 | Application DB | 进程内变量 |
| Graph 执行状态 | LangGraph Checkpointer | 前端 |
| 会话事件 | Event Store | LLM Provider |
| 长期记忆 | Memory Service | 原始聊天全文 |
| Tool 定义 | Tool Registry | 模型输出 |
| 审批决定 | Approval Service | 模型 |
| 执行审计 | Audit Service | 普通应用日志 |

## 5. 模块间调用原则

- API 不直接执行长时间 Graph，只创建 Run 并返回 Run ID。
- Graph Node 不直接访问 FastAPI Request。
- Graph Node 不持有全局 AsyncSession；通过 Unit of Work / Repository Factory 获取短事务。
- Model Adapter 不执行 Tool，只归一化模型响应。
- Tool Executor 不决定模型下一步做什么，只执行被授权能力。
- Memory 不作为业务数据的权威来源。
- Plugin 不允许任意修改宿主核心对象，只能通过稳定 SDK 注册扩展。

## 6. 部署边界

推荐至少拆分 `api`、`worker`、`scheduler`、`indexer`、`sandbox`、`postgres`、`redis` 和 `object-storage`。初期可采用同仓库模块化单体部署，但 API 与长任务 Worker 应具有独立进程和资源池。

## 7. 架构验收

- API 进程重启不丢失已接受 Run。
- Worker 重启后可恢复可恢复的 Graph。
- 一个租户无法读取另一个租户的 Thread、Memory、Artifact。
- Tool 在执行时重新校验权限。
- 外部写操作超时进入 UNKNOWN 对账，而不是自动重复。
- 所有高风险写操作有审批或明确的低风险豁免策略。
- 每次 Run 可追踪到 Model Call、Tool Execution、Approval 和最终结果。
