# Data Privacy and Safety

## Role in the Agent System

This subsystem protects user data, business information, model inputs/outputs, and generated artifacts. It defines which data may enter Context, which data may leave the system, and how harmful or unauthorized actions are blocked.

## 1. Data classification

Define PUBLIC, INTERNAL, CONFIDENTIAL, and RESTRICTED classes. Classification must be attached to sources and fields, not guessed from prompt text.

## 2. Data flow controls

- Ingress: validate, scan, classify, and minimize.
- Context: filter by identity, tenant, task, and classification.
- Provider egress: apply vendor/region policy and redact unnecessary fields.
- Tool output: sanitize and minimize before model injection.
- Storage: encrypt, scope, and set retention.
- Export: require authorized download and audit.

## 3. Prompt injection defense

Treat user input, web pages, uploaded files, MCP results, Skill text, and retrieved documents as untrusted. Keep instructions separate from evidence. Use allowlisted capabilities, server-side policy checks, schema validation, and approval for consequential actions.

## 4. Safety layers

Input classification → policy check → model refusal handling → tool permission → output validation → audit. No single model classifier should be the only control for critical operations.

## 5. Privacy by design

Minimize personal data, avoid retaining secrets, support data subject requests where applicable, define retention/deletion behavior, and ensure logs and traces do not become a shadow copy of the entire application database.

## 6. Provider governance

Track provider, region, data processing terms, retention settings, training-use terms, and model capability. Route sensitive workloads only to approved providers and configurations.
