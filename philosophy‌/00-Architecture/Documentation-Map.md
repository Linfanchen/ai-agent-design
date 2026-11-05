# 文档导航与模块深度索引

## 本文档在系统中的作用

这是 V4.0 的导航入口，帮助读者按架构角色、工程模块和实施顺序查找文档。每个模块不只一篇概念说明，还配有算法、接口、数据模型、测试或实现文档。

## 1. 目录结构

| 目录 | 核心内容 | 适合角色 |
|---|---|---|
| `00-Architecture` | 总体蓝图、设计原则、依赖地图 | 架构师 / 技术负责人 |
| `01-Foundations` | Agent Definition、模型 Provider、Prompt、asyncio | 全体工程师 |
| `02-Harness-and-Runtime` | Harness、Factory、DI、生命周期、Middleware、Loop | Runtime 工程师 |
| `03-LangGraph-and-Orchestration` | StateGraph、Checkpoint、Interrupt、Multi-Agent | 编排工程师 |
| `04-Context-and-Memory` | Context Compiler、Token Budget、Memory、RAG | 知识工程师 |
| `05-Business-Catalog` | 业务实体、Capability、Catalog Compiler、漂移检测 | 平台 / 集成工程师 |
| `06-Tools-and-Discovery` | Registry、Dynamic Discovery、Executor、幂等事务 | 能力工程师 |
| `07-Skills-Plugins-and-MCP` | Skill、Plugin、MCP、Sandbox | 扩展平台工程师 |
| `08-Security-and-Governance` | Identity、RBAC/ABAC、Policy、Approval、Injection、Audit | 安全 / 平台工程师 |
| `09-Evaluation-and-Observability` | Dataset、Trajectory、Evaluator、Judge、Regression、Trace | 质量工程师 |
| `10-Application-and-Platform` | FastAPI、Worker、Queue、Event、部署和 SLO | 后端 / 运维工程师 |
| `11-Data-Model-and-Implementation` | PostgreSQL、代码契约、端到端案例、实施路线 | 实施团队 |
| `12-Reference-Projects` | 六个开源项目架构拆解和采用矩阵 | 架构师 |
| `13-Appendices` | ADR、评审清单、术语和官方来源 | 全体工程师 |

## 2. 用户指定模块的深化位置

| 模块 | 核心设计 | 算法 / 实现 / 配套文档 |
|---|---|---|
| Harness | `02-Harness-and-Runtime/Harness-Architecture-and-Composition.md` | Factory、Container、DI、装配顺序、生命周期、隔离 |
| Agent Loop | `02-Harness-and-Runtime/Agent-Loop-Engine.md` | 并行 Tool、Retry、Budget、No-progress、故障矩阵 |
| LangGraph | `03-LangGraph-and-Orchestration/LangGraph-Python-StateGraph-Implementation.md` | Graph Pattern、Reducer、Checkpoint、Interrupt、Migration |
| Context | `04-Context-and-Memory/Context-Compiler-Architecture.md` | Priority、Token Budget、Selection、Compression、Reference Implementation |
| Memory | `04-Context-and-Memory/Memory-Engine-Architecture-and-Data-Model.md` | Extraction、Scoring、Conflict、Merge、Forget、Retrieval |
| Business Catalog | `05-Business-Catalog/Business-System-Intelligence-and-Catalog.md` | Python AST Scanner、API / Schema Mapping、Entity / Capability、Drift |
| Tool Registry | `06-Tools-and-Discovery/Tool-Registry-Schema-and-Versioning.md` | Dynamic Discovery、Schema、Executor、Policy、Idempotency |
| Multi-Agent | `03-LangGraph-and-Orchestration/Multi-Agent-Delegation-and-Coordination.md` | Supervisor Protocol、Budget、Permission Delegation、Fan-out / Fan-in |
| Skills | `07-Skills-Plugins-and-MCP/Skills-System-SKILL-md-and-Progressive-Disclosure.md` | Package Loader、Front Matter、Path Validation、Progressive Loading |
| Plugin | `07-Skills-Plugins-and-MCP/Plugin-System-Manifest-SDK-Lifecycle.md` | Host API、Scope、Registration、Unload、Isolation |
| Security | `08-Security-and-Governance/Identity-Tenant-RBAC-ABAC.md` | PDP / PEP、Approval、Prompt Injection、Audit / Privacy |
| Evaluation | `09-Evaluation-and-Observability/Evaluation-Dataset-Trajectory-and-Golden-Cases.md` | Evaluator、Judge、Regression、Release Gate、Trace |

