# 18. Security 与 Tenant Governance

## Role in the Agent System

Enforces identity, tenant boundaries, resource permissions, policy decisions, and least privilege.

**Primary responsibility:** Own this capability as a reusable subsystem with explicit contracts, state ownership, security boundaries, and observable outcomes.

**Boundary:** This subsystem must not silently take over responsibilities owned by adjacent layers. Its public interface should be testable without running a full Agent conversation.

## 1. 安全边界

必须分别保护：用户输入、检索内容、Memory、Business Catalog、Tool Schema、Tool Execution、Plugin、Skill、MCP Server、Workspace / Browser / Terminal、Provider Credential、Session / Checkpoint。

## 2. Identity Context

```typescript
interface IdentityContext {
  userId: string;
  tenantId: string;
  roles: string[];
  permissions: string[];
  authnMethod: string;
  requestId: string;
  traceId: string;
  locale?: string;
}
```

userId / tenantId / roles 不得从模型生成的 Tool Arguments 中读取。由 Gateway 从认证上下文注入，并在 Java 服务中再次校验。

## 3. RBAC + ABAC

RBAC：角色和权限编码。ABAC：资源属性、组织范围、数据归属、敏感级别、时间和风险。

最终决策至少考虑 `principal + action + resource + environment + tenant + policy`。

## 4. 多租户隔离

- 每个查询强制 tenant filter。
- 写操作验证资源 tenant。
- Memory / Session / Catalog / Tool Execution 都要有 tenant scope。
- Cache Key 必须包含 tenant / scope。
- MCP Credential 按 tenant / user 管理。
- 必要时 PostgreSQL RLS 作为纵深防御。
- 跨租户管理员权限必须显式授权和审计。

## 5. Prompt Injection

检索文档、网页、Skill、MCP Tool Result 都是不可信数据。它们不能修改 System Policy 或授予权限。

防护：
- 把指令和引用数据分隔。
- Tool Allowlist。
- 参数 Schema 校验。
- 高风险操作审批。
- 资源 ACL。
- 外部网页/文档内容标记为 untrusted。
- 禁止把检索到的指令直接作为系统指令执行。

## 6. Tool Security

最小权限、每次执行重新授权、限制参数长度/分页/时间范围、速率限制、超时和并发限制、幂等、输出脱敏、审计、高风险命令沙箱化。

## 7. Plugin / Skill Security

插件代码视为可执行第三方代码；Skill Markdown 视为不可信指令输入；外部脚本必须隔离运行；Manifest 声明权限；Secret 通过 Secret Manager；禁止插件访问未授权文件系统路径；插件升级要做安全扫描和回滚准备。

## 8. MCP Security

MCP Server 连接需做认证、授权、来源可信度和工具级权限控制。不要因为 Server 能返回一个 Tool 就默认允许当前用户调用。MCP 的工具列表是能力发现，不是业务授权。

## 9. 数据分类

建议 PUBLIC、INTERNAL、CONFIDENTIAL、RESTRICTED。Context Builder 根据当前用户、任务和数据级别过滤。敏感字段默认不进入 LLM 上下文，必要时脱敏或只传派生结果。

## 10. 安全测试

Cross-tenant access、Tool spoofing、Prompt injection、Unauthorized describe/call、Replay attack、Approval parameter tampering、Secret leakage、Plugin escape、Workspace path traversal、Duplicate write。
