# Evaluation and Testing

## Role in the Agent System

Evaluation determines whether an Agent reliably completes intended tasks without violating policies. Testing validates deterministic components; evaluation measures probabilistic behavior across models, prompts, tools, and real-world task distributions.

## 1. Test pyramid

- Unit tests: schemas, reducers, policy rules, parsers, adapters.
- Component tests: Context Builder, Memory Recall, Tool Discovery, Skill Loader.
- Graph tests: node behavior, conditional routes, interrupts, state transitions.
- Integration tests: model/provider, database, MCP, sandbox, Java business APIs.
- End-to-end tests: full user task.
- Regression tests: replay saved trajectories with side effects mocked.
- Adversarial tests: injection, ambiguous identity, cross-tenant access, malicious tools.
- Load tests: concurrent threads, long-running tasks, streaming reconnect.

## 2. Evaluation dataset

Each case should define user input, tenant/user context, expected intent, required facts, allowed tools, expected tool arguments, prohibited actions, expected output schema, evidence requirements, and scoring rubric.

## 3. Metrics

Task success, groundedness, factual correctness, tool selection accuracy, argument validity, workflow compliance, clarification precision, policy violation rate, hallucinated completion rate, cost, latency, and human acceptance.

## 4. LLM-as-judge

Use judges for scalable qualitative scoring, but calibrate against human labels. Keep judge prompt/model/version fixed. For critical decisions, require deterministic checks or human review.

## 5. Test doubles

Mock provider, fake tool executor, deterministic clock, in-memory checkpointer, synthetic tenant, fake approval service, and sandbox fixture. Never call production write APIs from unit tests.

## 6. Release gates

Run golden set, security suite, cost/latency regression, provider matrix, migration tests, and canary evaluation before publishing Agent Definition, Skill, Tool, Plugin, or Prompt changes.
