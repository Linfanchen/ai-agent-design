# Runtime Factory、Container 与依赖注入

## 本文档在系统中的作用

Factory 决定创建什么；Container 持有本次运行可访问的依赖；Dependency Injection 决定依赖如何进入组件；Scope 决定依赖何时释放。

## 1. 依赖生命周期矩阵

| 生命周期 | 示例 | 创建位置 | 销毁位置 |
|---|---|---|---|
| Process | AsyncEngine、HTTPX AsyncClient、Redis Pool | FastAPI lifespan | Process shutdown |
| Tenant Config Cache | 已验证租户策略快照 | Cache Adapter | TTL / invalidation |
| Request | Principal、Request ID | Auth Middleware | Request 完成 |
| Run | Runtime Scope、预算、取消状态 | RuntimeFactory | Run 终止或挂起清理 |
| Step | AsyncSession、HTTP Response | Node / Repository | Step 结束 |
| Tool | Tool Context、临时文件 | ToolExecutor | Tool 完成 |
| Sandbox | 容器 / Job | Sandbox Adapter | 容器完成或强制清理 |

## 2. Container 不应成为 Service Locator 大杂烩

Container 只暴露明确依赖属性或 `get_port()`。不要允许业务代码随意通过字符串获取任意对象。推荐构造函数注入，便于静态检查和单元测试。

## 3. Protocol 定义

```python
from typing import Protocol

class Clock(Protocol):
    def now(self): ...

class ModelPort(Protocol):
    async def complete(self, request): ...

class MemoryPort(Protocol):
    async def recall(self, query, scope): ...

class ToolExecutionPort(Protocol):
    async def invoke(self, request, principal): ...

class PolicyPort(Protocol):
    async def authorize(self, decision_input): ...
```

Port 表达业务需要，而不是简单复制第三方 SDK 的所有方法。

## 4. AsyncExitStack 资源容器

```python
from contextlib import AsyncExitStack

class RunResourceScope:
    def __init__(self):
        self._stack = AsyncExitStack()
        self._entered = False

    async def __aenter__(self):
        await self._stack.__aenter__()
        self._entered = True
        return self

    async def enter_async_context(self, manager):
        if not self._entered:
            raise RuntimeError("scope is not entered")
        return await self._stack.enter_async_context(manager)

    async def __aexit__(self, exc_type, exc, tb):
        return await self._stack.__aexit__(exc_type, exc, tb)
```

Container 不应长期保存 AsyncSession。需要 DB 时，通过 Session Factory 创建短生命周期 Session。

## 5. Factory 输入输出

`RunSpec` 至少包含 tenant_id、agent_release_id、thread_id、run_id、input_ref、idempotency_key、requested_mode。Principal 从认证层注入，不从用户请求 JSON 反序列化。

`AgentRuntime` 至少暴露 `ainvoke()`、`astream()`、`resume()`、`cancel()` 和 `aclose()`。实际 API 根据 LangGraph Runnable 接口封装，不直接把内部 Graph 对象暴露给外部。

## 6. 单例与多租户

Provider SDK 的 HTTP Client 可以按进程复用，但 Provider Credential 和 Model Profile 必须按租户 / Release 选择。缓存对象不能因复用而携带上一个用户的 Authorization Header。

## 7. 测试替身

注入 FakeModelProvider、FakePolicyEngine、FakeToolRegistry、InMemoryEventPublisher、InMemoryMemoryStore、FakeClock、FakeCheckpointer。测试验证 Factory 不会在缺少 Principal、Tenant 或 Published Release 时创建 Runtime。
