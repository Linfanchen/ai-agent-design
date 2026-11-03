# Agent Engineering Architecture Handbook

> Reusable engineering blueprint for designing, building, evaluating, deploying, and operating professional Agent systems.
> Version: 2.0 | Language: Chinese content, English-only file and directory names | Baseline date: 2026-11-03

## Mission

This handbook is a system-level design package, not a prompt cookbook or a framework tutorial. It describes the engineering capabilities needed to turn model calls into reliable, secure, long-running Agent products.

It combines architectural lessons from:
- **AgentScope 2.0.11**: SDK building blocks, ReAct, Toolkit, Context, Event System, Permission/HITL, Middleware, Memory, Workspace/Sandbox, and its multi-tenant Agent Service capabilities.
- **DeerFlow**: the explicit Harness vs App split; Lead Agent, Middleware, Memory, Tools, Skills, Sandbox, Subagents, MCP, customization, workspace, threads, deployment, and operations.
- **LangGraph Python**: graph and functional APIs, state, nodes, edges, reducers, persistence, stores, interrupts, streaming, subgraphs, workflows/agents, fault tolerance, deployment, observability, and testing.
- **Octop / DeepSeek Harness / Hermes**: manager/runtime separation, memory pipelines, plugin composition, typed events, scopes, deferred tool discovery, and skill/provider extension patterns.

## Core architecture thesis

A professional Agent is not one prompt plus a model. It is a governed runtime composed of:

1. **Product / Application Layer** — users, channels, APIs, workspaces, threads, artifacts, administration.
2. **Harness Layer** — composition root, runtime lifecycle, middleware, policies, registries, providers, persistence adapters.
3. **Orchestration Layer** — Agent Loop, graph, workflow, planner, supervisor, subagents, interrupts, durable jobs.
4. **Intelligence Layer** — models, prompts, context, memory, RAG, business catalog, structured output.
5. **Capability Layer** — tools, skills, plugins, MCP, A2A/ACP, channels, browser and code execution.
6. **Governance Layer** — identity, tenant isolation, authorization, approval, audit, safety, privacy.
7. **Operations Layer** — tracing, evaluation, cost, latency, deployment, scaling, incident response.

## Recommended implementation baseline

For a TypeScript-first enterprise product:
- Next.js / React for product UI.
- Node.js + TypeScript for Agent API and Harness.
- LangGraph JS/TS for explicit orchestration and durable graph state.
- PostgreSQL for application truth, session metadata, event records, memory records, approvals, and catalogs.
- Redis for short-lived cache, rate limits, locks, and queue coordination.
- Object Storage for files, artifacts, and workspace blobs.
- FastAPI or isolated workers only where Python ecosystem tooling is materially better (for example, code scanning, document parsing, selected ML pipelines).
- Existing Java Spring Boot services remain the authority for enterprise business transactions and domain rules.

## How to use this package

1. Read `Documentation-Map.md`.
2. Understand the architecture in `Architecture-Overview.md`.
3. Read the three reference-project reviews before choosing patterns.
4. Read each subsystem document beginning with its **Role in the Agent System** section.
5. Use the implementation kit to convert design into interfaces, schemas, deployment topology, and acceptance tests.

## Important distinction

The reference projects have different goals and abstraction levels. Do not copy their class names or package layouts mechanically. Extract the architectural pattern, define an internal contract, and implement it in the chosen stack.

## Documentation rule

Every subsystem document starts by explaining its role in an Agent system, then covers responsibilities, boundaries, lifecycle, interfaces, state, failure modes, security, observability, and acceptance criteria where applicable.
