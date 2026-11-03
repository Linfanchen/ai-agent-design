# Engineering Checklists

## Role in the Agent System

This document is the cross-subsystem release gate. It helps reviewers verify that a change is safe, recoverable, observable, and compatible with the rest of the Agent platform.

## 1. Agent Definition
- [ ] Definition and version are explicit.
- [ ] Mission, boundaries, success criteria are defined.
- [ ] Tool, Skill, Plugin, model, and context profiles are scoped.
- [ ] Runtime budgets and timeouts are configured.

## 2. Runtime / Graph
- [ ] State schema is typed and versioned.
- [ ] Node side effects are idempotent or reconciled.
- [ ] Retry, timeout, interrupt, cancel, and resume are tested.
- [ ] Graph migrations support old checkpoints.
- [ ] Recursion/step/tool budgets prevent runaway execution.

## 3. Context / Memory / RAG
- [ ] Every source has scope, provenance, freshness, and classification.
- [ ] Live business state is fetched from the source system.
- [ ] Token budget is enforced.
- [ ] Memory has evidence, status, version, and deletion lifecycle.
- [ ] Retrieval enforces ACL before content reaches the model.

## 4. Tools / Skills / Plugins / MCP
- [ ] Tool schemas are versioned.
- [ ] Execution re-checks permissions.
- [ ] Write tools use domain APIs and idempotency.
- [ ] Skills are reviewed and allowlisted.
- [ ] Plugins declare permissions and have a disposer.
- [ ] MCP servers are trusted, authenticated, and scoped.

## 5. Security
- [ ] Tenant isolation is tested.
- [ ] Secrets are not present in prompts, traces, or checkpoints.
- [ ] Prompt injection tests exist.
- [ ] Approval binds exact action parameters and resource version.
- [ ] Sandbox escape and path traversal tests pass.

## 6. Application / Operations
- [ ] Streaming reconnect and event deduplication work.
- [ ] API errors are typed and tenant-safe.
- [ ] Health/readiness checks exist.
- [ ] Traces include model, prompt, Tool, Skill, Plugin, and graph versions.
- [ ] Golden evaluation and load tests pass.
- [ ] Rollback and incident runbooks exist.
