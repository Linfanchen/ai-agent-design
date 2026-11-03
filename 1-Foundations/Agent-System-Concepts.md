# Agent System Concepts

## Role in the Agent System

This document establishes the vocabulary and mental model used by every other subsystem. It prevents overloaded words such as Agent, Session, Memory, Tool, Skill, and Workflow from being used interchangeably.

## 1. Agent system vs LLM application

An LLM application often maps input to output. An Agent system maintains state, chooses actions, invokes tools, observes results, changes plans, handles interruptions, and may continue across multiple requests.

A production Agent therefore needs:
- an objective and operating policy;
- a model and prompt strategy;
- a stateful runtime;
- a set of allowed capabilities;
- context and knowledge sources;
- persistence and recovery;
- identity, authorization, and approval;
- observability and evaluation;
- a product surface through which humans interact.

## 2. Core terms

| Term | Definition |
|---|---|
| Agent Definition | Versioned configuration of role, model, tools, skills, policy, and runtime profile |
| Agent Instance | A configured Agent attached to a specific execution scope |
| Runtime | The process and services that execute an Agent |
| Harness | The composition and governance environment around the runtime |
| Graph | Nodes and transitions defining execution state and control flow |
| Workflow | A predetermined business process |
| Agent Loop | Repeated model reasoning and action cycle |
| Thread | Persistent execution/conversation continuity boundary |
| Session | Application-level conversation or execution container; exact meaning must be defined by product |
| Turn | One user request and its completion lifecycle |
| Step | One model or graph execution step |
| Tool | A callable action |
| Skill | A reusable procedure and domain instruction package |
| Plugin | An installable code extension |
| Memory | Retained knowledge intended for future recall |
| Context | Information supplied to a specific model call |
| Workspace | Files and working resources for an Agent task |
| Artifact | A user-visible output such as report, file, chart, or code package |
| Trajectory | A structured record of how a task was completed |

## 3. The capability taxonomy

- **Model capability**: reasoning, structured output, vision, audio, embeddings.
- **Orchestration capability**: graph, loop, routing, parallelism, delegation.
- **Information capability**: Context, Memory, RAG, Business Catalog.
- **Action capability**: Tool, MCP, browser, code execution, domain command.
- **Procedure capability**: Skill, SOP, workflow template.
- **Extension capability**: Plugin, middleware, provider adapter.
- **Governance capability**: identity, permission, approval, audit, safety.
- **Product capability**: channel, workspace, thread, artifact, admin console.

## 4. The capability composition rule

A user task should be translated into a task plan, then into required knowledge, tools, skills, workflows, and approvals. Do not choose an Agent architecture based on the number of tools or the popularity of a framework.
