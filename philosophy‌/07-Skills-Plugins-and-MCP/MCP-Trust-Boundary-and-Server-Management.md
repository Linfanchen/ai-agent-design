# MCP：Server 信任边界、连接管理与工具治理

## 本文档在系统中的作用

MCP 让 Agent 平台连接外部工具服务器，但 MCP 协议互操作不等于安全授权。平台需要管理 Server 身份、工具元数据、连接生命周期、凭证、风险和调用审计。

## 1. MCP Client / Server 角色

平台作为 MCP Client，连接一个或多个 MCP Server。Server 暴露 Tools、Resources、Prompts 等能力。内部统一 Tool Contract 应包装 MCP Tool，不把 MCP 协议对象直接暴露给业务 Graph。

## 2. Server Registry

每个 Server 配置：
- server_key；
- transport_type；
- endpoint / command；
- owner；
- trust_level；
- allowed_tenants；
- credential_ref；
- allowed_tool_patterns；
- network_policy；
- timeout；
- health_status；
- protocol_version；
- reviewed_at。

本地 stdio Server 也要视为执行代码，不可随意运行未知命令。

## 3. 信任等级

- First-party：组织维护和发布；
- Approved third-party：经过审核；
- Tenant-managed：租户自行配置；
- Untrusted / pending review：仅允许隔离测试，不允许生产 Agent 使用。

信任等级影响审核和部署方式，但不替代每次调用的 Policy。

## 4. 动态工具发现

连接 Server 后读取 Tool Metadata，进行 Schema 规范化、命名空间隔离、描述审核和版本快照。远端 Tool 名称可能冲突，应使用 `server_key::tool_name` 作为内部稳定 Key。

## 5. 连接生命周期

`Configured → Validating → Connected → Discovering → Reviewed → Enabled → Draining → Disabled`

心跳失败、协议不兼容、凭证过期或 Server 被撤销时，阻止新调用并清理连接。

## 6. 安全边界

- SSRF：禁止连接内网元数据地址、云 Instance Metadata、未授权网段；
- DNS Rebinding：解析并校验最终 IP，连接前后执行策略；
- Secret：按 Server Scope 获取，不回传给模型；
- Tool Description：视为不可信输入；
- Output：校验大小、Schema 和敏感字段；
- Timeout：设置连接、读取和整体 Deadline；
- Network Egress：按 Server allowlist；
- Audit：记录 Server、Tool、Version、Principal、参数 Hash 和结果。

## 7. MCP Tool 调用路径

`Agent Discovery → Internal Registry → Policy Filter → Describe → Model Tool Call → Internal Executor → MCP Client → Remote Server → Output Validation → Audit`

模型不能直接指定任意 Server URL 或 Transport 参数。

## 8. Server 版本与变更

记录 Protocol Version、Tool Schema Hash、Server Build / Version（若可获取）。Schema 变化触发 Drift Review；删除 Tool 或修改副作用语义时阻断旧 Binding。

## 9. 测试

恶意 Tool Description、超大输出、Server 超时、DNS Rebinding、私网地址、凭证撤销、Schema 变化、Server 重启、重复调用和跨 Tenant Server 访问。
