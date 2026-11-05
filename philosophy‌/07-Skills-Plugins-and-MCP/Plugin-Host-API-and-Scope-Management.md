# Plugin Host API、Scope 注册与卸载机制

## 本文档在系统中的作用

本文件定义插件如何安全地与 Host 交互、如何注册扩展点、如何管理插件 Scope，以及卸载时如何撤销注册和资源。

## 1. Host API 设计原则

Host API 只暴露稳定能力，不暴露私有实现。插件可以请求注册 Tool、Middleware、Context Provider、Evaluator 和 Channel Adapter，但不能直接操作宿主数据库、修改 Policy 或访问 Secret 明文。

## 2. Plugin Context

PluginContext 包含 plugin_id、version、tenant_scope（若插件为租户级）、logger、metrics、secret_reference_resolver、registration_api、clock 和 cancellation scope。所有对象都有最小权限。

## 3. Registration Handle

每次注册返回 Handle：
- registration_id；
- plugin_id；
- extension_type；
- extension_key；
- version；
- scope_id；
- registered_at。

卸载插件时按 scope_id 批量注销，避免遗漏事件监听或 Tool Handler。

## 4. Lifecycle Manager

初始化顺序：验证 Manifest → 检查依赖 → 授权 → 创建隔离上下文 → 调用 setup → 验证注册结果 → 标记 Active。

卸载顺序：标记 Draining → 停止新请求 → 等待在途调用 → 取消后台任务 → 注销注册 → 关闭资源 → 清理缓存 → 标记 Disabled。

## 5. Hook 设计

Hook 有事件类型、超时、错误策略和幂等要求。非关键 Hook 失败只记录；安全 Hook 失败可阻断。禁止 Hook 在导入时自动执行副作用。

## 6. In-Process 与 Out-of-Process

In-Process 仅适合可信、已审核、同一部署供应链的插件。第三方或租户自定义插件应放在独立进程 / 容器，通过 RPC 调用。Python Import / Entry Point 不是安全隔离。

## 7. 版本兼容

Host API 采用显式 `api_version`。Plugin Manifest 声明支持范围。Host 升级时跑 Plugin Contract Test。Breaking Change 需要兼容期和迁移工具。

## 8. 观测

记录插件初始化耗时、调用次数、错误率、资源消耗、注册数、卸载耗时和异常遗留资源。Plugin 不得自行关闭 Host 的全局连接池。

## 9. 验收

插件初始化失败时无残留注册；卸载后所有 Tool Binding 禁用；在途请求按策略完成或取消；后台任务被清理；不可信插件无法访问宿主文件和密钥。
