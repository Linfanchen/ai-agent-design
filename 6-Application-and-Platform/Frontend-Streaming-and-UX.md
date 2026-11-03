# Frontend Streaming and Agent UX

## Role in the Agent System

The frontend renders the Agent's structured execution events into a comprehensible experience. It is not merely a text stream: it must represent plans, progress, tool activity, approvals, clarification, artifacts, failures, and resumable runs.

## 1. UI state model

Conversation state, active run state, message state, plan state, tool progress, approval state, artifact state, and connection state should be separate typed models.

## 2. Event-to-UI mapping

| Event | UI |
|---|---|
| text delta | append to current assistant bubble |
| plan created/updated | task list / progress panel |
| tool started/progress/completed | tool activity card |
| clarification required | question/input card |
| approval required | approval card with exact target/change |
| artifact created | file/preview card |
| run interrupted | resume banner |
| run failed | error state with retry/reconcile option |
| run completed | final message and artifact list |

## 3. Streaming UX

Use sequence numbers and event IDs. Handle reconnect and replay. Do not create duplicate bubbles when the same event is replayed. Keep a loading state distinct from a running tool state.

## 4. Human control

Users need cancel, pause where supported, answer clarification, approve/reject, and inspect relevant tool results. Do not expose dangerous raw arguments without appropriate formatting or redaction.

## 5. Structured draft channel

For business form generation, separate natural-language message deltas from structured draft events. Draft data remains transient until the user explicitly adopts or submits it. This avoids confusing model-generated proposals with persisted business records.

## 6. Accessibility and usability

Keyboard navigation, screen reader labels, visible error state, clear pending/complete status, responsive layout, and safe confirmation for destructive actions.
