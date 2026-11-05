# Agent Loop：Python 参考实现与状态机

## 本文档在系统中的作用

给出 Loop 核心控制流程的可实现形状。生产系统建议由 LangGraph 管理长流程状态，而 Loop Engine 作为一个受控执行组件，负责一次模型决策与工具调用轮次。

## 1. Loop 状态

`READY → MODEL_REQUESTED → MODEL_RESPONDED → TOOL_DISPATCHING → OBSERVING → READY`

终止状态：
- `COMPLETED`
- `NEED_CLARIFICATION`
- `WAITING_APPROVAL`
- `REFUSED`
- `FAILED`
- `CANCELLED`
- `BUDGET_EXCEEDED`
- `NO_PROGRESS`
- `DEADLINE_EXCEEDED`

所有状态变化都产生内部 Event，并由 Run Orchestrator 更新权威 Run 状态。

## 2. 执行请求

```python
from dataclasses import dataclass
from datetime import datetime

@dataclass(frozen=True)
class LoopLimits:
    max_model_calls: int
    max_tool_calls: int
    max_parallel_tools: int
    max_steps: int
    max_runtime_seconds: int
    max_cost_usd: float
    max_repeat_fingerprint: int

@dataclass(frozen=True)
class LoopRequest:
    run_id: str
    thread_id: str
    user_input: str
    deadline_at: datetime
    limits: LoopLimits
```

## 3. 核心伪实现

```python
import asyncio

async def run_loop(request, ports):
    model_calls = 0
    tool_calls = 0
    step = 0
    fingerprints = {}

    while step < request.limits.max_steps:
        ports.cancellation.raise_if_cancelled()
        ports.deadline.raise_if_expired()

        if model_calls >= request.limits.max_model_calls:
            return await ports.terminal.finish("BUDGET_EXCEEDED")

        context = await ports.context.compile(request)
        response = await ports.model.complete(
            ports.request_builder.build(context)
        )
        model_calls += 1
        await ports.events.model_responded(response)

        decision = ports.response_parser.parse(response)

        if decision.kind == "FINAL":
            return await ports.output_validator.validate_and_finish(decision)

        if decision.kind == "CLARIFICATION":
            return await ports.terminal.wait_for_input(decision)

        if decision.kind == "REFUSAL":
            return await ports.terminal.finish("REFUSED")

        if decision.kind != "TOOL_REQUEST":
            return await ports.terminal.finish("FAILED")

        calls = decision.tool_calls
        if tool_calls + len(calls) > request.limits.max_tool_calls:
            return await ports.terminal.finish("BUDGET_EXCEEDED")

        fingerprint = ports.fingerprinter.for_calls(calls)
        fingerprints[fingerprint] = fingerprints.get(fingerprint, 0) + 1
        if fingerprints[fingerprint] >= request.limits.max_repeat_fingerprint:
            return await ports.terminal.finish("NO_PROGRESS")

        outcomes = await ports.tool_dispatcher.execute(
            calls,
            max_concurrency=request.limits.max_parallel_tools,
        )
        tool_calls += len(calls)
        await ports.observations.append(outcomes)
        step += 1

    return await ports.terminal.finish("BUDGET_EXCEEDED")
```

此示例省略了细节：模型流式调用、异常分类、预算预留、Tool UNKNOWN 对账、LangGraph Checkpoint 和 Event Transaction。它展示控制顺序，不应直接作为生产代码复制。

## 4. Retry 的边界

模型读取类请求可以按错误分类重试；Tool 执行必须由 Tool Executor 决定。Loop 不得捕获任意 Tool Exception 后直接重试 Handler。外部写操作超时应交由执行记录对账。

## 5. 并行 Tool

并行调度器根据 Tool Contract 中的 `concurrency_class` 和 `resource_key` 决定是否并行。默认：
- 独立只读：可并行；
- 同资源写：串行；
- 不可证明独立的写：串行；
- 有事务依赖：交由业务 Workflow；
- 需要前置结果：按依赖 DAG 排序。

## 6. Loop Guard

重复检测不能只看 Tool 名称。建议 Fingerprint = Hash(Tool Version + Canonical Arguments + Target Resource + Current Step Intent)。对同参数但不同业务上下文的合法调用，需要包含 Intent / Resource Version。

## 7. 退出前置条件

成功结束前检查：
- 必需业务步骤完成；
- 写操作有权威结果；
- 必要结果已持久化；
- 用户可见输出通过 Schema / Policy；
- Run 状态与 Graph 状态一致；
- 未遗留运行中的 Tool Task；
- 预算账本已提交。

## 8. 最低测试

无工具结束、单工具、多个并行只读工具、参数错误、权限拒绝、超时 UNKNOWN、重复调用、用户取消、预算耗尽、模型流中断、最终输出 Schema 错误。
