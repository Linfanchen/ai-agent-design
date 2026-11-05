# MCP：Python Client / Server 接入与治理

## 本文档在系统中的作用

MCP 提供统一协议，使 Agent Host 能连接外部工具服务器，并发现 Tools、Resources 和 Prompts。MCP 是互操作协议，不是内部权限系统，也不是对远程服务的信任背书。

## 1. MCP 三类原语

- Tools：模型驱动的动作；
- Resources：应用决定加载的上下文资源；
- Prompts：用户选择的可复用消息模板。

三者控制方不同，不能把 Resource 自动当 Tool，也不能把 Prompt 当平台 Policy。

## 2. MCP Client Adapter

负责建立传输连接、协议版本协商、列举 Server Capabilities、发现 Tools / Resources / Prompts、归一化 Schema、维护连接生命周期、处理超时取消错误，并附加身份与审计上下文。

## 3. Server Registry

每个 MCP Server 保存 server_id、tenant_id、endpoint / transport、auth mode、credential_ref、allowed tool names、network policy、trust level、server version、health status、last discovery time。

禁止任意用户提交 URL 后让 Worker 直接访问，否则可能产生 SSRF。Endpoint 必须经过管理员配置和网络策略审核。

## 4. Tool Discovery 映射

MCP Tool 进入内部 Tool Registry 时包装为 Remote Tool Binding：原始 MCP Tool Name、MCP Server ID、Schema Snapshot、Remote Version / Last Seen、Timeout、Auth Reference、Risk Level、Internal Permission Mapping。

每次调用重新检查 Server 和 Tool 是否仍被允许。

## 5. Python SDK

官方 Python SDK 提供 MCP Server / Client 能力，支持标准传输。V4 将 MCP SDK 封装在 Adapter 内，业务代码不直接依赖 SDK 对象。官方文档当前区分 v2 稳定线与 v1 维护线，实施时必须锁定版本并按 Migration Guide 处理。

## 6. MCP Server 暴露原则

不要把整个内部数据库、所有 API 或任意 SQL 作为 MCP Tools 暴露。只暴露有明确 Schema、权限和业务语义的 Capability。Resources 也必须做 ACL 和 Tenant 过滤。

## 7. 安全重点

SSRF / DNS Rebinding、OAuth Token / Secret 管理、Tool Description Prompt Injection、远程 Server 被替换、Schema 突变、结果内容包含恶意指令、远程调用超时但副作用已发生、跨租户 Server 复用。

## 8. 测试

使用本地测试 Server / In-Memory Client 验证工具发现、Schema、错误、取消、连接重建、授权、传输兼容和恶意响应处理。
