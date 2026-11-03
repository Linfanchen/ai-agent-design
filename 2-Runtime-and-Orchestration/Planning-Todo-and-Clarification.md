# Planning, Todo, and Clarification

## Role in the Agent System

Planning converts a broad user goal into a manageable sequence of tasks. Todo state makes progress explicit; clarification resolves missing information before risky or expensive work; replanning adapts when observations invalidate the original plan.

## 1. Planning modes

- Implicit plan: model reasons internally and exposes concise progress only.
- Explicit task plan: structured task list visible to the user.
- Workflow plan: deterministic business process defined by code.
- Delegation plan: parent plan with child tasks and dependencies.

## 2. Plan schema

```typescript
interface PlanTask {
  id: string;
  title: string;
  description: string;
  status: "pending" | "in_progress" | "blocked" | "completed" | "failed" | "cancelled";
  dependsOn: string[];
  assignedTo?: string;
  expectedOutput?: string;
  evidenceRefs: string[];
}
```

## 3. Clarification policy

Ask when:
- the target entity is ambiguous;
- required business parameters are missing;
- the requested operation is irreversible or high impact;
- the model cannot distinguish between materially different interpretations;
- user intent or authority is unclear.

Do not ask for information already available in trusted context. Ask the single most decision-relevant question, and persist the answer as a structured fact with source and timestamp.

## 4. Plan lifecycle

Draft → Validated → Running → Replanning → Completed / Partially Completed / Blocked / Failed / Cancelled.

A plan is not a guarantee. Each task should have preconditions, output contract, deadline, retry policy, and evidence requirement.

## 5. Replanning

Replan when a dependency fails, new information changes the goal, the task exceeds its budget, a subagent returns contradictory evidence, or a human rejects an action. Preserve completed task results and do not redo successful expensive work unless stale.

## 6. User experience

Show meaningful milestones, not every internal thought. Provide the current objective, completed work, blocked reason, next action, and expected user input. Avoid exposing private reasoning traces.
