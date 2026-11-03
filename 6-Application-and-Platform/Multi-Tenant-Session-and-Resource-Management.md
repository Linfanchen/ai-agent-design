# Multi-Tenant Session and Resource Management

## Role in the Agent System

This subsystem manages the ownership and isolation of Agents, users, conversations, threads, runs, memory, workspaces, tools, skills, and shared resources in a multi-user product.

## 1. Resource hierarchy

Organization/Tenant → Users and Groups → Agent Definitions → Conversations → Threads → Runs/Turns → Tool Executions and Artifacts.

A product may map one conversation to one thread or allow multiple Agent threads under one conversation. Define this explicitly.

## 2. Ownership model

Every resource should have tenant ID and owner/scope metadata. Shared resources need explicit visibility such as private, team, tenant, or public catalog.

## 3. Isolation

- DB queries must enforce tenant scope.
- Object storage keys must include tenant and resource scope.
- Redis keys must include tenant.
- Memory namespaces must include tenant and subject scope.
- Tool discovery must filter by tenant and user permissions.
- Workspace paths must be tenant-scoped.
- Background jobs must revalidate tenant and permissions at execution time.

## 4. Session semantics

Define:
- conversation: user-facing grouping;
- thread: durable runtime state/checkpoint scope;
- run: one invocation;
- turn: user request lifecycle;
- message: conversation content;
- child thread: subagent or delegated execution.

Do not overload one `session_id` for all of these.

## 5. Resource sharing

Support group/org-level sharing for model profiles, MCP servers, skills, workspaces, and Agent templates. Sharing a resource does not automatically grant every member permission to invoke all actions it exposes.

## 6. Retention and quotas

Per-tenant limits for concurrent runs, stored files, memory size, token usage, model spend, tool calls, and background jobs. Retention policies must cover sessions, events, traces, checkpoints, and artifacts.
