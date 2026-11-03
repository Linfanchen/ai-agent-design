# Workspace and Artifact Management

## Role in the Agent System

Workspace is the Agent's scoped working area for files and intermediate resources. Artifact Management turns useful outputs into durable, user-visible deliverables with ownership, version, preview, and access controls.

## 1. Workspace vs Artifact vs Business Record

- Workspace file: working material, temporary notes, code, downloaded source.
- Artifact: final or intermediate deliverable intended to be presented to a user.
- Business record: authoritative entity stored in the enterprise application.

A generated spreadsheet is an artifact; an approved expense record is a business record.

## 2. Workspace scope

Support tenant, user, project, thread, and task scopes. Every path must be normalized and resolved within the authorized root. Never trust model-generated absolute paths.

## 3. Backend abstraction

Use a Workspace Backend interface for local development, object storage, and isolated sandbox mounts. Store metadata in PostgreSQL and binary objects in object storage.

## 4. Artifact lifecycle

Created → Validated → Scanned → Published → Versioned → Archived / Deleted.

Metadata: artifact ID, tenant, owner, thread, task, type, filename, MIME type, size, checksum, object URI, preview URI, sensitivity, status, version, created time, retention policy.

## 5. Presentation tools

A `present_artifact` tool should register a file and return a structured artifact reference. Do not ask the model to invent download URLs. The application generates authorized links.

## 6. Security

Scan uploads and generated archives, enforce file type/size limits, prevent path traversal, sanitize filenames, use short-lived signed URLs, and record downloads. Do not expose workspace mounts directly to a browser or MCP server without an explicit path and permission contract.
