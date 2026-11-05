# Middleware、Hooks 与执行管线

## 本文档在系统中的作用

Middleware 为 Agent Run 和 Tool Execution 提供横切能力，例如鉴权、预算、限流、日志、Trace、输入过滤、输出校验和审计，避免每个 Graph Node 重复实现。

## 1. 两条管线

Run Pipeline：`Authenticate → Resolve Tenant → Load Release → Quota → Context → Graph → Output Validation → Persist`

Tool Pipeline：`Resolve Tool → Validate Schema → Reauthorize → Policy → Approval → Idempotency → Execute → Verify → Audit → Return`

Run Middleware 与 Tool Middleware 的输入和风险不同，不要混成一个全局列表。

## 2. Middleware Contract

```python
from typing import Protocol, Awaitable, Callable, Any
Next = Callable[[Any], Awaitable[Any]]

class Middleware(Protocol):
    async def __call__(self, context: Any, next_handler: Next) -> Any: ...
```

真实实现应为不同管线定义类型化 Context，避免 Any 泄漏到业务代码。

## 3. 执行顺序

鉴权必须在数据读取之前；策略必须在副作用之前；审计至少记录执行意图和最终结果。限流与预算在高成本操作前完成。

## 4. Hook 与 Middleware 区别

Middleware 包裹执行并可阻止、修改或返回结果；Hook 是事件通知，例如 `on_run_started`、`on_tool_completed`。Hook 默认不应阻断主流程。若 Hook 失败影响安全或账务，必须明确设置为阻断型 Hook 并使用持久化 Outbox。

## 5. Fail-open / Fail-closed

- Policy Engine 不可用：高风险写操作 Fail-closed。
- Audit Outbox 不可写：高风险写操作 Fail-closed；普通只读可按策略降级。
- Metrics 不可用：通常 Fail-open。
- Optional Memory Recall 不可用：可降级，但需记录。
- Tenant Resolver 不可用：Fail-closed。
- Output Formatter 不可用：返回安全错误，不得伪造成功。

## 6. 防止重复副作用

Middleware 可能因重试、Graph 恢复或请求重复而再次进入。副作用只能放在具有幂等键的 Executor / Use Case 中，不能在通用 `before_run` Hook 中直接扣费或更新业务状态。

## 7. 测试

测试执行顺序、短路行为、异常传播、取消传播、Hook 重复投递、Outbox 重试和安全失败策略。
