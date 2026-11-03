# Agent Definition and Identity

## Role in the Agent System

Agent Definition is the versioned blueprint that tells the Harness what an Agent is allowed and expected to do. It separates a reusable specialist configuration from the transient Runtime Instance executing a request.

## 1. Definition components

- Identity: stable ID, name, description, owner, domain, version.
- Mission: goals, non-goals, user types, success criteria.
- Behavior: system policy, tone, clarification rules, response format.
- Model profile: provider, model, temperature, reasoning level, token budget.
- Capability profile: tool groups, individual tools, skills, plugins.
- Context profile: memory scopes, RAG collections, business domains, history policy.
- Governance profile: permissions, approval policy, data classification.
- Runtime profile: graph ID, max steps, timeouts, concurrency, retry budget.
- Product profile: channel availability, UI presentation, artifact support.

## 2. Definition vs instance

Definition is immutable per published version. Instance binds the definition to tenant/user/session and current execution state. Updating an Agent Definition must not silently mutate an already-running execution unless an explicit live-update policy permits it.

## 3. Example schema

```typescript
interface AgentDefinition {
  id: string;
  version: string;
  displayName: string;
  description: string;
  domain: string;
  ownerId: string;
  systemPolicyRef: string;
  promptProfileId: string;
  modelProfileId: string;
  graphId: string;
  toolGroupIds: string[];
  skillIds: string[];
  pluginIds: string[];
  contextProfileId: string;
  permissionProfileId: string;
  runtimeProfileId: string;
  status: "draft" | "published" | "deprecated";
}
```

## 4. Agent personas and boundaries

Persona describes communication style and domain stance; it is not a security boundary. "You are an HR assistant" does not authorize HR data access. Permissions must be enforced independently.

## 5. Agent lifecycle

Draft → Validate → Test → Review → Publish → Deploy → Monitor → Deprecate / Rollback.

## 6. Design recommendations

- Keep stable Agent IDs independent of display names.
- Version all behavior-changing configuration.
- Support environment-specific overrides without changing the definition.
- Define default-deny tool and skill access.
- Separate system policy from editable persona text.
- Record which definition version each run used.
- Support cloning and inheritance only with explicit override semantics.