## 3. 推荐阅读路径

### 架构师
1. System Architecture Blueprint；
2. Design Principles；
3. Harness Architecture；
4. LangGraph Python；
5. Context Compiler；
6. Business Catalog；
7. Tool Executor；
8. Security；
9. Evaluation；
10. Reference Project Comparison。

### Runtime 工程师
1. Python Async；
2. Harness Factory；
3. Agent Lifecycle；
4. Agent Loop；
5. LangGraph；
6. Checkpoint / Interrupt；
7. Worker / Lease；
8. Event Protocol。

### 工具与业务集成工程师
1. Business Catalog；
2. Catalog Compiler；
3. Tool Registry；
4. Dynamic Tool Discovery；
5. Tool Executor；
6. Identity / Policy；
7. End-to-End Business Agent Case。

### 知识工程师
1. Context Compiler；
2. Token Budget；
3. Memory Engine；
4. Memory Governance；
5. RAG；
6. Evaluation Dataset。

### 安全与质量工程师
1. Identity / RBAC / ABAC；
2. Policy Engine；
3. HITL Approval；
4. Prompt Injection；
5. Audit / Privacy；
6. Evaluation Runner；
7. Regression Gate。

## 4. 文档阅读的检查问题

阅读每篇文档时必须回答：这个子系统的职责是什么？不负责什么？由谁创建？谁持有生命周期？状态在哪里？接口如何定义？失败如何恢复？权限如何执行？哪些数据可进入模型？如何测试？上线门槛是什么？
## 5. 工程实现专项文档

| 主题 | 文档 | 解决的问题 |
|---|---|---|
| Model Adapter | `01-Foundations/Model-Provider-Adapter-Reference-Implementation.md` | 统一模型请求、响应、能力和错误 |
| Agent Loop | `02-Harness-and-Runtime/Agent-Loop-Reference-Implementation.md` | Loop 状态机和 Python 控制流程 |
| Graph State | `03-LangGraph-and-Orchestration/State-Schema-Reducer-and-Checkpoint-Reference.md` | State、Reducer、Checkpoint、迁移 |
| Context Budget | `04-Context-and-Memory/Context-Token-Budget-Worked-Example.md` | Token 预算、硬约束、压缩 |
| Catalog Drift | `05-Business-Catalog/Catalog-Drift-Detection-and-Release-Workflow.md` | 业务 API 漂移和版本发布 |
| Tool Execution | `06-Tools-and-Discovery/Tool-Execution-State-Machine-and-Idempotency.md` | 幂等、UNKNOWN、对账、补偿 |
| MCP Security | `07-Skills-Plugins-and-MCP/MCP-Trust-Boundary-and-Server-Management.md` | MCP Server 信任和网络边界 |
| Approval | `08-Security-and-Governance/Approval-Intent-Binding-and-TOCTOU.md` | 审批绑定、竞态和安全恢复 |
| Evaluation Format | `09-Evaluation-and-Observability/Evaluation-Dataset-Format-and-Trajectory-Schema.md` | Dataset 与 Trajectory 结构 |
| Run Recovery | `10-Application-and-Platform/Run-State-Machine-and-Recovery.md` | Run / Job 状态和恢复 |
| Database Scale | `11-Data-Model-and-Implementation/Database-Partitioning-Indexes-and-Retention.md` | PostgreSQL 扩展和保留 |
| 90-Day Plan | `00-Architecture/05-Implementation-Readiness-and-First-90-Days.md` | 实施优先级与验收 |

## 6. 目录维护原则

新增模块文档后，必须同步更新 README、Documentation Map 和 File Manifest。文档文件名使用英文 kebab-case，正文使用中文；示例代码必须标明参考实现、伪代码或生产注意事项。
