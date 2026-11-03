# Glossary

## Role in the Agent System

This glossary provides a shared language across architecture, product, runtime, and operations teams.

| Term | Definition |
|---|---|
| Agent | Goal-oriented system that reasons, acts, observes, and adapts |
| Agent Definition | Versioned blueprint of an Agent's behavior and capabilities |
| Agent Instance | Agent Definition bound to a specific runtime scope |
| Harness | Reusable runtime composition and governance environment |
| Runtime | Execution host for Agent tasks |
| Agent Loop | Model reasoning and action cycle |
| ReAct | Reasoning and acting pattern |
| Graph | Nodes and edges representing execution |
| Workflow | Predetermined process with controlled transitions |
| State | Structured runtime data |
| Checkpoint | Persisted graph state snapshot |
| Store | Cross-thread application data storage |
| Thread | Durable graph execution continuity scope |
| Session | Application-defined conversation/execution container |
| Turn | One user request lifecycle |
| Step | One model or graph step |
| Event | Typed occurrence in the execution lifecycle |
| Context | Information visible to one model request |
| Memory | Retained information intended for future recall |
| RAG | Retrieval-Augmented Generation |
| Tool | Callable action exposed to an Agent |
| Tool Registry | Tool metadata and handler catalog |
| Tool Discovery | Search/describe/load tools on demand |
| Skill | Reusable procedural knowledge package |
| Plugin | Installable code extension |
| Middleware | Lifecycle wrapper for cross-cutting behavior |
| Hook | Named extension point |
| Policy | Server-side authorization and governance rule |
| HITL | Human-in-the-loop interaction or approval |
| Sandbox | Isolated execution environment |
| Workspace | Scoped working files/resources |
| Artifact | User-visible deliverable |
| Subagent | Child Agent executing a delegated task |
| A2A / ACP | Agent interoperability protocols |
| MCP | Model Context Protocol |
| Trajectory | Structured execution record |
| Evaluation | Measurement of Agent task quality and safety |
