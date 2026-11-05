# Agent Harness 架构与组合机制

## 本文档在系统中的作用

Harness 是 Agent 的运行时外壳。它将模型、Prompt、Context、Memory、Tool Registry、Policy、Graph、事件、预算和可观测性装配为一个受控执行环境。Harness 不负责替模型推理，也不替业务系统维护业务事实；它保证一次 Agent Run 在明确依赖、权限和资源边界内运行。

## 1. Harness 解决什么问题

如果只写一个 `agent.run()`，初期很快，但会出现每个 Agent 重复创建模型客户端、Prompt / Memory / Tool 混杂、租户配置通过全局变量传递、Tool 绕过权限、资源没有统一关闭、单测依赖真实模型和数据库、运行中配置漂移、不同入口无法复用 Agent 等问题。

Harness 将这些问题收敛为运行时装配和生命周期机制。

## 2. 核心组件

| 组件 | 主要职责 | 不负责 |
|---|---|---|
| RuntimeFactory | 根据 Release 与 Principal 创建 Run Scope | 不执行 Graph |
| RuntimeContainer | 持有本次运行可访问的依赖 | 不做业务授权决策 |
| ProviderRegistry | 选择模型 Provider / Model Profile | 不拼接业务 Prompt |
| PromptBuilder | 组装 Prompt Sections | 不决定权限 |
| ContextCompiler | 召回、筛选、排序、预算和渲染上下文 | 不执行业务 Tool |
| MemoryService | 记忆候选、检索、更新 | 不作为业务事实来源 |
| ToolRegistry | Tool 元数据和版本 | 不直接绕过 Executor |
| ToolExecutor | 参数验证、策略、审批、执行、审计 | 不决定下一 Graph Node |
| PolicyEngine | 对 Principal、Action、Resource 做决策 | 不接受模型授予权限 |
| GraphFactory | 创建 LangGraph Graph / Compiled Graph | 不管理 API Request |
| EventPublisher | 发送持久事件和实时事件 | 不承担业务事务 |
| UsageMeter | 预算和成本计量 | 不决定业务结果 |

## 3. Harness 与 Graph 的边界

Graph 负责“流程如何推进”；Harness 负责“执行环境和能力如何提供”。Graph Node 只依赖稳定 Runtime Ports，例如 `runtime.model`、`runtime.tools`、`runtime.context`。它不应知道 Provider SDK、数据库连接池、FastAPI Request 或 Docker 客户端。

Harness 在 Graph 编译前完成依赖装配；每次运行通过可信 Runtime Context / Runnable Config 传入 tenant_id、run_id、principal_ref 等标识。敏感对象不放入可序列化 Graph State。

## 4. Runtime Scope

每次 Run 创建一个 Scope：
- `run_id`、`thread_id`、`tenant_id`；
- `agent_release_id`；
- `principal_ref`；
- `budget`；
- `deadline`；
- `cancellation_token`；
- `trace_context`；
- `resource_stack`；
- `policy_snapshot`。

Scope 不能跨租户复用。一个 Conversation 可以有多个 Run，但 Run Scope 独立。

## 5. 生命周期

```text
Resolve Release
  → Validate Principal / Tenant
  → Acquire Quota
  → Build Run Scope
  → Open Resource Stack
  → Build Provider / Context / Tool Executor
  → Build or Load Graph
  → Execute / Stream
  → Persist Result / Usage / Events
  → Close Run Resources
  → Release Unused Quota
```

进程级资源（DB Engine、HTTP Pool）属于 Application Lifespan；Run 级资源（临时目录、Run-scoped adapter、trace span）属于 Runtime Scope；Step 级资源（短 DB Session、HTTP response）必须在 Step 完成时关闭。

## 6. Factory 职责与示例

Factory 输入是经过验证的 `RunSpec`，输出可执行的 `AgentRuntime`。Factory 不接受任意 Python 类名、Import Path 或用户提供的 callable，否则会成为远程代码执行入口。

```python
class RuntimeFactory:
    def __init__(self, release_repo, provider_factory, tool_registry,
                 policy_factory, context_factory, graph_factory):
        self.release_repo = release_repo
        self.provider_factory = provider_factory
        self.tool_registry = tool_registry
        self.policy_factory = policy_factory
        self.context_factory = context_factory
        self.graph_factory = graph_factory

    async def create(self, run_spec, principal):
        release = await self.release_repo.get_published(run_spec.release_id)
        self._validate_scope(release, run_spec, principal)
        scope = await self._create_scope(run_spec, principal, release)
        provider = self.provider_factory.build(release.model_profile)
        policy = self.policy_factory.build(release.policy_profile, principal)
        tools = self.tool_registry.bind(release.tool_bindings, scope, policy)
        context = self.context_factory.build(release.context_profile, scope)
        graph = self.graph_factory.build(release.graph_key, provider, tools, context)
        return AgentRuntime(scope=scope, graph=graph)
```

示例省略类型和异常映射。真实实现应使用 Protocol、Pydantic DTO 和明确的 async context manager。

## 7. 依赖注入策略

