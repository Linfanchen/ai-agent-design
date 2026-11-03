# LangGraph Python Architecture Review

## Role in the Agent System

This review explains which orchestration primitives LangGraph should own in the proposed architecture. LangGraph is the graph runtime, not the entire Harness, product application, or business domain layer.

## 1. Documentation map

The LangGraph Python documentation is organized around:
- Overview and concepts;
- Graph API and Functional API;
- workflows and agents;
- state, nodes, edges, reducers, Command, Send, runtime context;
- persistence, checkpointers, stores, memory;
- interrupts and human-in-the-loop;
- streaming and event streaming;
- subgraphs and multi-agent composition;
- time travel;
- fault tolerance;
- deployment, local server, application structure;
- observability and Studio;
- testing and troubleshooting.

This map is a useful reference for a complete orchestration subsystem.

## 2. Core architecture semantics

### Graph API
StateGraph, state schema, nodes, edges, reducers, conditional routing, dynamic Send, Command, runtime context, recursion limit, graph compilation, and graph migrations.

### Functional API
A more code-centric model for wrapping existing functions and task steps. Choose Graph API when explicit topology and state transitions are central; choose Functional API when the workflow is naturally imperative and can be represented as durable tasks.

### Persistence
Checkpointer stores thread-scoped graph snapshots. Store holds application-defined cross-thread data. These are complementary, not interchangeable.

### Interrupts
Interrupt pauses graph execution, persists state, surfaces a JSON-serializable payload, and resumes with the same thread ID. A production approval record still belongs in the application's durable business database.

### Streaming
Streaming supports messages, updates, values, custom events, tasks, checkpoints, and debug modes. Newer event-streaming APIs offer typed projections; keep the frontend contract independent of framework-specific stream chunks.

### Subgraphs
Subgraphs support reuse, independent development, and multi-agent composition. Parent/child state sharing must be explicit.

### Fault tolerance
Retries, timeouts, and error handlers are composable. External side effects still require idempotency and reconciliation.

### Testing and observability
Build graph instances with isolated checkpointers in tests. Use traces to inspect each run and evaluate behavior.

## 3. Design decisions for this handbook

- LangGraph owns orchestration and checkpoint semantics.
- Application tables own conversation metadata, event audit, tool execution, approval, and business state.
- A durable Store owns long-term memory records or references, not the graph checkpoint.
- Business services own transactions.
- UI consumes normalized application events, not raw LangGraph internal event structures.
- Graph State must be versioned and migratable.

## 4. Python vs TypeScript

The user-provided URL is the Python LangGraph documentation. The conceptual patterns apply across the ecosystem, but API details and feature parity must be checked against the selected JS/TS version. Do not copy Python signatures into TypeScript code.

## 5. Sources

- <https://docs.langchain.com/oss/python/langgraph/overview>
- <https://docs.langchain.com/oss/python/langgraph/graph-api>
- <https://docs.langchain.com/oss/python/langgraph/workflows-agents>
- <https://docs.langchain.com/oss/python/langgraph/persistence>
- <https://docs.langchain.com/oss/python/langgraph/interrupts>
- <https://docs.langchain.com/oss/python/langgraph/streaming>
- <https://docs.langchain.com/oss/python/langgraph/use-subgraphs>
- <https://docs.langchain.com/oss/python/langgraph/fault-tolerance>
- <https://docs.langchain.com/oss/python/langgraph/deploy>
- <https://docs.langchain.com/oss/python/langgraph/observability>
- <https://docs.langchain.com/oss/python/langgraph/test>
