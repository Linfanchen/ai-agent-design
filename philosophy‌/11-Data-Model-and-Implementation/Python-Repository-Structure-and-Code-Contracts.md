# Python 仓库结构与核心代码契约

## 本文档在系统中的作用

将架构映射为 Python 包目录，定义代码归属、稳定接口和禁止跨边界泄漏的对象。目标是多个 Agent 项目复用底层框架，业务 Agent 只实现自己的 Domain / Graph / Tool。

## 1. 推荐目录

```text
agent-platform/
├── pyproject.toml
├── uv.lock
├── alembic.ini
├── migrations/
├── src/agent_platform/
│   ├── domain/
│   │   ├── agents/
│   │   ├── runs/
│   │   ├── tools/
│   │   ├── memory/
│   │   ├── catalog/
│   │   ├── approval/
│   │   └── policies/
│   ├── application/
│   │   ├── commands/
│   │   ├── queries/
│   │   └── services/
│   ├── runtime/
│   │   ├── harness/
│   │   ├── factory/
│   │   ├── loop/
│   │   ├── graph/
│   │   ├── context/
│   │   └── scope/
│   ├── capabilities/
│   │   ├── registry/
│   │   ├── discovery/
│   │   ├── executor/
│   │   ├── skills/
│   │   ├── plugins/
│   │   └── mcp/
│   ├── knowledge/
│   │   ├── memory/
│   │   ├── rag/
│   │   └── business_catalog/
│   ├── governance/
│   │   ├── identity/
│   │   ├── policy/
│   │   ├── approval/
│   │   └── audit/
│   ├── adapters/
│   │   ├── llm/
│   │   ├── persistence/
│   │   ├── cache/
│   │   ├── vector/
│   │   ├── storage/
│   │   └── sandbox/
│   ├── interfaces/
│   │   ├── http/
│   │   ├── websocket/
│   │   ├── mcp_server/
│   │   └── cli/
│   ├── workers/
│   └── observability/
├── tests/
│   ├── unit/
│   ├── contract/
│   ├── integration/
│   ├── evaluation/
│   └── security/
└── deploy/
```

## 2. 核心 Protocol

```python
from typing import Protocol, AsyncIterator

class ModelPort(Protocol):
    async def complete(self, request): ...
    def stream(self, request) -> AsyncIterator: ...

class ContextPort(Protocol):
    async def compile(self, request, scope): ...

class ToolSearchPort(Protocol):
    async def search(self, query, principal, scope): ...
    async def describe(self, tool_ref, principal, scope): ...

class ToolExecutorPort(Protocol):
    async def execute(self, request, principal, scope): ...

class EventPort(Protocol):
    async def append(self, run_id, event): ...
    async def replay(self, run_id, after_sequence): ...
```

LangGraph Checkpointer 优先使用官方实现；`CheckpointPort` 是平台概念接口，不要求重造 LangGraph 内部存储引擎。

## 3. Domain / DTO / ORM 分离

同一业务概念可能有 Domain Entity、Pydantic API DTO、Graph State TypedDict、SQLAlchemy ORM、Tool Input Model。它们不能直接共用一个类。跨层通过 Mapper 转换，并在边界校验。

## 4. 错误类型

统一 Domain Error：NotFound、PermissionDenied、ValidationFailed、Conflict、RateLimited、DependencyUnavailable、ExecutionUnknown、BudgetExceeded、Cancelled。

错误对象包含 code、safe_message、retryable、correlation_id、details_ref。不要将原始 SDK Exception 返回模型或客户端。

## 5. Python 代码规范

关键公共接口有 Type Hint；I/O 使用 async；CPU 密集工作隔离；Pydantic V2 用于边界 Schema；SQLAlchemy AsyncSession 短生命周期；不在模块 Import 阶段连接数据库；不使用全局可变 Tenant Context；`contextvars` 仅传递 Trace / Correlation，不替代显式 Principal；使用 Ruff / mypy 或 pyright；依赖通过 lockfile 固定。

## 6. CI 门禁

CI 执行格式化、Lint、Type Check、Unit Test、Contract Test、Migration Check、Security Scan、Golden Evaluation。核心模块要求覆盖分支而不仅是行覆盖。
