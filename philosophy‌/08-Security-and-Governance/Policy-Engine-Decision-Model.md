# Policy Engine：决策模型与执行点

## 本文档在系统中的作用

Policy Engine 将“这个用户能不能做这个动作”从 Prompt 和业务代码中抽离，形成统一、可测试、可审计的授权决策系统。

## 1. Policy Input

principal、tenant、action、resource、environment、agent_release、tool_version、data_classification、approval_state、delegation_scope。

## 2. Policy Decision

`ALLOW`、`DENY`、`REQUIRE_APPROVAL`、`CONDITIONAL_ALLOW`。Decision 同时返回 policy_id、policy_version、reason_code、obligations、expires_at。

不要把内部敏感规则原文返回给模型；只返回可安全解释的 reason code。

## 3. Policy Evaluation 顺序

1. 身份有效；
2. Tenant 匹配；
3. Agent Release 允许；
4. Tool Binding 允许；
5. Resource ACL；
6. Action Permission；
7. 数据分类；
8. 环境限制；
9. 风险级别；
10. Approval Requirement；
11. Rate / Quota。

## 4. 条件授权

例如允许读取某项目，但只能返回姓名、状态和阶段，不能返回薪资、联系方式。Policy 不仅控制能否调用，还可定义字段级 Masking / Projection。

## 5. Policy Snapshot

Run 保存策略版本；Tool Invoke 时检查当前撤销规则。对于安全关键策略，应允许全局 Revocation Overlay 覆盖旧 Snapshot。

## 6. 决策缓存

缓存键至少包含 Tenant、Principal、Action、Resource、Policy Version、Resource Version 和必要环境属性。权限变化必须使缓存失效。写操作不应只依赖过期较长的授权缓存。

## 7. Policy Test

每条 Policy 都有 Allow Case、Deny Case、Boundary Case、Missing Attribute Case、Expired Grant Case 和 Tenant Isolation Case。

## 8. Policy Rule 的优先级

建议规则分层：
1. Platform Deny：平台硬性禁止；
2. Tenant Policy：租户组织规则；
3. Agent Release Policy：该 Agent 可用范围；
4. Principal Role / Attribute；
5. Resource ACL；
6. Action Risk / Approval；
7. Environment / Quota。

Deny Override Allow。策略冲突时按明确优先级处理，不能依赖规则加载顺序偶然决定。

## 9. Policy Explainability

对模型返回安全摘要，例如“当前用户无权修改该项目”或“该动作需要管理员审批”。不要返回内部策略表达式、其他用户信息或规则细节。对管理员提供 Policy Trace，记录命中规则和条件评估结果。

## 10. Policy Simulation

发布前提供 Dry Run：输入 Principal、Resource、Action，输出 Decision 和命中规则。使用生产脱敏样本模拟策略变更影响，识别哪些原本允许的动作将被拒绝、哪些动作可能被错误放行。

## 11. Policy Drift

Agent Release 引用的 Policy Version 可能落后于全局撤销策略。系统需要区分可继续使用的历史 Snapshot 和必须立即生效的 Revocation Overlay。
