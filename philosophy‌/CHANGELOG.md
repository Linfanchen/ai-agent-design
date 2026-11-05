# CHANGELOG

## V4.0 — 2026-11-03

### 架构重构
- 核心实现统一为 Python 3.12+ 技术体系；
- 移除 V3 中 TypeScript / Java 核心实现路线；
- 明确 Agent Application、Harness、Orchestration、Knowledge、Capabilities、Governance、Operations 七层边界；
- 明确 Control Plane / Data Plane、Process / Request / Run / Step 生命周期；
- 明确业务数据、Graph State、Checkpoint、Event Log、Memory 的所有权。

### 用户指定模块深化
- Harness：Factory、Container、DI、装配顺序、生命周期、资源隔离；
- Agent Loop：并行 Tool、Retry、Timeout、Cancellation、Budget、No-progress；
- LangGraph：Python StateGraph、State Schema、Reducer、Checkpoint、Interrupt、Subgraph、恢复与迁移；
- Context：Compiler Pipeline、Trust、Priority、Token Budget、压缩和来源追踪；
- Memory：提取、评分、冲突、合并、版本、过期、遗忘、删除和检索；
- Business Catalog：Python AST / OpenAPI / PostgreSQL 扫描、Entity / Capability、Snapshot、Drift；
- Tool Registry：Definition / Version / Binding / Handler / Execution、Search / Describe / Invoke；
- Multi-Agent：Supervisor、Delegation Contract、权限下放、并行预算、取消和结果聚合；
- Skills：SKILL.md、Front Matter、渐进式加载、包校验和 Sandbox；
- Plugin：Manifest、Host SDK、Entry Points、Scope、生命周期、卸载和隔离；
- Security：Principal、Tenant、RBAC / ABAC、PDP / PEP、Approval、Prompt Injection、Audit；
- Evaluation：Dataset、Trajectory、Evaluator、LLM Judge、Regression、Release Gate。

### 工程落地补充
- PostgreSQL 核心 DDL；
- Python 仓库结构和 Protocol；
- FastAPI / Worker / Queue / Lease / SSE；
- 端到端业务 Agent 案例；
- Python 参考实现骨架；
- 生产部署、SLO 和容量规划；
- 六个参考项目的架构借鉴和采用矩阵。

### 文档约定
- 目录与文件名英文；
- 正文中文；
- 子系统文档开篇说明“本文档在系统中的作用”；
- 区分架构设计、代码示例、伪代码和需要按锁定依赖版本验证的 API。
