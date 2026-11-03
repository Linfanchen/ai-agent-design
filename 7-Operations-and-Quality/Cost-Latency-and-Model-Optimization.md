# Cost, Latency, and Model Optimization

## Role in the Agent System

This subsystem optimizes the total cost and responsiveness of completed tasks without compromising correctness, safety, or evidence quality.

## 1. Cost model

Track input/output tokens, cached tokens, embedding calls, reranking, image/audio/video processing, tool usage, sandbox runtime, storage, and external service charges. Attribute cost to tenant, Agent, task, model, and Tool.

## 2. Latency model

Measure time to first token, model latency, tool latency, retrieval latency, queue wait, approval wait, subagent critical path, and end-to-end completion.

## 3. Optimization levers

- choose a smaller model for classification and extraction;
- reserve stronger reasoning models for complex planning;
- defer Tool Schema loading;
- retrieve fewer but higher-quality context sources;
- summarize oversized tool results;
- cache stable context and retrieval;
- parallelize independent read-only tools;
- use background jobs for long tasks;
- avoid unnecessary model round trips;
- use structured deterministic code for simple transforms.

## 4. Model routing

Route by required capability, task risk, context size, quality target, latency SLO, data policy, and current provider health. Do not route solely by cheapest price.

## 5. Budget enforcement

Enforce per-turn and per-task budgets for tokens, provider spend, tool calls, wall time, subagents, and sandbox resources. When a budget is reached, summarize partial progress and ask whether to continue if the task is still useful.

## 6. Optimization safety

A faster or cheaper route must pass the same golden evaluation and policy suite. Monitor quality drift after model changes.
