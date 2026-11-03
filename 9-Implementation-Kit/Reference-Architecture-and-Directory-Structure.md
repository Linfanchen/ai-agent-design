# Reference Architecture and Directory Structure

## Role in the Agent System

This document translates subsystem boundaries into a maintainable TypeScript repository layout. It is the bridge from architecture to code ownership.

## 1. Recommended monorepo layout

```text
agent-platform/
  apps/
    agent-api/
    admin-web/
    chat-web/
  workers/
    agent-worker/
    memory-worker/
    indexing-worker/
    sandbox-worker/
  packages/
    contracts/
    core/
    runtime/
    orchestration/
    model-providers/
    prompt-engine/
    context-engine/
    memory-engine/
    rag-engine/
    business-catalog/
    tool-registry/
    tool-executor/
    skill-engine/
    plugin-sdk/
    plugin-host/
    mcp-adapter/
    agent-protocols/
    workspace/
    governance/
    event-bus/
    observability/
    evaluation/
    persistence/
    config/
    testkit/
  skills/
    platform/
    organization/
    examples/
  plugins/
    first-party/
  graphs/
    general-assistant/
    business-assistant/
  infra/
    docker/
    k8s/
    migrations/
    dashboards/
  docs/
    architecture/
    runbooks/
```

## 2. Package ownership

- `contracts`: shared DTOs, event schemas, IDs, error codes.
- `runtime`: lifecycle, runtime factory, invoke/stream/resume/cancel.
- `orchestration`: graph builders, nodes, reducers, subgraphs.
- `model-providers`: vendor adapters and capability descriptors.
- `context-engine`: context source retrieval, ranking, token budget.
- `memory-engine`: memory candidate, promotion, recall, retention.
- `tool-registry`: metadata and discovery.
- `tool-executor`: execution, policy gate, idempotency, result normalization.
- `governance`: identity, tenant, policy, approval, audit.
- `business-catalog`: code/API/DDL scanning and capability mapping.
- `observability`: trace, event, trajectory, metrics.
- `evaluation`: golden datasets, graders, regression suite.

## 3. Dependency direction

`apps → packages/contracts + runtime facade`

`runtime → orchestration + engines + governance contracts`

`tool-executor → business adapters / MCP adapters / sandbox adapters`

`business adapters → Java domain APIs`

No package should import internal files from another package. Use public exports and contract tests.

## 4. Modular monolith first

Do not deploy every package as a microservice. Start with API + runtime worker + background worker + sandbox boundary. Split only when scaling, failure isolation, security, or team ownership requires it.
