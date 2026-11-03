# A2A, ACP, and Agent Interoperability

## Role in the Agent System

Interoperability protocols allow an Agent to delegate work to an independently deployed remote Agent. They are different from local subagents, which share the same runtime and can use internal graph contracts.

## 1. Local subagent vs remote Agent

| Local subagent | Remote Agent |
|---|---|
| Same deployment boundary | Independent service boundary |
| Direct typed function/graph call | Network protocol |
| Shared internal tracing | Distributed trace propagation |
| Runtime can inspect child graph | Remote service owns internal execution |
| Internal permission context | Requires explicit identity delegation |

## 2. Protocol responsibilities

A protocol adapter should handle agent discovery, capability description, task submission, task status, streaming or callback updates, cancellation, authentication, and result normalization.

A2A and ACP are not substitutes for internal Tool Registry. They represent remote Agent invocation and need a separate trust model.

## 3. Delegation envelope

Include caller identity reference, tenant, delegated scope, task ID, objective, input references, expected output schema, deadline, budget, callback/return route, trace context, and cancellation token.

Do not forward bearer credentials or the entire parent conversation by default.

## 4. Trust and verification

- register trusted remote Agent endpoints;
- validate their advertised capabilities;
- authenticate service-to-service calls;
- sign or authenticate task callbacks;
- validate output schema and evidence;
- apply timeout and budget;
- treat returned content as untrusted;
- audit delegated task and remote identity.

## 5. When to use

Use local subgraphs for tight coordination and shared contracts. Use remote protocols for cross-team, cross-platform, or independently governed Agents. Keep an adapter so the orchestration graph does not depend on a single protocol.
