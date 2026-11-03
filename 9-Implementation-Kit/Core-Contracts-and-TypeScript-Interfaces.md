# 24. API Contracts 与参考实现

## Role in the Agent System

Defines stable TypeScript contracts between runtime engines and external business services.

**Primary responsibility:** Own this capability as a reusable subsystem with explicit contracts, state ownership, security boundaries, and observable outcomes.

**Boundary:** This subsystem must not silently take over responsibilities owned by adjacent layers. Its public interface should be testable without running a full Agent conversation.

## 1. 核心上下文

```typescript
interface RuntimeContext {
  tenantId: string;
  userId: string;
  sessionId: string;
  conversationId: string;
  agentId: string;
  agentVersion: string;
  traceId: string;
  permissions: string[];
  locale?: string;
}
```

该对象由服务端构建，不能由模型 Tool Arguments 覆盖。

## 2. Capability Contract

```typescript
interface CapabilityDefinition {
  id: string;
  name: string;
  description: string;
  domain: string;
  operation: "query" | "command";
  inputSchema: JSONSchema;
  outputSchema?: JSONSchema;
  relatedApiIds: string[];
  requiredPermissions: string[];
  riskLevel: "low" | "medium" | "high" | "critical";
  approvalPolicy: "none" | "conditional" | "always";
  version: string;
}
```

## 3. Tool Discovery Contract

```typescript
interface ToolDiscoveryService {
  search(input: {
    query: string;
    domain?: string;
    operation?: "query" | "command";
    limit: number;
    context: RuntimeContext;
  }): Promise<ToolSearchResult[]>;

  describe(input: {
    toolId: string;
    version?: string;
    context: RuntimeContext;
  }): Promise<ToolDefinition>;

  resolve(input: {
    toolId: string;
    version: string;
    context: RuntimeContext;
  }): Promise<ExecutableTool>;
}
```

## 4. Tool Executor Contract

```typescript
interface ToolExecutionRequest {
  toolId: string;
  toolVersion: string;
  executionId: string;
  idempotencyKey?: string;
  arguments: unknown;
  context: RuntimeContext;
  approvalId?: string;
}

interface ToolExecutionResult<T = unknown> {
  status: "succeeded" | "failed" | "waiting" | "unknown";
  data?: T;
  error?: {
    code: string;
    message: string;
    retryable: boolean;
  };
  metadata: {
    executionId: string;
    durationMs: number;
    sourceVersion?: string;
  };
}
```

## 5. Memory Contract

```typescript
interface MemoryService {
  capture(input: RawMemoryEvent): Promise<void>;
  extract(scope: MemoryScope): Promise<MemoryCandidate[]>;
  promote(candidateId: string, reviewer?: string): Promise<MemoryItem>;
  recall(input: {
    query: string;
    scope: MemoryScope;
    tokenBudget: number;
    context: RuntimeContext;
  }): Promise<MemoryRecallResult>;
  archive(memoryId: string, reason: string): Promise<void>;
}
```

## 6. Skill Contract

```typescript
interface SkillDefinition {
  id: string;
  version: string;
  name: string;
  description: string;
  domain: string;
  trustLevel: "platform" | "organization" | "project" | "user" | "external";
  allowedToolIds: string[];
  contentUri: string;
  status: "draft" | "approved" | "published" | "deprecated";
}
```

## 7. Plugin Contract

```typescript
interface AgentPlugin {
  manifest: PluginManifest;
  initialize(ctx: PluginContext): Promise<PluginDisposer>;
  health?(): Promise<PluginHealth>;
}

type PluginDisposer = () => Promise<void>;
```

## 8. 业务案例：查询并修改学员状态

推荐步骤：
1. `search_business_capabilities("带教项目、学员、进度、状态变更")`
2. 加载 `mentorship.project.search`、`mentorship.learner.search`、`mentorship.learner.progress.read`、`mentorship.learner.status.transition`
3. 查询项目。
4. 查询张三，若同名返回候选并追问。
5. 查询实时进度和当前状态。
6. 校验状态机和权限。
7. 创建 Approval。
8. 用户确认后调用 Java Domain Command。
9. Read-after-write 验证。
10. 保存 Tool Execution、Audit、Trajectory。

模型不提供 tenantId/userId；它只提供业务参数。身份由 Runtime 注入。

## 9. 标准错误码

`TOOL_NOT_FOUND`、`TOOL_VERSION_MISMATCH`、`SCHEMA_VALIDATION_FAILED`、`PERMISSION_DENIED`、`TENANT_SCOPE_VIOLATION`、`APPROVAL_REQUIRED`、`APPROVAL_EXPIRED`、`RESOURCE_NOT_FOUND`、`RESOURCE_VERSION_CONFLICT`、`BUSINESS_RULE_REJECTED`、`IDEMPOTENCY_CONFLICT`、`EXECUTION_TIMEOUT_UNKNOWN`、`PROVIDER_RATE_LIMITED`、`PROVIDER_UNAVAILABLE`。

错误内容必须避免泄露无权资源和内部密钥。
