# 14. Session、Event Log 与 Checkpoint

## Role in the Agent System

Separates conversation history, durable events, graph state snapshots, and execution recovery.

**Primary responsibility:** Own this capability as a reusable subsystem with explicit contracts, state ownership, security boundaries, and observable outcomes.

**Boundary:** This subsystem must not silently take over responsibilities owned by adjacent layers. Its public interface should be testable without running a full Agent conversation.

## 1. 三者职责

| 组件 | 作用 | 典型存储 |
|---|---|---|
| Session | 对话和执行归属 | session 表 |
| Event Log | 发生过的持久事实 | append-only event 表 |
| Checkpoint | 图当前状态快照 | LangGraph checkpointer |

Event Log 适合审计和重建历史；Checkpoint 适合快速恢复执行。二者不应相互替代。

## 2. Session 层级

Conversation：用户可见的会话容器。Session：某 Agent 在某 Scope 下的执行历史。Turn：一次用户输入。Step：一次模型请求周期。Tool Execution：工具执行实例。Child Session：Subagent 子任务。

## 3. 事件分类

持久事件：
- `turn.started`
- `user.message`
- `assistant.message`
- `assistant.tool_call`
- `tool.execution.started`
- `tool.execution.completed`
- `tool.execution.failed`
- `approval.requested`
- `approval.resolved`
- `turn.completed`
- `turn.failed`
- `session.compacted`

瞬态事件：token chunk、typing indicator、heartbeat、progress pulse、transient UI state。

瞬态事件可通过 WebSocket/SSE 推送，但不应无差别持久化成大量事件。

## 4. Event Schema

```typescript
interface AgentEvent<T = unknown> {
  eventId: string;
  tenantId: string;
  sessionId: string;
  turnId?: string;
  stepId?: string;
  sequence: number;
  eventType: string;
  schemaVersion: number;
  occurredAt: string;
  actorType: "user" | "agent" | "tool" | "system";
  actorId: string;
  correlationId: string;
  causationId?: string;
  payload: T;
}
```

## 5. Checkpoint

Checkpoint 至少保存 graph state、current node、pending interrupt、task plan、discovered tool references、completed tool execution references、context snapshot reference、agent/model/version metadata。

不要把大文件、超长 Tool Result 或密钥放进 Checkpoint。大对象保存在 Object Storage / DB，State 保存引用。

## 6. 恢复与重复执行

“恰好一次执行”不能仅靠 Checkpoint 保证。网络超时可能出现服务端已完成操作但 Agent 未收到结果。

写操作必须：
- 幂等键
- 业务命令 ID
- 业务记录版本 / 乐观锁
- Tool Execution 状态机
- 查询操作结果的恢复接口

推荐状态：
`CREATED → AUTHORIZED → APPROVED → EXECUTING → SUCCEEDED / FAILED / UNKNOWN`

UNKNOWN 表示调用超时但无法确认业务是否提交，必须先查询业务命令状态，不能直接重放。

## 7. 事件重放

Replay 用于调试、审计和测试。重放时默认不重新执行外部写工具；使用已保存 Tool Result；需要重新执行时必须显式选择 sandbox / dry-run；保留当时 Tool Schema、Skill、Prompt、Model、Catalog 版本。

## 8. 并发控制

同一 Session 同时只允许一个写入 Turn，或使用明确并发策略。用户 steer / follow-up 要定义队列语义。子 Session 独立执行，但共享业务资源的写操作由业务层锁/版本控制。

## 9. LangGraph 集成

`thread_id` 应稳定关联 Session 或执行线程。Checkpointer 负责图状态持久化；应用仍需维护自己的 Session/Event/Tool Execution/Audit 业务表。Human Interrupt 要保存 pending operation，审批完成后通过 resume 恢复。
