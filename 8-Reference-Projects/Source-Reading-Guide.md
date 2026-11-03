# Source Reading Guide

## Role in the Agent System

This guide explains how to inspect reference projects efficiently and turn source observations into architecture decisions without copying implementation details blindly.

## 1. Read in this order

### AgentScope 2.0.11
1. Versioned quickstart and concept pages.
2. ReAct, Toolkit, Model, Context, Event System.
3. Permission/HITL, Middleware, Memory, Workspace/Sandbox.
4. Agent Service: Serving, Agent Team, Channels, RAG, resource sharing, persistence, scheduling.
5. Record exact tag/commit and test the selected APIs.

### DeerFlow
1. Introduction and Harness vs App.
2. Design Principles and Lead Agent.
3. Middleware ordering.
4. Memory, Tools, Skills, Sandbox, Subagents, MCP.
5. Customization and Integration Guide.
6. App: Workspace, Agents/Threads, Deployment, Operations.

### LangGraph Python
1. Overview and Workflows/Agents.
2. Graph API, Functional API, state, reducers, nodes, edges.
3. Persistence, Checkpointers, Stores, Memory.
4. Interrupts, Streaming, Subgraphs, Time Travel.
5. Fault Tolerance, Deployment, Observability, Testing.

### Other projects
- Octop: platform architecture, Harness manager/runtime, workspace, memory.
- DeepSeek Harness: architecture, session, scope, tools, plugin extension cookbook.
- Hermes: conversation loop, prompt builder, context engine, memory provider, tool search, skills, plugins.

## 2. Source analysis template

For each subsystem, record:
- Problem solved;
- public contract;
- internal package boundary;
- lifecycle;
- state ownership;
- extension points;
- failure and recovery;
- security assumptions;
- tests and benchmarks;
- what is safe to borrow;
- what should not be copied;
- version and commit SHA.

## 3. Avoid common analysis errors

- Do not infer a feature from a README badge alone.
- Do not assume an old branch matches current docs.
- Do not mix AgentScope 1.x and 2.x.
- Do not treat a protocol as an authorization model.
- Do not infer production readiness from an example.
- Do not copy framework-specific APIs into a different language.
