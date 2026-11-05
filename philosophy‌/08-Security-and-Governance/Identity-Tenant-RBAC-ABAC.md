# Identity、Tenant、RBAC 与 ABAC

## 本文档在系统中的作用

Identity 与 Authorization 确定“谁在代表谁、属于哪个租户、能对哪个资源做什么”。模型输出不是身份，Prompt 不是权限，Tool Description 也不是授权规则。

## 1. Principal

可信 Principal 至少包含 principal_id、principal_type（user / service / agent / plugin）、tenant_id、roles、groups、authentication_method、session_id、delegated_by、delegation_id、token_expiry。Principal 由认证和身份映射层构造，不从模型生成参数中读取。

## 2. Tenant Resolution

Tenant 可由登录上下文、域名、组织映射或受信任 API Key 确定。客户端传入 tenant_id 只能作为请求意图，服务端必须验证用户是否属于该 Tenant。

## 3. RBAC 与 ABAC

RBAC 适合稳定角色：admin、manager、mentor、learner、auditor。

ABAC 用于细粒度条件：
- principal.tenant_id == resource.tenant_id；
- principal.department in resource.allowed_departments；
- action == "learner.status.transition"；
- resource.status == "active"；
- context.approval_id is valid；
- context.data_classification <= principal.clearance。

## 4. PDP / PEP

PDP 负责计算 Allow / Deny / RequireApproval；PEP 在 Tool Executor、API、Repository 或 Resource Gateway 处执行结果。只在 UI 隐藏按钮不是权限控制。

## 5. 默认拒绝

没有匹配规则、Policy 服务异常、Resource Scope 无法解析、Tenant 不确定时，高风险动作默认 Deny。只读降级也需要明确策略。

## 6. 行级数据隔离

Repository 接收 TenantScope，所有查询强制 Tenant Filter。PostgreSQL Row Level Security 可作为纵深防御，但连接池中的 session variable 必须在每个事务安全设置和重置，避免连接复用导致租户上下文泄漏。

## 7. Agent Delegation

Agent 只能使用被 Release 绑定并且 Principal 当前有权使用的 Capability。子 Agent 获得受限 Delegated Grant，不能继承父 Agent 的所有权限。

## 8. 权限变化

权限撤销后，下一次 Tool Invoke 必须重新授权。长任务不能仅凭 Run 创建时权限继续执行数小时。敏感操作在副作用前检查 Policy Version / Revocation State。

## 9. 测试

跨租户读写、角色边界、资源所有权变更、权限中途撤销、子 Agent 越权、缓存污染、RLS 连接复用等。

## 10. 授权决策必须发生在数据边界

只在 Agent 开始时授权一次是不够的。每次敏感读取和每次写入都应经过 PEP。长任务期间，用户可能被移出项目、角色可能变化、资源可能转移 Tenant，因此在 Tool Invoke 时重新检查。

## 11. Principal 与 Delegation

用户触发 Agent 时，Agent 不是用户本人，也不是独立管理员。Agent 代表用户在有限范围内行动。Principal 可包含 `actor_user_id`、`execution_agent_id` 和 `delegation_id`，Audit 应同时记录发起用户与实际执行 Agent。

## 12. Repository Tenant Scope

Repository 方法显式接收 Scope：
```python
async def get_learner(
    session,
    tenant_scope,
    learner_id: str,
):
    stmt = select(Learner).where(
        Learner.id == learner_id,
        Learner.tenant_id == tenant_scope.tenant_id,
    )
    return await session.scalar(stmt)
```

示例中的 Tenant 条件必须由 Repository 强制追加，不能由调用者传入一个可覆盖的过滤表达式。

## 13. RLS 纵深防御

PostgreSQL RLS 可作为第二层隔离。使用连接池时，必须保证每个事务都设置并重置当前 Tenant 上下文，不能让上一个租户的 session variable 留在被复用连接中。RLS Policy 和 Repository Filter 应有一致性测试。

## 14. 资源枚举防护

当用户无权访问某资源时，按产品安全策略统一返回 404 或 403。不能通过错误消息差异泄露其他租户是否存在某个 Learner / Project。
