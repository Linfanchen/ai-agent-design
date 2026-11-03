# AgentScope 2.0.11 Architecture Review

## Role in the Agent System

This review extracts design patterns from AgentScope 2.0.11 that can inform the SDK building-block layer and production Agent Service layer. It is a reference analysis, not a recommendation to adopt AgentScope's Python implementation in a TypeScript-first product.

## 1. Documentation navigation model

The versioned documentation entry separates quick start and core concept introduction, while the project capability map exposes:
- ReAct;
- Toolkit;
- Model;
- Context;
- Event System;
- Permission & HITL;
- Middleware;
- Memory;
- Workspace / Sandbox;
- Agent Service: serving, Agent Team, channels, RAG service, MCP & Skill Hub, resource sharing, persistence, scheduling.

The most important organizational lesson is to distinguish the **Agent SDK building blocks** from the **Agent Service / application capabilities**.

## 2. SDK-level building blocks

| Component | Role | What to borrow |
|---|---|---|
| Agent / ReAct | reasoning and action loop | clear reply and stream APIs; interrupt/resume |
| Toolkit | tool management | central registry, native tools, MCP, Skills |
| Model | provider integration | credentials and model adapter separation |
| Context | prompt/context lifecycle | compaction, tool result offload, injection |
| Event System | execution event stream | typed events for text, tools, multimodal |
| Permission & HITL | action control | resource/tool-level permission and confirmation |
| Middleware | cross-cutting hooks | composable hooks around model and tool lifecycle |
| Memory | persistent knowledge | replaceable backend providers |
| Workspace / Sandbox | isolated execution | multiple sandbox backends and scoped resources |

## 3. Agent Service capabilities

AgentScope's service capability map includes multi-tenancy, multi-session isolation, leader-worker Agent Team, channels, RAG service, MCP/Skill Hub, resource sharing, persistence, and scheduling/background task offloading.

This is a major gap to address in a Harness-only architecture: a professional Agent product needs a control plane and service layer around the SDK.

## 4. Architectural takeaways

1. Expose an Agent SDK contract and a separately deployable Agent Service.
2. Treat Event System as a first-class subsystem, not a UI afterthought.
3. Model Permission/HITL as explicit capabilities.
4. Provide resource sharing for models, MCP servers, Skills, and Workspaces.
5. Add durable scheduling and background wakeup.
6. Make sandbox a provider abstraction.
7. Use middleware for cross-cutting behavior but preserve explicit graph nodes for business workflows.

## 5. Version discipline

The user-linked docs are specifically AgentScope 2.0.11. The official quickstart pins `agentscope==2.0.11` and even demonstrates source checkout by commit. Do not mix 1.x and 2.x concepts or treat the moving `latest` documentation as the 2.0.11 contract.

## 6. What not to copy

AgentScope is Python-first. Reuse the architecture and contracts, not Python class APIs. The legacy AgentScope Runtime documentation should be treated carefully because the Runtime project announced that its capabilities were integrated into AgentScope 2.0; do not model it as an independent current runtime without checking the version and repository status.

## 7. Sources

- <https://docs.agentscope.io/zh/versions/2.0.11>
- <https://docs.agentscope.io/zh/versions/2.0.11/quickstart>
- <https://pypi.org/project/agentscope/2.0.11/>
- <https://runtime.agentscope.io/en/concept.html>
