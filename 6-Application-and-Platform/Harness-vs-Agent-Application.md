# Harness vs Agent Application

## Role in the Agent System

This document draws the boundary between the reusable runtime foundation and the product built on top of it. It prevents product UI, user management, billing, and domain workflows from leaking into the core Agent engine.

## 1. Harness

Owns:
- Agent Runtime construction and lifecycle;
- model, tool, memory, context, skill, and plugin composition;
- middleware and hooks;
- state/checkpoint adapters;
- execution policies and runtime contracts;
- stream events and cancellation semantics.

## 2. Agent Application

Owns:
- user-facing UI and interaction;
- accounts, organizations, tenant membership;
- Agent catalog and configuration screens;
- conversation/thread management;
- file uploads, workspaces, artifacts;
- billing, quotas, usage, and plans;
- channel configuration;
- admin, audit search, and operations workflows.

## 3. Why separate them

The same Harness can power multiple products: internal HR Agent, research Agent, coding Agent, customer support Agent, or a vertical SaaS. Product requirements change more frequently than core runtime semantics.

## 4. Integration contract

The App creates an invocation with trusted identity and configuration references. Harness returns typed events, final output, artifact references, and run status. App owns presentation and product-specific lifecycle.

## 5. Design test

If a feature is only useful to one product's UI or business model, it probably belongs in the App. If it is a reusable execution primitive with stable semantics across Agents, it probably belongs in the Harness.
