# Python Agent 平台实施路线与发布门禁

## 本文档在系统中的作用

把完整架构拆成可执行工程阶段，避免第一版就实现所有高级能力。每阶段都有交付物、依赖和验收门槛。

## Phase 0：基础工程

交付 Python 项目结构、FastAPI lifespan、Settings / Logging / Trace、PostgreSQL / SQLAlchemy Async、Alembic、CI / pytest。验收：启动、关闭、健康检查、数据库迁移和异步测试稳定。

## Phase 1：Harness / Runtime

交付 Agent Definition / Release、Runtime Factory、Runtime Scope、Provider Adapter、Agent Loop、Run / Job、Event Stream。验收：Fake Model 下完成多轮 Tool Calling；Worker 重启可恢复；Run 状态一致。

## Phase 2：LangGraph Durable Execution

交付 StateGraph、Checkpointer、Interrupt / Resume、State Version、Subgraph、Streaming。验收：中断恢复、进程重启、旧 State 迁移和重复 Resume 测试通过。

## Phase 3：Tool Platform

交付 Tool Registry、Pydantic Schema、Dynamic Search / Describe、Tool Executor、Policy、Idempotency、Audit。验收：未授权 Tool 永远不能执行；超时写操作进入 UNKNOWN 并可对账。

## Phase 4：Context / Memory / Catalog

交付 Context Compiler、Token Budget、Memory Lifecycle、RAG、Business Catalog、Catalog Compiler。验收：业务状态实时读取；Memory 不覆盖业务事实；Catalog Drift 可发现。

## Phase 5：Multi-Agent / Skills / Plugins / MCP

交付有界委派、子 Agent 权限、Skill Loader、Plugin SDK、MCP Client、Sandbox。验收：子 Agent 不扩大权限；不可信插件不在主进程运行；MCP SSRF 防护通过。

## Phase 6：Evaluation / Production Hardening

交付 Golden Dataset、Evaluator、Trajectory、Cost Ledger、SLO、Load / Chaos Test、Release Gate。验收：版本发布前质量、安全、成本和恢复门禁自动执行。

## 不建议第一版实现

无约束 Auto-Growing Memory、自主修改 Tool / Policy、任意 Python Code Execution、无限制 Multi-Agent、Plugin Marketplace、自动生成并自动发布 Skill、模型自行修改业务数据库 Schema。这些能力必须在核心运行时、安全和评测稳定后再考虑。
