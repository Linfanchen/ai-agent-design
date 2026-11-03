# Incident Response and Operations

## Role in the Agent System

This subsystem defines how operators diagnose and recover production Agent failures while preserving state, auditability, and user trust.

## 1. Operational views

- active runs and their status;
- stuck/expired jobs and queue depth;
- provider health, 429/5xx rate, and latency;
- tool failure and unknown-outcome counts;
- pending approvals;
- memory/indexing lag;
- sandbox capacity;
- token/cost per tenant;
- trace and evaluation regression.

## 2. Common incidents

Provider outage, runaway Agent loop, tool schema drift, MCP server disconnect, sandbox exhaustion, cross-tenant data exposure, stuck approval, checkpoint incompatibility, queue backlog, and artifact storage failure.

## 3. Runbook structure

Symptom → impact → detection → immediate containment → diagnosis → recovery → data reconciliation → customer communication → postmortem.

## 4. Kill switches

Provide tenant-level and platform-level controls to disable a model, Tool, Plugin, Skill, MCP server, Agent Definition, channel, or write operation. Kill switches must be auditable and should not require a full application redeploy.

## 5. Postmortem

Record timeline, root cause, contributing factors, customer impact, data integrity status, security impact, recovery steps, and preventive actions. Convert recurring incidents into regression tests.
