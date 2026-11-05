# 实施就绪度与首个 90 天工程路线

## 本文档在系统中的作用

将 V4.0 架构蓝图转化为可执行项目计划。目标不是 90 天内做完所有模块，而是先交付一条安全、可恢复、可评测的 Agent 生产链路，再逐步补齐知识、扩展和治理能力。

## 1. MVP 必须具备

- Agent Definition / Release；
- FastAPI Run API；
- PostgreSQL Run / Job / Event；
- Worker Lease；
- Python Harness Factory；
- 一个 Model Adapter；
- 一个 LangGraph；
- Tool Registry + Executor；
- Tenant / Principal / Policy；
- Idempotency；
- Approval / Interrupt / Resume；
- SSE 回放；
- Trace / Usage；
- 基础 Golden Dataset。

## 2. 可以后置

- 自动 Memory Consolidation；
- 复杂多路 Tool Search；
- 多 Agent Fan-out；
- Skill Marketplace；
- 第三方 Plugin Marketplace；
- 大规模 Catalog 自动语义编译；
- 多向量库路由；
- 高级 LLM Judge 集成。

## 3. 0–30 天：纵向切片

交付一条完整链路：用户请求 → Run 创建 → Worker → Harness → LangGraph → 模型 → 一个只读 Tool → Event → 最终回答。

验收：
- Run 创建幂等；
- Worker 崩溃可重领；
- Event 可断线回放；
- Tool 权限拒绝有效；
- Trace 能定位模型和工具耗时；
- Fake Provider 测试通过。

## 4. 31–60 天：可控写操作

增加：
- Tool Version / Binding；
- 写操作幂等；
- UNKNOWN 状态和对账；
- Approval Service；
- Interrupt / Resume；
- Resource Version / Optimistic Lock；
- Audit / Usage Ledger。

验收：
- 未授权写操作无法到达 Handler；
- 审批绑定 Intent Hash；
- 审批后资源版本变化会阻止执行；
- 外部响应丢失不会重复产生副作用。

## 5. 61–90 天：平台化

增加：
- Context Compiler；
- Catalog Snapshot；
- 基础 Memory；
- Tool Search / Describe；
- Evaluation Runner；
- Release Gate；
- Sandbox / MCP 的受控接入。

验收：
- Agent Release 可复现；
- Catalog Drift 可检测；
- Memory 有 Scope 和证据；
- 核心 Golden / Security Set 自动回归；
- 发布有可审计的 Gate 结果。

## 6. 技术债优先级

P0：跨租户泄漏、未授权执行、重复副作用、恢复后状态不一致。
P1：缺少幂等、缺少可观测性、无法回放、没有版本固定。
P2：Context 成本过高、Tool Search 精度低、Memory 噪声高。
P3：高级插件市场、复杂自动化和 UI 体验优化。

## 7. 团队分工建议

- Runtime：Harness、Loop、LangGraph、Worker；
- Capability：Catalog、Tool Registry、Executor；
- Knowledge：Context、Memory、RAG；
- Governance：Identity、Policy、Approval、Audit；
- Quality：Evaluation、Tracing、Regression；
- Platform：API、DB、Queue、Deployment、SLO。

小团队可以由同一工程师兼任多个方向，但接口和验收责任仍要明确。
