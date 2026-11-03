# Graph, Workflow, and Execution Patterns

## Role in the Agent System

This subsystem defines the control flow of an Agent task. It determines which steps are deterministic, which steps use model judgment, how state moves, and how execution branches, loops, pauses, and resumes.

## 1. Choose the right abstraction

| Pattern | Use when | Avoid when |
|---|---|---|
| Fixed workflow | Steps and transitions are known | User intent is highly open-ended |
| ReAct loop | Model must choose next action from observations | Business process requires strict fixed order |
| Router | Choose one specialist path | Multiple specialists must collaborate |
| Parallel fan-out | Independent research/checks can run concurrently | Tasks share mutable state |
| Supervisor-subagents | Complex task needs decomposition and synthesis | Task is small and context is simple |
| Graph + workflow hybrid | Some steps deterministic, others uncertain | You are forcing all logic into LLM nodes |

LangGraph explicitly separates predetermined workflows from dynamic agents and supports mixing both in one graph.

## 2. Graph building blocks

- State Schema: typed state contract.
- Nodes: deterministic functions, model calls, tool dispatch, validation, approval, or subgraphs.
- Edges: normal, conditional, or dynamic routing.
- Reducers: define how concurrent node updates merge.
- Runtime Context: identity, configuration, dependencies, and services not treated as mutable graph state.
- Checkpointer: persistence of graph state.
- Store: cross-thread application data.
- Interrupt: pause for external input.
- Subgraph: reusable graph with an input/output contract.

## 3. State design

State should contain only what the graph needs to progress. Large documents and tool payloads should be stored externally and referenced by ID. Separate:
- user input;
- plan and task status;
- messages;
- discovered capability references;
- completed execution references;
- pending approval;
- result and evidence references;
- error/recovery metadata.

Do not put secrets, full database connections, or large binary payloads in graph state.

## 4. Reducers and concurrency

Parallel graph nodes may update state concurrently. Define merge semantics per key:
- append-only list with stable ordering;
- set union with deduplication;
- last-write-wins only where acceptable;
- conflict-detecting merge for shared entities;
- versioned compare-and-set for business state.

Never assume two parallel writes to the same state key will merge correctly without an explicit reducer.

## 5. Graph migrations

Persisted checkpoints can outlive code releases. Version State Schema and graph definitions. Define migrations or compatibility readers for renamed fields, removed nodes, changed subgraph interfaces, and changed interrupt payloads.

## 6. Practical rule

Use LangGraph as orchestration infrastructure, not as the place to hide all domain logic. Nodes should call focused services; Java business services own transaction rules.
