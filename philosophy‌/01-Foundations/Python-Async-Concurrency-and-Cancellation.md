# Python asyncio 并发、取消与资源管理

## 本文档在系统中的作用

Python 异步模型决定 Agent 如何并发调用模型、工具和存储。很多 Agent 可靠性问题并非模型能力问题，而是 AsyncSession 共享、Task 取消处理不当、阻塞 I/O 占用 Event Loop 和后台任务生命周期失控。

## 1. 并发类型

- I/O-bound：HTTP、数据库、Redis、对象存储，使用 async / await。
- CPU-bound：大型文件解析、重排序、复杂计算，使用 Process Pool 或独立 Worker。
- Blocking SDK：使用线程池隔离，不要直接阻塞 Event Loop。
- GPU / Sandbox：独立服务或资源池，不在 API 进程中任意运行。

## 2. Structured Concurrency

对一组必须共同完成的并行 Tool，优先使用 `asyncio.TaskGroup`，让子任务生命周期受父任务约束。

```python
import asyncio

async def execute_parallel(tool_calls, executor):
    results = {}
    async with asyncio.TaskGroup() as group:
        tasks = {
            call.id: group.create_task(executor.execute(call))
            for call in tool_calls
        }
    for call_id, task in tasks.items():
        results[call_id] = task.result()
    return results
```

上例表示“全组成功”的 TaskGroup 语义。若业务允许部分成功，应捕获每个 Tool 的领域异常并转为 `ToolOutcome`，不要让一个普通业务错误导致所有任务无差别取消。

## 3. 取消传播

`asyncio.CancelledError` 是控制流，不应被宽泛 `except Exception` 吞掉。清理资源后通常需要继续抛出。

```python
async def run_step():
    try:
        return await perform_io()
    except asyncio.CancelledError:
        await release_temporary_resources()
        raise
```

外部请求断开、用户取消 Run、Worker Shutdown 是不同取消来源，应记录 Cancellation Reason。

## 4. Timeout

`asyncio.timeout()` 可以限制协程执行时间，但超时取消不代表外部系统没有执行副作用。Tool Executor 应把本地 Timeout 与外部操作结果分开建模。

## 5. SQLAlchemy AsyncSession 规则

`AsyncSession` 是有状态事务对象，不允许多个并发 Task 共享同一个 Session。每个并行 Tool / Repository Unit of Work 必须独立创建 Session。事务尽量短，不要在持有数据库事务期间等待 LLM 或外部 HTTP。

## 6. FastAPI 生命周期

使用 FastAPI lifespan 管理进程级资源，如 HTTP Client、DB Engine、Redis Pool、Telemetry Provider。不要在每个请求中重复创建连接池，也不要把请求级 Principal 或 AsyncSession 放入进程级单例。

## 7. 背压和并发上限

每个 Provider 设置并发 Semaphore；每个 Tenant 设置 Run 并发配额；Tool Category 设置并发上限；Sandbox 使用独立容量池；队列积压时拒绝、排队或降级；不允许无限创建 asyncio Task。

## 8. 必测场景

取消发生在模型流、Tool HTTP 请求、DB 事务、子 Agent、Sandbox、SSE 断开和 Worker Shutdown 时，分别验证资源释放、状态一致性和副作用对账。
