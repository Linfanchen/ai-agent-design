# Agent Engineering Architecture Handbook V4.0

> 面向生产级 Agent 平台的 Python 工程架构手册。目录和文件名使用英文，正文使用中文。

## 1. V4.0 定位

本手册不是 Prompt 技巧合集，也不是开源项目功能清单。它回答如何使用 Python 技术体系，把一个或多个 LLM 组织成能够理解任务、检索知识、发现能力、调用受控工具、保持状态、接受人工干预、从故障恢复，并可被评测、审计、部署和持续运营的 Agent 系统。

V4.0 是对 V3.0 的架构重构。核心实现统一为 Python，不再提供 TypeScript / Java 作为核心实现路径。外部系统可以通过 HTTP、gRPC、消息或 MCP 接入，但 Agent 平台内部以 Python 为主。

## 2. 技术基线

| 领域 | 技术基线 | 设计边界 |
|---|---|---|
| Python | Python 3.12+ | 统一类型、异步和运行时基线 |
| API | FastAPI + Pydantic V2 | 接入、身份、资源 API，不承载长任务执行 |
| 编排 | LangGraph Python | Graph 状态机、持久化、中断、恢复 |
| Harness | 自建 Python Composition Root | 装配 Provider、Context、Tools、Policy、Graph |
| 数据访问 | SQLAlchemy 2.x Async | 平台元数据与业务服务访问 |
| 数据库 | PostgreSQL | 控制面、事件、审计、记忆元数据、Catalog |
| 缓存 | Redis（可选） | 缓存、限流、轻量协调，不作为事实来源 |
| 向量检索 | pgvector / Milvus Adapter | 语义召回；权限过滤先于内容注入 |
| 异步执行 | asyncio + Worker Queue | I/O 并发与长任务解耦 |
| 模型 | Provider Adapter | OpenAI、Anthropic、Qwen、DeepSeek 等 |
| 工具协议 | Function Calling / MCP | 内部 Tool Contract 与外部协议分离 |
| 插件 | Python Plugin SDK | Entry Points + Manifest + 生命周期 |
| 沙箱 | Docker / 独立执行服务 | 不可信代码与主进程隔离 |
| 可观测性 | OpenTelemetry | Trace、Metrics、Logs、Trajectory |
| 测试 | pytest、pytest-asyncio、契约测试 | 单测、集成、故障注入、回归评测 |

版本基线是架构基准，不代表依赖版本永久固定。正式实施必须生成 lockfile，锁定 LangGraph / LangChain / MCP SDK 的兼容版本，并运行 Contract Tests。

## 3. 关键架构原则

1. 模型不拥有业务状态。应用负责 Session、Thread、Run、Checkpoint、Memory、Approval、Audit。
2. LangGraph 不等于 Harness。Graph 负责流程编排，Harness 负责依赖装配、策略、工具、上下文和生命周期。
3. 业务数据的权威性不属于 Memory 或 RAG。业务状态必须实时从权威业务 API / 数据库读取。
4. Tool 是受控能力，不是任意函数。每次执行都需要 Schema、身份、租户、策略、预算、幂等和审计。
5. 动态发现不是动态授权。Search 只找候选；Invoke 时必须重新鉴权和检查版本。
6. 长任务不能依赖 HTTP 请求生命周期。Run 进入持久化 Job，由 Worker 执行；SSE/WebSocket 只是事件订阅。
7. 并行不等于共享资源。每个并行任务必须有独立 SQLAlchemy AsyncSession、预算和取消边界。
8. 失败必须有确定语义。外部写操作超时要区分 FAILED 与 UNKNOWN，不能盲目重试。
9. 高风险动作必须可解释、可审批、可对账。模型不得自行提升权限或绕过审批。
10. 每项能力都必须可测试。Tool Contract、Graph Resume、权限拒绝、Memory Scope、Evaluation Regression 都要自动化。

## 4. 目录导航

