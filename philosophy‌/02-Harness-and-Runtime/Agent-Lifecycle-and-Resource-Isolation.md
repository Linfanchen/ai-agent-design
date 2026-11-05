# Agent 生命周期与资源隔离

## 本文档在系统中的作用

定义 Agent、Runtime、Run、Step、Tool Execution 和资源的生命周期，解决创建、复用、挂起、恢复、取消和销毁边界不清的问题。

## 1. Definition 与 Runtime 不是同一对象

Agent Definition 是持久化配置；Agent Release 是不可变配置快照；Runtime 是执行容器；Run 是一次任务；Thread 是对话 / 执行连续性边界。它们的生命周期不同。

## 2. Run 状态机

```text
queued → starting → running
                    ├→ waiting_input
                    ├→ waiting_approval
                    ├→ succeeded
                    ├→ partially_succeeded
                    ├→ failed
                    ├→ cancel_requested → cancelled
                    └→ expired
```

`waiting_approval` 不是失败；用户等待期间 Worker 不应一直占用执行槽。恢复时创建新的 Worker Attempt，但保持同一 Run 的逻辑身份。

## 3. 资源隔离

- Process：连接池和只读配置。
- Tenant：配额、密钥引用、策略、命名空间。
- Run：Trace、预算、取消令牌、临时工作区。
- Step：DB Session、HTTP Response、临时对象。
- Sandbox：独立容器、CPU / Memory / PID / Network 限制。

## 4. 取消语义

区分 User Cancel、Deadline Exceeded、Worker Shutdown、Parent Cancel、Security Revocation。取消时先停止新 Tool Dispatch，再向当前协程传播 Cancel，最后检查是否存在已提交的外部副作用需要对账。

## 5. 挂起与恢复

挂起时保存 Graph Checkpoint、Run / Job 状态和 Artifact 引用，释放 Worker Slot 和短期资源。恢复时重新验证 Principal、Tool、Policy 和预算，重新装配 Runtime，不复用旧 DB Session / HTTP Response / Container Handle。

## 6. 资源清理保障

每个资源创建必须能找到唯一 Owner 和释放点。临时资源有 TTL Sweeper，避免进程崩溃后遗留文件、容器和租约。Sandbox 使用 Lease + Heartbeat + Reaper。

## 7. 验收测试

对每个状态执行进程重启测试；对每种取消来源执行清理测试；对挂起 24 小时后恢复测试；对 Release 被撤销、权限被收回、Tool 版本被禁用的恢复场景做验证。
