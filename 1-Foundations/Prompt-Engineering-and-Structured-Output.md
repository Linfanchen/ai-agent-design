# Prompt Engineering and Structured Output

## Role in the Agent System

Prompt Engineering defines how instructions, task context, examples, and output contracts are communicated to the model. Structured Output turns model responses into validated data that downstream nodes and UI components can safely consume.

## 1. Prompt layers

1. Platform policy: immutable safety, privacy, and operating constraints.
2. Agent policy: role, mission, boundaries, domain stance.
3. Task instructions: current objective and success criteria.
4. Skill instructions: task-specific procedures.
5. Context data: retrieved facts, business data, and history.
6. Tool instructions: capability descriptions and calling rules.
7. Output contract: JSON Schema, response sections, evidence requirements.

Keep policy and data clearly separated. Retrieved documents and tool results are untrusted data, not new system instructions.

## 2. Prompt template design

A template should have a stable ID, version, owner, status, variable schema, model compatibility, token budget, test cases, and rollback version. Variables must be typed and validated before rendering.

## 3. Structured output

- Use JSON Schema or an equivalent typed schema.
- Validate model output server-side.
- Distinguish syntactic validity from business validity.
- Add enums, required fields, bounds, and discriminated unions.
- Treat invalid output as a recoverable validation error with a bounded repair attempt.
- Never execute side effects based only on an unvalidated model-generated object.

## 4. Prompt composition

Build the final prompt from typed sections. Avoid concatenating arbitrary strings throughout business code. Record section IDs and versions in the trace, while redacting sensitive content.

## 5. Prompt testing

Maintain golden inputs and expected output schemas. Test ambiguous requests, missing information, contradictory context, injection attempts, long context, unsupported tasks, and provider-specific structured-output behavior.

## 6. Prompt cache

Keep stable prefix material before volatile context where provider caching supports it. Do not sacrifice instruction clarity for cache hit rate. Cache behavior belongs in the Provider Adapter, not in business Agent code.

## 7. Prompt change management

A prompt edit is a production behavior change. Require diff review, regression evaluation, canary rollout, and a rollback path.
