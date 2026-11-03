# Design Principles

## Role in the Agent System

These principles are the review rules applied across all subsystems. They prevent a system from becoming a collection of clever prompts with unclear ownership and unsafe execution paths.

## 1. Core principles

1. **Explicit state ownership** — every durable state has one authoritative owner.
2. **Model is not authority** — the model proposes decisions; policy and domain services decide what is allowed.
3. **Deterministic where correctness matters** — use code and workflows for business invariants; use LLM reasoning for ambiguity and synthesis.
4. **Capability over raw API** — expose business-oriented operations, not every controller endpoint.
5. **Least privilege by default** — every Agent, subagent, Tool, Skill, Plugin, and sandbox gets only required access.
6. **Progressive disclosure** — load Tool Schemas, Skills, Memory, and documents only when relevant.
7. **Durable execution** — checkpoint long-running work and make external side effects idempotent.
8. **Observable by construction** — every model call, context build, delegation, tool execution, and approval is traceable.
9. **Version everything that changes behavior** — Agent Definition, Prompt, Tool Schema, Skill, Plugin, Catalog, and workflow.
10. **No hidden coupling** — modules communicate through explicit contracts, not global mutable state.
11. **Failure is a first-class state** — timeout, partial success, unknown outcome, cancellation, and compensation are modeled.
12. **Human control for high-impact actions** — approval is bound to exact resource, parameters, version, and expiry.

## 2. Build vs buy vs adapt

Use a framework for orchestration primitives when it provides state, graph, persistence, interrupts, and streaming. Build your own business catalog, governance policy, approval model, tool semantics, and product experience where the domain requires control. Adapt open-source patterns; avoid cloning internal classes across languages.

## 3. Architecture review questions

- Is this a product concern, Harness concern, orchestration concern, or domain concern?
- Is this a stable fact, transient state, or derived view?
- Does the model need to see this data?
- Is this a Tool, Skill, Plugin, workflow node, or knowledge source?
- Who authorizes it?
- What happens after timeout or process restart?
- How is it tested and traced?
- How is it versioned and rolled back?
