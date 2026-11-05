# Policy Engine：PDP / PEP 的 Python 实现边界

## 本文档在系统中的作用

本文件将 RBAC / ABAC 规则转化为可执行授权架构。它说明 Policy Decision Point 如何做决策、Policy Enforcement Point 如何拦截动作，以及如何避免把授权逻辑分散在 Prompt 和各个 Tool Handler 中。

## 1. PDP 与 PEP

PDP 是策略决策点，接收 Principal、Action、Resource、Environment，返回 Allow / Deny / Require Approval / Conditional Allow。
PEP 是策略执行点，位于 API、Tool Executor、Artifact Gateway、Memory Retrieval 和 Repository 边界。任何可能泄露数据或产生副作用的路径都必须经过 PEP。

## 2. Policy Input

```python
from pydantic import BaseModel, Field
from typing import Literal

class PolicyInput(BaseModel):
    tenant_id: str
    principal_id: str
    principal_type: str
    action: str
    resource_type: str
    resource_id: str
    resource_tenant_id: str
    data_classification: str
    environment: dict
    agent_release_id: str | None = None
    tool_version_id: str | None = None
    approval_id: str | None = None

class PolicyDecision(BaseModel):
    effect: Literal["allow", "deny", "require_approval", "conditional_allow"]
    policy_id: str | None = None
    policy_version: str | None = None
    reason_code: str
    obligations: dict = Field(default_factory=dict)
    expires_at: str | None = None
```

生产实现应使用 `Field(default_factory=dict)`，并为 Environment 定义白名单 Schema。

## 3. 规则执行顺序

1. Principal 有效；
2. Tenant 一致；
3. Resource 存在且可见；
4. Agent Release / Tool Binding 有效；
5. Action 在角色权限内；
6. ABAC 条件满足；
7. Data Classification 允许；
8. Environment / Time / Network 限制满足；
9. 风险级别与审批策略满足；
10. Quota / Rate Limit 满足。

## 4. Conditional Allow

例如允许读取 Learner，但仅允许字段 `learner_id`、`display_name`、`status`。Policy Decision 返回 Projection Obligation，PEP 在 Repository / Serializer 层实施字段过滤。不要让模型自行决定哪些敏感字段可见。

## 5. Policy Cache

Cache Key 包含 Tenant、Principal、Action、Resource、Policy Version、Resource Version 和必要环境属性。缓存 TTL 应与风险匹配。写操作重新检查实时撤销状态。

## 6. 策略变更

Policy Version 不可变；变更发布后，新请求使用新版本。高危撤销通过 Revocation Overlay 即时生效。审计保存命中 Policy ID / Version 和 Decision Reason Code。

## 7. Fail-closed

Policy 服务不可用时，高风险写操作拒绝；Tenant 不确定时拒绝；Resource Scope 无法解析时拒绝；权限缓存过期且无法刷新时拒绝敏感动作。

## 8. 测试

角色允许 / 拒绝、跨租户、字段脱敏、审批义务、时间条件、策略撤销、缓存过期、Delegated Grant 到期、Policy 服务超时。

## 9. Policy Decision 的执行伪代码

```python
async def authorize_tool_call(request, principal, scope):
    tool = await registry.get(request.tool_key, request.version)
    await binding_service.require_enabled(scope.agent_release_id, tool.id)

    resource = await resource_resolver.resolve(
        tool.resource_type, request.arguments, principal.tenant_id
    )
    if resource.tenant_id != principal.tenant_id:
        return PolicyDecision(effect="deny", reason_code="TENANT_MISMATCH")

    decision = await policy_engine.evaluate(
        principal=principal,
        action=tool.action,
        resource=resource,
        environment=scope.environment,
    )
    return decision
```

真实代码应将 `resource_resolver` 做成只读且受控的解析器；不能让模型直接指定资源 Tenant。Policy 决策返回后，PEP 才能执行或拒绝。

## 10. Policy Rule 的版本化

每个 Policy Rule 保存 Rule ID、Version、Scope、Effect、Conditions、Obligations、Priority、Effective Time、Author、Reviewer 和 Test Cases。修改规则时生成新版本，不原地覆盖历史。

## 11. 字段级权限

Read Capability 可声明允许字段集合。Repository Query Projection 和 Serializer 共同实施字段过滤。若模型请求未授权字段，返回字段被隐藏的安全结果，而不是把完整对象传给模型后要求它不要展示。

## 12. 策略测试自动生成

从每条规则生成：
- 满足条件的 Allow Case；
- 缺少关键属性的 Deny Case；
- Tenant 不匹配 Case；
- Role 不足 Case；
- Resource State 不满足 Case；
- Approval 缺失 / 过期 Case；
- Policy Version 撤销 Case。

Policy 变更必须运行全部相关 Case。