- `00-Architecture`：总架构、设计原则、术语和边界。
- `01-Foundations`：Agent Definition、模型 Provider、Prompt、Python 异步基础。
- `02-Harness-and-Runtime`：Harness、Factory、Container、生命周期、Middleware、Agent Loop。
- `03-LangGraph-and-Orchestration`：StateGraph、Checkpoint、Interrupt、恢复、Multi-Agent。
- `04-Context-and-Memory`：Context Compiler、Token Budget、Memory Engine、RAG。
- `05-Business-Catalog`：业务资产扫描、语义目录、编译、漂移检测。
- `06-Tools-and-Discovery`：Tool Registry、Dynamic Discovery、Executor、事务和幂等。
- `07-Skills-Plugins-and-MCP`：Skills、Plugins、MCP、Sandbox。
- `08-Security-and-Governance`：Identity、RBAC/ABAC、Policy、Approval、Audit、Prompt Injection。
- `09-Evaluation-and-Observability`：Dataset、Evaluator、Judge、Trajectory、Tracing、Regression。
- `10-Application-and-Platform`：FastAPI、Job、事件、部署、资源管理。
- `11-Data-Model-and-Implementation`：PostgreSQL DDL、Python 项目骨架、端到端案例、路线图。
- `12-Reference-Projects`：AgentScope、DeerFlow、LangGraph、Octop、DeepSeek Harness、Hermes 的架构借鉴。
- `13-Appendices`：ADR、Checklist、术语、来源和版本策略。

## 5. 子系统文档统一标准

每个核心子系统回答：作用是什么；不负责什么；核心对象是什么；组件如何协作；生命周期是什么；数据如何流动；关键接口是什么；失败如何恢复；并发如何控制；权限如何执行；如何测试；如何部署；如何判断实现完成。

## 6. 阅读顺序

架构师：总蓝图 → Harness → LangGraph → Context → Tool → Security → Evaluation → 数据模型。
Runtime 工程师：Python 异步模型 → Harness Factory → Agent Loop → LangGraph → Checkpoint → Worker / Job → Event Stream。
能力工程师：Business Catalog → Tool Registry → Dynamic Discovery → Executor → Skills / Plugins / MCP。
知识工程师：Context Compiler → Token Budget → Memory → RAG → Evaluation。
平台工程师：FastAPI → Tenant / Resource → PostgreSQL → Observability → Deployment / SLO。

## 7. 重要声明

本手册是工程设计蓝图，不是已经通过执行验证的完整软件产品。代码示例用于表达边界和推荐实现。涉及特定库的 API 签名时，以项目锁定版本的官方文档和 Contract Test 为准。
## 8. V4.0 深化补充

本版本进一步补齐了工程实现层面的参考材料：
- Model Provider Adapter 的统一请求 / 响应契约和能力描述符；
- Agent Loop 的 Python 控制流程骨架、终止状态和 Loop Guard；
- LangGraph State Schema、Reducer、Checkpoint 和迁移参考；
- Context Token Budget 的可计算示例；
- Catalog Drift 的分类、Snapshot 发布、灰度和回滚；
- Tool Execution 状态机、UNKNOWN 对账、幂等和补偿；
- MCP Server 信任边界、SSRF 防护和连接生命周期；
- Approval Intent Binding、TOCTOU 和 Resume 二次校验；
- Evaluation Dataset / Trajectory JSON 格式；
- Run / Job 状态机与 Worker Crash 恢复；
- PostgreSQL 分区、索引、归档和保留；
- 首个 90 天的纵向切片实施路线。

## 9. 交付边界

压缩包包含 Markdown 架构文档、Python 参考代码片段、SQL DDL、Mermaid 时序图和测试验收清单。它是可供研发团队评审和拆任务的架构蓝图，不是已经安装依赖、通过集成测试并可直接部署的完整 Python 项目。正式开发需按锁定依赖版本补齐具体 Adapter、Repository、Worker、迁移和自动化测试。
