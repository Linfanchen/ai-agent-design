# MCP Integration

## Role in the Agent System

MCP is a protocol adapter for connecting an Agent to external tools, resources, and prompts. It standardizes discovery and communication but does not provide the application's complete authorization, tenant isolation, business semantics, or audit model.

## 1. MCP boundary

MCP Server exposes capabilities. MCP Client manages connections and protocol calls. Harness adapts those capabilities into the internal Tool Registry. Policy Engine decides whether the current principal may discover and execute each capability.

## 2. Connection lifecycle

Configured → Authenticated → Initialized → Capability Discovery → Active → Degraded / Disconnected → Reconnect / Disabled.

Track server ID, endpoint or command, transport, version, capabilities, health, credential reference, tenant scope, and allowed tool list.

## 3. Tool discovery

Fetch remote schemas, normalize them into internal Tool Definitions, attach source server and version, filter by scope, and expose only selected tools. Support lazy loading when a server exposes many tools.

## 4. Authentication and secrets

Support the transport's authentication mechanism (for example OAuth or service credentials) through a Secret Manager. Never put tokens in prompts, tool descriptions, logs, or persisted graph state.

## 5. Security controls

- allowlist servers and transport endpoints;
- restrict local process launch and command arguments;
- enforce per-tool permissions;
- re-authorize at execution time;
- apply network egress rules;
- limit output size and execution time;
- treat server responses as untrusted;
- audit connection and invocation events.

## 6. Cache and refresh

Cache tool metadata with server/version/tenant keys. Invalidate on configuration change, server health failure, capability list change, or explicit refresh. Do not let stale schemas bypass current authorization.

## 7. When not to use MCP

For core business commands that need strong typed contracts, stable idempotency, and transactional semantics, a native internal Tool adapter over a domain API may be safer. MCP is an integration protocol, not a reason to externalize every internal service.