- 进程级：FastAPI lifespan 创建 Engine、HTTP Client、Redis Pool、Telemetry。
- 请求级：认证后的 Principal、Correlation ID。
- Run 级：Runtime Scope、预算、策略、Tool Binding。
- Step 级：AsyncSession、外部 HTTP Response、临时文件句柄。

依赖注入应显式、可测试。不要使用模块级全局可变对象保存租户配置。

## 8. 资源释放

Runtime Scope 实现 async context manager，内部使用 `contextlib.AsyncExitStack` 注册资源。即使 Graph 抛异常或任务被取消，也必须执行清理。清理失败记录为 Secondary Failure，不能覆盖原始错误。

## 9. 多租户隔离

Runtime 创建前检查 Tenant 与 Agent Release 绑定；Provider Credential 通过 Secret Reference 获取；Tool Registry 返回租户可用 Binding；Memory / RAG Search 带 Tenant Scope；Workspace 使用租户隔离路径或 Bucket；缓存 Key 必须含 Tenant 与版本；Graph State 不放密钥或完整 Principal。

## 10. 可测试性

Harness 测试不应需要真实 LLM：Factory Test、Scope Test、Tenant Test、Release Test、Cancellation Test、Provider Test、Tool Test。测试环境注入 FakeProvider、InMemoryCheckpointer、FakeToolExecutor。

## 11. 反模式

一个全局 `agent = Agent()` 服务所有租户；Graph Node 内直接 `os.getenv()` 取密钥；每个 Node 自己 new HTTP Client；Tool 绕过 Executor 直接写库；从数据库动态加载任意 Python Import Path；把 RuntimeContainer 序列化到 Checkpoint；把进程级资源和 Run 级资源混为一体。

## 13. Harness 的运行时不变量

Harness 在任何 Run 中都应保证以下不变量：

1. 每个 Run 只绑定一个不可变 Agent Release；
2. 每个 Run 只有一个明确 Tenant Scope；
3. 所有模型调用都通过 Model Port；
4. 所有模型 Tool Call 都经过 Tool Executor；
5. 所有业务写操作都有执行记录；
6. 所有外部副作用都具备幂等或对账策略；
7. Graph State 只保存可序列化数据；
8. 任何资源都有唯一 Owner 和释放点；
9. 任何高风险能力都受 Policy / Approval 控制；
10. Runtime 退出后不遗留 Run 级资源。

这些不变量应作为架构测试，而不只是开发规范。

## 14. Run Scope 的推荐结构

```python
from dataclasses import dataclass
from datetime import datetime
from typing import Protocol

class CancellationPort(Protocol):
    def is_cancelled(self) -> bool: ...
    def raise_if_cancelled(self) -> None: ...

@dataclass(frozen=True)
class RunScope:
    tenant_id: str
    principal_id: str
    run_id: str
    thread_id: str
    release_id: str
    catalog_snapshot_id: str
    policy_snapshot_id: str
    deadline_at: datetime
    cancellation: CancellationPort
    trace_id: str
```

Scope 只保存身份引用和不可变运行信息。可执行依赖由 RuntimeDependencies 注入；不要把 Scope 序列化到 Checkpoint 后再恢复完整对象。

## 15. Compiled Graph 缓存策略

Compiled Graph 可以按 `graph_key + graph_code_version + state_schema_version` 缓存。缓存中不得保存租户密钥、Principal、Run State、Approval 或 Tenant-specific Tool Authorization。

如果 Graph Node 闭包捕获了 Runtime Scope，就不能把这个 Compiled Graph 跨 Run 共享。正确做法是让 Node 通过可信 Runtime Context 获取本次 Run 依赖。

## 16. 运行时配置冻结

Run 创建时应生成 `RuntimeManifest`，记录 Provider、Model Profile、Prompt、Tool Version、Skill、Plugin、Policy、Catalog 和 Graph 版本。即使 Control Plane 随后发布新配置，本 Run 仍按原 Manifest 执行，安全撤销除外。

## 17. 装配失败分类

- `RELEASE_INVALID`：Release 结构不完整；
- `DEPENDENCY_MISSING`：Tool / Skill / Plugin 依赖缺失；
- `PROVIDER_UNAVAILABLE`：模型配置不可用；
- `POLICY_PROFILE_INVALID`：策略未发布或不兼容；
- `CATALOG_SNAPSHOT_MISSING`：业务目录不可用；
- `RESOURCE_ALLOCATION_FAILED`：预算或资源分配失败。

装配失败应在 Run 进入 `running` 前完成记录，避免 UI 长时间显示“正在执行”。

## 18. Harness 架构测试

- 不允许 Graph Node 导入具体数据库 Adapter；
- 不允许 Runtime Factory 返回未绑定 Policy 的 Tool Executor；
- 不允许缺少 Release Snapshot 的 Run；
- 不允许跨 Tenant 复用 Run Scope；
- 不允许资源释放失败覆盖主执行异常；
- 不允许编译缓存捕获 Principal / Secret；
- 不允许 Worker 在没有 Lease 的情况下执行 Run。
