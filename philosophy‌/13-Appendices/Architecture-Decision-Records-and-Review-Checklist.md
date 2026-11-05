# ADR 模板与架构评审清单

## 本文档在系统中的作用

ADR 记录为什么选择某种技术和边界；评审清单确保模块变更不会破坏租户隔离、恢复、幂等和可观测性。

## 1. ADR 模板

```markdown
# ADR-NNN：标题
- 状态：Proposed / Accepted / Deprecated / Superseded
- 日期：
- Owner：

## 背景与约束
## 决策
## 替代方案
## 正向影响
## 负向影响
## 安全与隐私影响
## 迁移计划
## 验证方式
## 复审条件
```

## 2. 架构评审清单

### Runtime
- [ ] Factory 是否集中装配？
- [ ] Process / Run / Step 资源生命周期是否清晰？
- [ ] 是否有取消和超时策略？
- [ ] Worker 重启是否可恢复？

### LangGraph
- [ ] State 是否可序列化？
- [ ] Runtime Context 是否与 State 分离？
- [ ] Reducer 是否确定性？
- [ ] Checkpoint / State Version 是否兼容？
- [ ] Node 副作用是否幂等？

### Context / Memory
- [ ] 来源、信任和权限是否明确？
- [ ] Token Budget 是否可解释？
- [ ] 记忆是否有证据、Scope、版本和过期？
- [ ] 删除是否覆盖索引和缓存？

### Tool
- [ ] Schema 是否严格？
- [ ] Invoke 是否重新鉴权？
- [ ] 是否有 Idempotency Key？
- [ ] Timeout 是否可能变 UNKNOWN？
- [ ] 高风险操作是否审批？
- [ ] 结果是否验证？

### Security
- [ ] Tenant 是否从可信 Principal 解析？
- [ ] 是否有默认拒绝？
- [ ] 是否防止 Prompt Injection 越权？
- [ ] 插件 / MCP / Sandbox 是否隔离？

### Evaluation
- [ ] 是否有 Golden Dataset？
- [ ] 是否测工具选择和参数？
- [ ] 是否测恢复和权限？
- [ ] 是否有发布门禁？

## 3. 发布阻断项

任何跨租户数据泄漏、未授权副作用、审批绕过、密钥暴露、任意代码执行、重复不可逆操作都应阻断发布。

## 4. ADR 评审规则

- 每项影响数据所有权、权限、持久化、部署或外部协议的决策必须有 ADR；
- ADR 必须列出至少两个替代方案；
- 决策需记录短期和长期代价；
- 影响 Checkpoint / Tool Contract 的变更需有迁移计划；
- 已接受 ADR 被替代时保留历史并标记 Superseded。

## 5. 架构评审会议的必答问题

1. 是否改变了数据事实来源？
2. 是否新增跨租户或跨进程边界？
3. 是否有新的外部副作用？
4. 是否能安全重试？
5. 是否有明确取消语义？
6. 是否需要审批？
7. 是否会增加 Token / 成本？
8. 是否影响历史 Checkpoint？
9. 是否需要回填 Catalog / Memory / Vector Index？
10. 如何回滚？

## 6. 发布验收证据

每次发布保存 CI 结果、Migration Plan、Contract Test、Security Test、Evaluation Report、Canary 指标和回滚版本。没有证据的“已测试”不作为发布依据。
