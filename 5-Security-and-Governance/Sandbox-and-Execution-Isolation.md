# 19. Workspace、Sandbox 与执行环境

## Role in the Agent System

Provides isolated file, shell, browser, and code execution environments.

**Primary responsibility:** Own this capability as a reusable subsystem with explicit contracts, state ownership, security boundaries, and observable outcomes.

**Boundary:** This subsystem must not silently take over responsibilities owned by adjacent layers. Its public interface should be testable without running a full Agent conversation.

## 1. Workspace 与业务数据分离

Workspace 是 Agent 工作文件空间，例如用户上传的文档、Agent 生成的报告、项目说明文件、Skill references、临时分析文件和任务交付物。它不是业务数据库，也不应作为用户权限和业务状态的权威来源。

## 2. Backend Abstraction

可支持 Local FS（开发）、S3 / COS（生产对象存储）、PostgreSQL metadata + Object Storage、Tenant-scoped Workspace、Read-only Workspace、Ephemeral Task Workspace。

```typescript
interface WorkspaceBackend {
  list(path: string): Promise<WorkspaceEntry[]>;
  read(path: string): Promise<Uint8Array>;
  write(path: string, data: Uint8Array, options?: WriteOptions): Promise<void>;
  delete(path: string): Promise<void>;
  stat(path: string): Promise<WorkspaceStat>;
}
```

所有路径必须做根目录约束、租户隔离和权限校验。

## 3. Sandbox

需要终端、代码执行、浏览器自动化时，必须使用 Sandbox：
- 独立进程或容器
- CPU / Memory / Time 限制
- 网络 egress policy
- 文件系统 Mount allowlist
- Secret 不默认暴露
- 进程树清理
- 日志和产物收集

企业业务 Agent 默认不应拥有 Shell。按 Agent / Task Scope 单独授权。

## 4. Browser / Computer Use

浏览器是高权限工具：
- 使用专用 Browser Context。
- Cookie / Credential Vault 隔离。
- 网站域名 allowlist。
- 下载文件隔离扫描。
- 页面内容作为不可信输入。
- 对外提交、支付、删除等动作必须审批。

## 5. Long-running Task

长任务应创建 Job ID、持久化 Job State、定期 heartbeat、支持取消、结果写入 Workspace / Artifact Store、Runtime Checkpoint 保存 Job Reference，Job 重试必须幂等。

## 6. 参考项目启示

Octop 的 BackendWorkspace 强调通过统一 Workspace API 访问本地或远程后端。Hermes 和 DeepSeek Harness 将 Terminal、Shell、Browser、Sandbox 作为独立能力接入，而不是 Agent Loop 的内置硬编码。
