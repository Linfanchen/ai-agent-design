# Agent 平台端到端时序图

## 本文档在系统中的作用

用时序图描述 Agent Run、Tool Call、Approval / Resume、Memory Consolidation 和 Evaluation 的跨模块协作，避免各子系统只理解自身而不理解系统级调用关系。

## 1. Run 创建与 Worker 执行

```mermaid
sequenceDiagram
    participant U as User
    participant API as FastAPI
    participant APP as Application Service
    participant DB as PostgreSQL
    participant W as Worker
    participant H as Harness Factory
    participant G as LangGraph
    participant EV as Event Store

    U->>API: POST /runs + Idempotency-Key
    API->>APP: CreateRun(Principal, Input)
    APP->>DB: Transaction: Run + Job + Outbox
    DB-->>APP: Commit
    APP-->>API: Run ID
    API-->>U: 202 Accepted
    W->>DB: Claim Job + Lease
    W->>H: Build Runtime from Release
    H-->>W: AgentRuntime
    W->>G: ainvoke / astream
    G->>EV: Append Run Events
    G-->>W: Final / Interrupt
    W->>DB: Update Run / Job
    W-->>U: Events delivered via SSE
```

## 2. Tool Search / Describe / Invoke

```mermaid
sequenceDiagram
    participant LLM as Model
    participant D as Discovery
    participant R as Registry
    participant P as Policy
    participant X as Executor
    participant H as Handler
    participant DB as Execution Store

    LLM->>D: Search(intent, scope)
    D->>R: Candidate Search
    R-->>D: Filtered Metadata
    D-->>LLM: Top-K Summaries
    LLM->>D: Describe(tool_ref)
    D->>R: Load Schema + Version
    R-->>LLM: Full Contract
    LLM->>X: Tool Call(arguments)
    X->>R: Resolve Version + Binding
    X->>P: Reauthorize Principal / Resource / Action
    P-->>X: Allow / Deny / Approval
    X->>DB: Create Idempotent Execution Intent
    X->>H: Execute(args, ToolContext)
    H-->>X: Result / Unknown
    X->>DB: Persist Outcome + Audit
    X-->>LLM: Structured Tool Result
```

## 3. Approval / Interrupt / Resume

```mermaid
sequenceDiagram
    participant G as Graph
    participant X as Tool Executor
    participant A as Approval Service
    participant DB as Checkpointer
    participant U as Approver
    participant API as Resume API
    participant W as Worker

    G->>X: Propose High-Risk Action
    X->>A: Create Approval(intent_hash)
    A-->>X: approval_id
    X-->>G: Require Approval
    G->>DB: Save Checkpoint + Interrupt
    G-->>U: Approval Required Event
    U->>A: Approve / Reject
    A->>A: Validate Approver + Intent
    U->>API: Resume(interrupt_id, response)
    API->>A: Verify Approval State
    API->>W: Enqueue Resume Job
    W->>DB: Load Checkpoint
    W->>G: Resume Graph
    G->>X: Reauthorize + Compare Intent Hash
    X-->>G: Execute / Reject
```

## 4. Memory Consolidation

```mermaid
sequenceDiagram
    participant RUN as Run
    participant O as Outbox
    participant M as Memory Worker
    participant E as Extractor
    participant V as Validator
    participant S as Memory Store
    participant I as Vector Index

    RUN->>O: Run Completed Event
    O-->>M: Deliver Event
    M->>E: Extract Candidates from Approved Sources
    E-->>M: Candidate Facts + Evidence
    M->>V: Validate Scope / Sensitivity / Conflict
    V->>S: Create Candidate / New Version
    S-->>M: Memory ID
    M->>I: Upsert Authorized Embedding
    I-->>M: Index Result
    M->>S: Mark Active / Failed
```

## 5. Evaluation Release Gate

```mermaid
sequenceDiagram
    participant C as Control Plane
    participant E as Evaluation Runner
    participant D as Dataset
    participant R as Runtime
    participant J as Evaluators
    participant G as Release Gate

    C->>E: Evaluate Candidate Release
    E->>D: Load Fixed Dataset Version
    loop Each Case
        E->>R: Run in Isolated Fixture
        R-->>E: Result + Trajectory
        E->>J: Deterministic + Judge Evaluation
        J-->>E: Scores + Evidence
    end
    E->>G: Aggregate Metrics + Baseline Delta
    G-->>C: Pass / Block / Review
    C->>C: Publish only if Gate Passed
```

## 6. 阅读时重点检查

- Run 创建与执行分离；
- Tool Search 与 Invoke 的授权边界；
- Approval Service 与 Graph Interrupt 分离；
- Memory Worker 异步化；
- Evaluation 与生产身份隔离；
- 每个跨模块边界都有持久化 ID 和 correlation_id。
