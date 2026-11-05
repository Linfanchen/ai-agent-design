# Tool Registry / Executor：完整 Python 契约与实现骨架

## 本文档在系统中的作用

本文件将 Tool 从注册元数据延伸到可执行契约，给出 Pydantic 输入输出、Handler、Context、Executor 和错误语义的 Python 骨架。

## 1. Tool Context

Tool Context 只包含执行必需的可信引用：
- tenant_id；
- principal_ref；
- run_id；
- trace_id；
- deadline；
- cancellation scope；
- policy service；
- repository factory；
- secret resolver；
- event publisher。

不将完整密钥、FastAPI Request 或共享 AsyncSession 传给 Handler。

## 2. 输入与输出 Schema

```python
from pydantic import BaseModel, ConfigDict, Field
from typing import Literal

class LearnerReadInput(BaseModel):
    model_config = ConfigDict(extra="forbid")
    learner_id: str = Field(min_length=1, max_length=80)
    include_history: bool = False

class LearnerReadOutput(BaseModel):
    model_config = ConfigDict(extra="forbid")
    learner_id: str
    display_name: str
    status: str
    resource_version: int
    evidence_ref: str
```

Schema 校验只是第一层。Handler 还要检查资源存在、租户归属、字段可见性和业务约束。

## 3. Handler Contract

```python
from typing import Protocol, TypeVar, Generic

I = TypeVar("I", bound=BaseModel)
O = TypeVar("O", bound=BaseModel)

class TypedToolHandler(Protocol, Generic[I, O]):
    input_model: type[I]
    output_model: type[O]

    async def execute(self, args: I, context: "ToolContext") -> O:
        ...
```

## 4. Executor Contract

```python
class ToolExecutor:
    async def execute(self, request, principal, scope):
        definition = await self.registry.get_version(request.tool_key, request.version)
        binding = await self.bindings.require_enabled(scope.agent_release_id, definition.id)
        args = definition.input_model.model_validate(request.arguments)

        await self.authorizer.require_tool_access(principal, binding, args, scope)
        decision = await self.policy.evaluate(
            principal=principal,
            action=definition.action,
            resource=await self.resource_resolver.resolve(args),
            context=scope.policy_context,
        )
        if decision.kind == "DENY":
            return self.results.denied(decision.safe_reason)

        if decision.kind == "REQUIRE_APPROVAL":
            return await self.approvals.request_and_suspend(request, decision, scope)

        execution = await self.idempotency.begin(request, args)
        if execution.is_existing_terminal:
            return execution.cached_result

        try:
            output = await self.handler_runner.run(definition, args, scope)
            validated = definition.output_model.model_validate(output)
            await self.verifier.verify(definition, args, validated, scope)
            return await self.idempotency.complete(execution, validated)
        except ExternalOutcomeUnknown:
            await self.idempotency.mark_unknown(execution)
            raise
```

此骨架强调顺序；实际实现需要把事务、异常分类、审计和 Outbox 的提交边界设计完整。

## 5. Tool Metadata 不等于 Handler

Tool Registry 保存元数据；Handler Registry 保存已审核的可执行对象。运行时通过内部 `handler_ref` 映射到预注册 Handler，不允许模型提交 Python 路径。

## 6. 结果错误语义

- `INVALID_ARGUMENTS`：Schema 不通过；
- `NOT_FOUND`：资源不存在或按安全策略隐藏；
- `PERMISSION_DENIED`：无权；
- `APPROVAL_REQUIRED`：等待审批；
- `PRECONDITION_FAILED`：状态或版本冲突；
- `RATE_LIMITED`：配额或并发限制；
- `EXECUTION_UNKNOWN`：外部结果未知；
- `DEPENDENCY_UNAVAILABLE`：下游不可用。

## 7. 事务与 Session

Handler 使用 Repository Factory 获取短 AsyncSession。不要在 Executor 外层持有一个 Session 并并行传给多个 Tool。外部 HTTP 调用不放在 DB Transaction 中。

## 8. 测试

Input Schema、Output Schema、Handler Contract、Tenant Filter、Policy、Approval、Idempotency、Timeout、Cancellation、External UNKNOWN、Audit、Output Evidence。
