# Configuration and Secrets Management

## Role in the Agent System

Configuration determines how the Harness is assembled and how each Agent behaves across environments. Secret Management supplies credentials without exposing them to prompts, logs, checkpoints, or user-editable configuration.

## 1. Configuration layers

1. Platform defaults.
2. Environment configuration.
3. Tenant configuration.
4. Agent Definition.
5. User/session overrides where permitted.
6. Runtime request parameters.

Define precedence explicitly. High-risk security settings should not be overridable by a user or model.

## 2. Configuration domains

Models, model routing, tools, tool groups, tool search, MCP servers, skills, plugins, memory, context budgets, summarization, sandbox, subagents, channels, checkpointer/store, guardrails, quotas, tracing, and feature flags.

DeerFlow's configuration surface is a useful example: model/tool/sandbox/skill/subagent/memory/summarization/checkpointer/guardrail/channel settings are separate top-level domains.

## 3. Schema and validation

Use typed schemas with defaults, bounds, enums, secret references, version, and migration rules. Validate on save and again on runtime assembly.

## 4. Secrets

Store provider API keys, MCP OAuth tokens, connector credentials, and signing keys in a Secret Manager or encrypted credential vault. Config stores a reference, not the secret value. Inject only into the component that needs it.

## 5. Dynamic reload

Support reloadable configuration only for components that can safely update in place. For changes to graph schema, plugin code, or tool contracts, use versioned rollout or restart. Record configuration snapshot/version per run.

## 6. Environment strategy

Keep development, staging, and production isolated. Never copy production secrets into local config files. Use environment variables only as a transport into the process, not as the long-term secret-management architecture.
