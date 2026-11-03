# Documentation Map

## 1. How the documentation is organized

The left-navigation structure is designed as a learning and implementation journey rather than a list of disconnected technologies.

| Section | Question answered | Main output |
|---|---|---|
| 00 Start Here | What is the whole system and how should I read it? | Mental model and architecture map |
| 01 Foundations | What is an Agent, how is it configured, and how does it use models? | Agent definition and model contracts |
| 02 Runtime and Orchestration | How does work execute, pause, resume, delegate, and stream? | Runtime and workflow design |
| 03 Context and Knowledge | What information does the Agent know and how is it retrieved? | Context, Memory, RAG, Business Catalog |
| 04 Capabilities and Extensions | How does the Agent act and gain new capabilities? | Tools, Skills, Plugins, MCP, protocols |
| 05 Security and Governance | Who can do what, where, and with whose approval? | Security and control plane |
| 06 Application and Platform | How does the runtime become a product or service? | APIs, UI, sessions, tenancy, configuration |
| 07 Operations and Quality | How do we test, observe, deploy, scale, and recover? | Production operations |
| 08 Reference Projects | What can be learned from existing systems? | Evidence-based design decisions |
| 09 Implementation Kit | How do we turn this into code and delivery? | Contracts, schema, roadmap, checklists |

## 2. Recommended reading paths

### Architecture designer
1. Architecture Overview
2. Design Principles
3. Agent System Concepts
4. Harness and Runtime
5. Graph Workflow and Execution Patterns
6. Context Engineering
7. Tool System and Discovery
8. Security and Governance
9. Reference Architecture and Directory Structure

### Runtime engineer
1. Harness and Runtime
2. Agent Loop and ReAct
3. Session State and Checkpoint
4. Middleware, Hooks and Policies
5. Event System and Streaming
6. Fault Tolerance and Recovery
7. Deployment and Scaling

### Multi-Agent engineer
1. Agent Definition and Identity
2. Planning, Todo and Clarification
3. Multi-Agent and Subagents
4. Graph Workflow and Execution Patterns
5. Long-Running Tasks and Scheduling
6. A2A / ACP Interoperability
7. Evaluation and Testing

### Context / memory engineer
1. Context Engineering
2. Memory Architecture
3. RAG and Knowledge Retrieval
4. Business System Intelligence
5. Workspace and Artifact Management
6. Cost, Latency and Model Optimization

### Capability engineer
1. Tool System and Discovery
2. Skills Engineering
3. Plugin Architecture
4. MCP Integration
5. Sandbox and Execution Isolation
6. Tool Audit and Execution Reliability

### Product / platform engineer
1. Harness vs Agent Application
2. Agent Service and API Design
3. Multi-Tenant Session and Resource Management
4. Channels and Gateway
5. Frontend Streaming and UX
6. Configuration and Secrets Management
7. Deployment and Scaling

## 3. How to read each subsystem document

Every subsystem document uses a common template:
1. Role in the Agent System
2. Problem and design goals
3. Responsibilities and non-responsibilities
4. Architecture and lifecycle
5. Data model and interfaces
6. Runtime interaction / sequence
7. Security and failure handling
8. Observability and evaluation
9. Implementation recommendations
10. Acceptance checklist

Not every subsystem needs all ten sections, but the role and boundary must always be explicit.

## 4. Reference-project navigation

- AgentScope 2.0.11: use for SDK-level building blocks and production Agent Service features.
- DeerFlow: use for the separation of Harness and product application, plus the practical subsystem navigation.
- LangGraph Python: use for graph semantics, durable execution, persistence, interrupts, streaming, subgraphs, fault tolerance, and testing.
- Octop: use for multi-agent platform and Memory architecture.
- DeepSeek Harness: use for plugin composition, scope, events, and session design.
- Hermes: use for dynamic Tool Search, Skills, Memory Provider, and Context Engine patterns.
