# Python 参考实现骨架

## 本文档在系统中的作用

给出核心模块如何组合成最小 Python 项目的骨架。它不是完整可运行产品，而是帮助开发团队建立文件结构、依赖注入和边界的起点。

## 1. 推荐 pyproject 依赖分组

```toml
[project]
name = "agent-platform"
requires-python = ">=3.12"
dependencies = [
  "fastapi",
  "uvicorn[standard]",
  "pydantic>=2",
  "pydantic-settings",
  "sqlalchemy>=2",
  "asyncpg",
  "alembic",
  "langgraph",
  "httpx",
  "tenacity",
  "redis",
  "structlog",
  "opentelemetry-api",
  "opentelemetry-sdk",
]

[project.optional-dependencies]
mcp = ["mcp[cli]"]
vector = ["pgvector"]
test = ["pytest", "pytest-asyncio", "httpx", "respx"]
quality = ["ruff", "mypy"]
```

正式项目应通过 `uv lock` 或等价工具生成精确 lockfile；上面仅列出直接依赖类别，不代表完整兼容版本组合。

## 2. Settings

```python
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        env_prefix="AGENT_",
        env_file=".env",
        extra="ignore",
    )
    database_url: str
    redis_url: str | None = None
    environment: str = "local"
    default_run_timeout_seconds: int = 300
```

密钥不写入 Agent Definition 或 Release Manifest。生产使用 Secret Manager / Secret Reference。

## 3. FastAPI Lifespan

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker
import httpx

@asynccontextmanager
async def lifespan(app: FastAPI):
    engine = create_async_engine(settings.database_url, pool_pre_ping=True)
    session_factory = async_sessionmaker(engine, expire_on_commit=False)
    http_client = httpx.AsyncClient(timeout=httpx.Timeout(30.0))
    app.state.session_factory = session_factory
    app.state.http_client = http_client
    try:
        yield
    finally:
        await http_client.aclose()
        await engine.dispose()

app = FastAPI(lifespan=lifespan)
```

此处仅展示 Process 资源。不要将一个 AsyncSession 放入 app.state 并让所有请求共享。

## 4. Run Application Service

```python
class CreateRunService:
    def __init__(self, run_repo, job_repo, outbox_repo, quota_service):
        self.run_repo = run_repo
        self.job_repo = job_repo
        self.outbox_repo = outbox_repo
        self.quota_service = quota_service

    async def create(self, command, principal):
        await self.quota_service.assert_can_start(principal.tenant_id)
        async with self.run_repo.unit_of_work() as uow:
            existing = await self.run_repo.find_by_idempotency(
                principal.tenant_id, command.idempotency_key
            )
            if existing:
                existing.assert_same_request(command.request_hash)
                return existing
            run = await self.run_repo.create(command, principal)
            await self.job_repo.enqueue(run.id, queue="interactive")
            await self.outbox_repo.append("run.created", {"run_id": str(run.id)})
            await uow.commit()
        return run
```

实际 Repository / Unit of Work 应保证 Outbox 与 Run / Job 在同一数据库事务内提交。

## 5. Worker 执行骨架

```python
import asyncio

async def process_job(job, runtime_factory):
    runtime = None
    try:
        runtime = await runtime_factory.create_for_job(job)
        await runtime.execute()
    except asyncio.CancelledError:
        await mark_cancel_requested(job.run_id)
        raise
    except Exception as exc:
        await classify_and_persist_failure(job, exc)
    finally:
        if runtime is not None:
            await runtime.aclose()
```

Worker 的 Job Lease、Heartbeat、Checkpoint 和 Tool UNKNOWN 对账需要独立组件，不应隐藏在此函数里。

## 6. 测试目录

- `tests/unit`：领域规则、评分算法、Schema；
- `tests/contract`：Provider / Tool / Plugin / MCP Adapter；
- `tests/integration`：PostgreSQL、Redis、Checkpointer；
- `tests/evaluation`：Golden Dataset；
- `tests/security`：Tenant Isolation、Prompt Injection、SSRF、Sandbox；
- `tests/chaos`：Worker 崩溃、外部超时、Checkpoint 失败。

## 7. 实施提醒

先跑 Fake Provider + Fake Tools，确认 Runtime / Graph / Tool Executor / Event 的接口边界，再接真实模型。不要在第一阶段引入所有插件、向量库和多 Agent。
