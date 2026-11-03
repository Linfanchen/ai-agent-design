# Architecture Decision Records

## Role in the Agent System

ADRs record the decisions that constrain the whole platform. Each decision should have an owner, status, context, decision, alternatives, consequences, and revisit trigger.

## ADR-001 — LangGraph as the primary orchestration runtime
**Status: Accepted.** Use LangGraph JS/TS for graph state, routing, subgraphs, persistence integration, interrupts, and durable execution. Keep domain rules outside graph nodes.

## ADR-002 — Modular monolith first
**Status: Accepted.** Deploy a modular Harness with separate background and sandbox workers as needed. Split services only for scale, isolation, or ownership.

## ADR-003 — Harness and Agent Application are separate layers
**Status: Accepted.** The Harness is reusable runtime infrastructure; the App owns users, channels, UI, billing, workspaces, and product-specific workflows.

## ADR-004 — Business Catalog and Tool Registry are separate
**Status: Accepted.** Catalog describes the whole system; Registry exposes approved executable capabilities; model-visible tools are a scoped subset.

## ADR-005 — Generic query, domain-specific commands
**Status: Accepted.** Use constrained entity query tools for reads. Use typed domain commands for writes. No arbitrary SQL update tool.

## ADR-006 — Memory and thread checkpoint are separate
**Status: Accepted.** Checkpoints preserve thread execution state; Memory stores cross-thread reusable knowledge.

## ADR-007 — Durable Event Log and graph checkpoint are separate
**Status: Accepted.** Event Log supports audit and reconstruction; Checkpoint supports efficient resume.

## ADR-008 — Skill is not Plugin
**Status: Accepted.** Skill is a procedural knowledge package; Plugin is executable extension code.

## ADR-009 — Middleware is cross-cutting; graph nodes are explicit workflow steps
**Status: Accepted.** Do not hide critical business process transitions inside opaque middleware.

## ADR-010 — Tool execution is a governed side effect
**Status: Accepted.** Reauthorize at execution, enforce idempotency, model UNKNOWN outcomes, and read-after-write verify important mutations.

## ADR-011 — Multi-Agent delegation uses least privilege
**Status: Accepted.** Child Agents receive task-scoped context and capability grants, not the parent's entire history and permissions.

## ADR-012 — Event streaming is a typed application contract
**Status: Accepted.** UI consumes normalized events, not provider or graph framework internals.

## ADR-013 — Provider adapters normalize, not erase, provider differences
**Status: Accepted.** Capability descriptors expose feature support; unsupported capabilities fail or degrade explicitly.

## ADR-014 — Production changes are versioned and evaluated
**Status: Accepted.** Agent, Prompt, Tool, Skill, Plugin, Graph, Catalog, and Config changes require tests and rollback.
