# DeerFlow Harness and App Architecture Review

## Role in the Agent System

This review extracts DeerFlow's strongest contribution: a clear separation between a reusable Harness and a reference application, plus a practical subsystem-oriented documentation tree.

## 1. Documentation information architecture

The documentation is grouped into:
- Introduction: what it is, why, core concepts, Harness vs App.
- DeerFlow Harness: install, quick start, design principles, Lead Agent, middlewares, configuration, memory, tools, skills, sandbox, subagents, MCP, customization, integration guide.
- DeerFlow App: overview, quick start, deployment, configuration, workspace, agents/threads, operations/troubleshooting.
- Tutorials: task-oriented how-to.
- Reference: model providers and detailed reference.

This structure is useful because it separates conceptual learning, reusable runtime engineering, product deployment, hands-on learning, and lookup material.

## 2. Harness subsystem lessons

### Lead Agent
Treat the Lead Agent as the primary coordinator and user-facing reasoning loop. Keep the model, system instructions, tool set, and delegation strategy configurable.

### Middleware
DeerFlow shows a concrete ordered chain: input/tool sanitization, dynamic context, skill activation, durable context, summarization, todo, token usage, memory, image handling, deferred tool filtering, subagent limits, loop detection, token budget, and clarification. The order is part of the architecture.

### Tools
It distinguishes built-in tools, community tools, MCP tools, and skill tools. Tool groups restrict custom Agents. Tool Search lazily loads full schemas when many tools are available.

### Skills
Skills are task-oriented capability packages with `SKILL.md`, references, workflows, dependencies, security scanning, loading, and optional evolution. Skills are not merely prompts.

### Sandbox
Execution is abstracted behind a provider and must isolate files, shell, and browser activity. Host shell is not a security boundary.

### Subagents
Delegation is a first-class tool, with limits for concurrency and total calls, task contracts, and separate agent loops.

### MCP / Customization
MCP config is separated from core runtime config; extension points allow custom middleware, tools, skills, and replacement of built-in components.

## 3. App-layer lessons

The application adds:
- agent definitions/custom Agents;
- threads and checkpoint persistence;
- workspace workflows;
- deployment configuration;
- Gateway APIs;
- operations and troubleshooting;
- channels and user interaction.

A production architecture must therefore include more than the Harness SDK.

## 4. Design takeaways

1. Separate `harness` and `application`.
2. Create a subsystem doc for Lead Agent, Middleware, Memory, Tools, Skills, Sandbox, Subagents, MCP, Workspace, Threads, and Operations.
3. Treat middleware order as a reviewed configuration.
4. Use Skill allowlists per Agent.
5. Use Tool Groups and Deferred Tool Search to control context size.
6. Treat dynamic skill evolution as a privileged operation requiring security review.
7. Keep application config and runtime config distinct.

## 5. Sources

- <https://deerflow.tech/zh/docs>
- <https://deerflow.tech/zh/docs/harness/middlewares>
- <https://deerflow.tech/zh/docs/harness/tools>
- <https://deerflow.tech/zh/docs/harness/skills>
- <https://deerflow.tech/zh/docs/harness/mcp>
- <https://deerflow.tech/zh/docs/harness/customization>
- <https://deerflow.tech/zh/docs/application/agents-and-threads>
- <https://deerflow.tech/zh/docs/application/deployment-guide>
