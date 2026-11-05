# Harness 装配顺序、依赖图与运行时构建

## 本文档在系统中的作用

本文件把 Harness 从“组件列表”进一步落实为一个可验证的依赖装配过程。重点是组件创建顺序、依赖图、循环依赖控制、配置快照和失败时的部分资源清理。

## 1. 装配不是随意 new 对象

一次 Run 的装配需要先解析配置，再创建能力实例。推荐顺序：

1. 读取 Agent Release；
2. 验证 Release Published 且未撤销；
3. 解析 Model Profile / Policy / Context / ToolSet；
4. 校验所有版本引用完整；
5. 解析 Principal / Tenant Scope；
6. 预留预算；
7. 创建 RunResourceScope；
8. 创建或取得进程级 Provider Client；
9. 构造 Run-scoped Model Adapter；
10. 构造 Policy Context；
11. 构造 Tool Binding View；
12. 构造 Context Compiler；
13. 构造 Graph Dependencies；
14. 创建或加载 Compiled Graph；
15. 绑定 Checkpointer / Runtime Config；
16. 输出 AgentRuntime。

不要在第 8 步之前创建外部连接，不要在 Release 未验证前加载插件。

## 2. 依赖图

```text
RuntimeFactory
 ├─ ReleaseRepository
 ├─ Principal / Tenant Resolver
 ├─ BudgetService
 ├─ ResourceScopeFactory
 ├─ ProviderFactory
 │   └─ HTTP Client Pool
 ├─ PolicyFactory
 ├─ ToolRegistry
 │   ├─ Tool Metadata Store
 │   └─ Handler Registry
 ├─ ContextFactory
 │   ├─ History Provider
 │   ├─ Memory Provider
 │   ├─ Catalog Provider
 │   ├─ RAG Provider
 │   └─ Token Budgeter
 ├─ GraphFactory
 │   ├─ Graph Builder Registry
 │   └─ Checkpointer
 └─ Observability
```

依赖图必须是有向无环图。若 ToolRegistry 需要 Policy，而 Policy 又需要 ToolRegistry 的数据，使用 Tool Metadata Port 或 Policy Input DTO 解决，不要互相持有完整对象。

## 3. Run-scoped Dependency Container

Run Container 可以暴露只读依赖集合，但不要提供无约束的字符串 Service Locator。

```python
from dataclasses import dataclass

@dataclass(frozen=True)
class RuntimeDependencies:
    model: object
    context: object
    tools: object
    policy: object
    budget: object
    events: object
    repositories: object

@dataclass(frozen=True)
class RuntimeIdentity:
    tenant_id: str
    principal_id: str
    run_id: str
    thread_id: str
    release_id: str
```

`RuntimeDependencies` 中对象应实现明确 Protocol；上例使用 object 只是为了缩短示意。

## 4. 编译 Graph 与创建 Runtime 分开

Graph Builder 通常只依赖 Node 函数和稳定 Port。若 Graph 结构不随租户变化，可以缓存 Compiled Graph；但不能把租户 Principal、Tool Authorization、API Key 或 Run Scope 缓存在共享 Graph 实例中。

建议：
- Graph Definition / Compiled Graph：进程级或版本级缓存；
- Runtime Dependencies：Run 级；
- Principal / Policy Context：Run 级并在 Tool Invoke 重新验证；
- DB Session：Step 级。

## 5. 装配失败回滚

Factory 可能在创建 Provider 后创建 Tool Binding 失败。使用 AsyncExitStack 保证已创建资源按逆序关闭。预算预留失败时释放已申请的租约。Factory 只有在所有必要依赖成功后才返回 Runtime。

## 6. 配置完整性校验

Release Manifest 需要校验：
- graph_key 在 GraphBuilderRegistry 中存在；
- Model Profile 能力满足 Graph 要求；
- Tool Version 全部存在；
- Skill Package Checksum 一致；
- Plugin API Version 兼容；
- Policy Profile 已发布；
- Catalog Snapshot 可用；
- Checkpoint State Version 可迁移。

## 7. 缓存边界

可缓存：Compiled Graph、只读 Schema、公开 Model Capability、已发布 Catalog 索引。
谨慎缓存：Tenant Policy、Tool Binding、Credential Ref。
不缓存为全局对象：Principal、Authorization Result、AsyncSession、Run State、Approval State。

## 8. 验收

- Release 缺失依赖时 Factory 失败且不泄漏资源；
- 两个 Tenant 并发运行时不存在配置串用；
- 同一个 Compiled Graph 可以服务不同 Run，但不会共享 Run Scope；
- Plugin 被撤销后新 Invoke 失败；
- Factory 取消时所有已创建资源都释放。
