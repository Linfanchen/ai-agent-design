# 子系统地图与依赖规则

## 本文档在系统中的作用

定义 Python 仓库内的依赖方向，帮助团队保持模块边界。目录结构决定哪些模块可以依赖哪些模块，避免循环依赖。

## 1. 依赖方向

```text
interfaces (FastAPI / CLI / MCP)
          ↓
application (Use Cases / Commands / Queries)
          ↓
runtime (Harness / Graph / Run Execution)
          ↓
domain (Agent / Tool / Memory / Policy Contracts)
          ↑
adapters (LLM / DB / Redis / Vector / Sandbox / MCP)
```

`domain` 不依赖 FastAPI、SQLAlchemy、LangGraph 或具体 Provider SDK。`application` 依赖 domain ports。`adapters` 实现 ports。`runtime` 组合 domain 与 adapters。`interfaces` 只调用 application。

## 2. 推荐包边界

- `agent_platform.domain`：实体、值对象、枚举、领域错误、Port。
- `agent_platform.application`：用例、事务边界、命令查询、权限检查编排。
- `agent_platform.runtime`：Harness、Run Scope、Graph Factory、Loop。
- `agent_platform.capabilities`：Tool、Skill、Plugin、MCP。
- `agent_platform.knowledge`：Context、Memory、RAG、Catalog。
- `agent_platform.governance`：Identity、Policy、Approval、Audit。
- `agent_platform.adapters`：模型、数据库、缓存、向量、对象存储。
- `agent_platform.interfaces`：FastAPI、WebSocket、MCP Server。
- `agent_platform.workers`：Job Claim、执行 Worker、Scheduler。
- `agent_platform.observability`：Trace、Metrics、Trajectory、Evaluation。

## 3. 不允许的依赖

- domain → FastAPI；
- domain → SQLAlchemy ORM Model；
- Graph Node → FastAPI Request；
- Tool Definition → 具体模型 SDK；
- Memory → 业务 Repository 直接写入；
- Plugin → 修改宿主私有 Registry；
- API Router → 直接 new Graph 并执行长任务；
- Evaluator → 修改生产 Run 状态。

## 4. 共享类型策略

区分 Domain Value Objects、API DTO、Graph State Schema、Tool Input / Output Schema、Event Schema、Persistence ORM Models。跨边界通过 Mapper 转换，避免把 SQLAlchemy Model 直接序列化给模型或前端。

## 5. 循环依赖控制

使用 Protocol / ABC 定义 Port；在 Composition Root 注入实现。静态检查可用 import-linter 等工具，CI 中禁止 domain 依赖 adapters。不要通过 `sys.path` 动态注入掩盖包边界错误。
