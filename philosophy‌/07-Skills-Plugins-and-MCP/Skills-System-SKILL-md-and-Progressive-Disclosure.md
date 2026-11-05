# Skills：SKILL.md、渐进式加载与执行边界

## 本文档在系统中的作用

Skill 是可复用任务知识包，描述某类任务应该如何完成、使用哪些能力、遵守哪些规则。它不是 Tool，也不天然拥有执行权限。Skill 主要改变 Agent 的工作方法；Tool 才执行具体动作。

## 1. Skill 与 Tool 的区别

| 维度 | Skill | Tool |
|---|---|---|
| 本质 | 指令、流程、领域知识 | 可调用能力 |
| 触发 | 任务相关时加载 | 模型发起 Tool Call |
| 执行 | 指导 Agent 规划 | 由 Executor 执行 |
| 权限 | 不授予权限 | 需要独立授权 |
| 内容 | Markdown、资源、脚本引用 | Schema + Handler |
| 风险 | 指令注入、恶意脚本 | 副作用、数据访问 |

Skill 文档中写“可以删除数据”不代表 Agent 获得删除权限。

## 2. SKILL.md 推荐结构

```yaml
---
name: mentorship-report
description: 生成导师带教阶段报告，适用于项目总结和学员阶段评估
version: 1.2.0
tags: [mentorship, reporting]
allowed-tools:
  - mentorship.project.read
  - mentorship.learner.read
  - report.draft.create
---
```

正文包含 Purpose、When to Use、Required Inputs、Workflow、Decision Rules、Tool Usage Guidance、Output Format、Validation、Failure Handling、Examples、References。

## 3. 渐进式加载

Level 0：只加载 Skill Name + Description，供路由使用。
Level 1：选中 Skill 后加载 SKILL.md 主体。
Level 2：需要时再加载 references、examples、templates。
Level 3：脚本只在受限 Sandbox 中执行，并由 Tool Executor 授权。

不要把所有 Skill 全文放进每次 System Prompt。

## 4. Skill Manifest

保存 name、version、description、entry_file、resource_files、required_capability_tags、trust_level、checksum、author、license、scan_status、published_status。

## 5. 解析与验证

YAML Front Matter 严格解析；禁止重复 Skill Name；校验路径只能位于包根目录；禁止路径穿越；校验引用文件存在；校验依赖 Tool 可用；检查脚本类型、权限和网络需求；限制文件大小、递归深度和加载 Token。

## 6. Skill 安全

Skill 内容属于指令数据，不得覆盖平台 Policy。第三方 Skill 需要来源、许可证、checksum 和安全扫描。脚本默认不执行；如需执行，必须使用 Sandbox，禁止在 Agent 主进程中 `exec()` 或 `eval()`。

## 7. 版本与测试

Skill Version 不可变。更新时跑任务级 Golden Dataset，比较 Tool Selection、任务成功率、步骤数、成本和安全拒绝。Skill 退役后旧 Agent Release 仍需有明确兼容策略。

## 8. Skill 与 Agent Prompt 的关系

Agent Prompt 定义长期角色和平台约束；Skill 定义任务类型下的执行方法。Skill 不应复制全部平台规则，也不应包含秘密。加载 Skill 时保留 Skill Version 和 Source Ref，方便评测回归。

## 9. Skill 匹配机制

候选 Skill 可按名称、Tags、Description、Task Type 和 Domain 检索。先做 Agent Release Binding / Tenant Enablement 过滤，再做关键词与语义排序。若多个 Skill 高度相似，应返回少量候选并允许模型选择或向用户澄清。

## 10. Skill 冲突

两个 Skill 可能给出不同步骤。解决顺序：
1. 平台 Policy 优先；
2. Agent Release 配置的 Skill Priority；
3. 任务专用 Skill 优先于通用 Skill；
4. 版本和兼容性检查；
5. 无法消解时显式报告冲突。

Skill 不得通过文字覆盖 Tool Policy 或用户审批规则。

## 11. Skill 测试

每个 Skill 维护任务样本，检查是否正确触发、是否加载正确资源、是否调用预期 Tool、是否违反限制、成本是否合理。Skill 更新必须进入 Agent Release 回归评测。
