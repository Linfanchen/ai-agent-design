# Configuration Examples

## Role in the Agent System

This document provides a reference configuration surface for composing Agents and their runtime dependencies. Configuration is declarative; secrets are references; policy is enforced by code.

## 1. Agent Definition

```yaml
agent:
  id: enterprise-mentor-assistant
  version: 1.0.0
  display_name: Enterprise Mentor Assistant
  domain: mentorship
  graph: mentorship-supervisor
  prompt_profile: mentorship-default
  model_profile: balanced-enterprise
  context_profile: mentorship-context
  permission_profile: mentorship-read-write
  runtime_profile: standard-interactive

capabilities:
  tool_groups:
    - business-meta
    - mentorship-read
    - mentorship-write-approved
  skills:
    - mentee-progress-review
    - mentorship-plan-drafting
  plugins:
    - official-model-router

memory:
  scopes:
    - user
    - project
    - organization
  async_consolidation: true
  max_recall_tokens: 1800

governance:
  tenant_required: true
  default_deny: true
  write_approval: conditional
  audit_level: full

limits:
  max_steps: 20
  max_tool_calls: 30
  max_subagents: 4
  max_duration_seconds: 240
  max_cost_usd: 0.5
```

## 2. Model Profile

```yaml
id: balanced-enterprise
primary:
  provider: anthropic
  model: approved-model-id
fallback:
  provider: openai
  model: approved-fallback-id
routing:
  policy: capability-risk-cost
  require_structured_output: false
  require_tool_calling: true
timeouts:
  request_seconds: 90
  stream_idle_seconds: 30
```

## 3. Tool Definition

```yaml
id: mentorship.learner.status.transition
version: 1.0.0
name: transition_learner_status
operation: command
risk: medium
permission: mentorship.learner.status.update
approval: conditional
handler: java-api:teach.learner.status.transition
idempotency: required
```

## 4. Environment overlays

Use separate files or a config service for development, staging, and production. Environment overlays may change provider endpoints, log level, quotas, and sandbox resources. They must not silently loosen security policy in production.
