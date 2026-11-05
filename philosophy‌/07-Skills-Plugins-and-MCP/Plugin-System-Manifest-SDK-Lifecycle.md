# Plugin：Manifest、SDK、注册、隔离与生命周期

## 本文档在系统中的作用

Plugin 是扩展 Agent 平台本身的机制。插件可以注册 Tool、Context Provider、Memory Provider、Middleware、Evaluator、Channel Adapter 或管理命令。它与 Skill 不同：Plugin 扩展宿主能力，Skill 提供任务知识。

## 1. 插件类型

- Trusted In-Process Plugin：由平台团队审核，运行在宿主进程。
- Isolated Worker Plugin：通过 RPC / Queue 在独立进程执行。
- Remote Plugin：通过受控 HTTP / MCP 接入。
- UI Extension：只扩展管理界面，不直接获得后端业务权限。

不可信第三方插件不应在主 API / Worker 进程中直接 import 执行。

## 2. Manifest

```yaml
plugin:
  key: example.crm-extension
  version: 1.0.0
  api_version: 1
  entrypoint: example_plugin:create_plugin
  trust_level: reviewed
  capabilities:
    - tool
    - context_provider
  permissions:
    - crm.customer.read
  dependencies:
    python: ">=3.12"
  resources:
    max_memory_mb: 256
    timeout_seconds: 10
```

Manifest 是声明，不是授权。权限还需要管理员批准，并由 Policy Engine 在每次调用时执行。

## 3. SDK 设计

插件通过稳定 Host API 注册 `register_tool()`、`register_middleware()`、`register_context_provider()`、`register_memory_provider()`、`register_evaluator()`、`register_channel_adapter()`。SDK 不暴露内部数据库 Session、密钥明文或私有 RuntimeContainer。

## 4. Python Entry Points

可使用 `importlib.metadata.entry_points(group="agent_platform.plugins")` 发现已安装插件。发现阶段只读取元数据；加载阶段先验证包来源、版本、Manifest 和信任等级。

Entry Points 不是安全沙箱。导入 Python 包会执行模块顶层代码，所以仅对可信、经过部署审核的包启用 In-Process Loading。

## 5. 生命周期

`Discovered → Verified → Installed → Configured → Initialized → Active → Draining → Disabled → Unloaded`

卸载时停止接收新调用、等待或取消在途执行、注销 Tool / Hook、关闭连接池和后台任务、清理临时资源、保留安装与审计记录。

## 6. Scope

Plugin 注册对象必须归属 Plugin Scope。卸载 Plugin 时，其 Tool Binding、Event Listener、Background Task 和 Resource Handle 应可一起撤销。

## 7. 版本兼容

Host API Version 与 Plugin Version 分开。Plugin Manifest 声明兼容范围。Host 升级前运行 Plugin Contract Tests。Breaking API 变更通过版本化 SDK 和迁移窗口处理。

## 8. 安全治理

默认最小权限、网络出口白名单、Secret Reference、CPU / Memory / Timeout 限制、依赖漏洞扫描、包 checksum / 签名、插件来源和许可证、运行时审计、紧急撤销。

## 9. 测试

插件重复注册、初始化失败、卸载清理、Hook 异常、后台任务泄漏、Host API 不兼容、越权调用和恶意依赖。

## 10. 插件配置与密钥引用

Plugin Config 只保存普通配置和 Secret Reference。密钥由 Host 的 Secret Resolver 按 Plugin Scope / Tenant Scope 获取。Plugin 不得把密钥写入日志、Manifest、错误信息或返回值。

## 11. Plugin Capability Grant

安装插件不代表授予所有权限。管理员审核 Manifest 后，对具体 Extension Type 和 Tool Permission 授权。每个插件调用仍要经过 Tool Executor / Policy。插件不能通过注册 Middleware 绕过核心 PEP。

## 12. 插件故障隔离

- 插件初始化超时：安装失败并清理；
- 插件 Hook 异常：按 Hook Criticality 决定阻断或降级；
- 插件内存泄漏：进程级监控，必要时重启隔离 Worker；
- 插件失联：熔断并标记 unavailable；
- 插件卸载时有在途任务：等待或取消并保留状态；
- 插件升级：先创建新版本并灰度，不原地覆盖。

## 13. Plugin SDK 兼容性测试

SDK 需要提供最小可运行模板、Manifest Schema、Tool / Hook Contract Test、Local Test Host、Version Compatibility Check 和安全开发指南。插件作者不应依赖宿主私有目录结构。
