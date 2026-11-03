# 12. Skills Engineering

## Role in the Agent System

Defines reusable procedural knowledge packages and their discovery, loading, security, and release lifecycle.

**Primary responsibility:** Own this capability as a reusable subsystem with explicit contracts, state ownership, security boundaries, and observable outcomes.

**Boundary:** This subsystem must not silently take over responsibilities owned by adjacent layers. Its public interface should be testable without running a full Agent conversation.

## 1. Skill 定义

Skill 是可按需加载的程序性知识包，告诉 Agent 在特定任务中如何工作。它不是 Tool，也不是任意可执行插件。

Skill 负责触发条件、任务步骤、领域规则、检查清单、异常处理、输出要求、可选参考文档和脚本。Tool 负责实际操作。

## 2. Progressive Disclosure

| 层级 | 加载内容 | 目标 |
|---|---|---|
| L0 | name、description、tags | 发现候选 |
| L1 | SKILL.md 主体 | 任务执行方法 |
| L2 | references / examples / schemas | 深入细节 |
| L3 | scripts / assets / templates | 受控执行资源 |

不要在每次 Prompt 中注入所有 Skill 正文。

## 3. Skill 目录规范

```text
skills/
  mentorship/
    mentee-progress-review/
      SKILL.md
      references/
        progress-rules.md
        exception-cases.md
      examples/
        expected-output.json
      scripts/
        validate-report.ts
```

示例 frontmatter：

```yaml
---
name: mentee-progress-review
description: Review a learner's mentorship progress and identify overdue work.
version: 1.0.0
domain: mentorship
owner: hr-operations
risk: read-only
allowed-tools:
  - search_mentees
  - get_mentee_progress
tags:
  - learner
  - progress
---
```

## 4. Skill 生命周期

Draft → Validate → Security Scan → Test → Review → Publish → Deprecate / Rollback。

生产 Skill 必须有 owner、版本、适用范围、允许工具、评估用例、来源和许可证、变更记录。

## 5. Skill Trust

区分 Platform-owned、Organization-approved、Project-local、User-private、External-untrusted。

外部 Skill 不得直接获得任意文件、网络、终端和业务写权限。脚本要独立审核并在 Sandbox 中运行。

## 6. Skill 与 Workflow

Skill 可以描述步骤，但对确定性、高风险业务流程，执行顺序和状态迁移应由 Workflow / Java Service 控制。模型不能通过忽略 Skill 文本绕过业务规则。

## 7. Skill Learning

从成功轨迹提取候选 Skill：
1. 识别重复流程。
2. 总结目标、前置条件和步骤。
3. 提取工具依赖。
4. 提取失败案例。
5. 生成 Skill Candidate。
6. 生成回归测试。
7. 人工审核。
8. 发布新版本。

企业业务规则和安全策略必须人工审核；用户个人工作偏好可以采用更轻的确认机制。

## 8. Skill Bundles

Bundle 可以把一项复杂工作所需的多个 Skill 组合起来，例如 `quarterly-mentorship-review`：
- project-analysis
- mentee-progress-review
- overdue-identification
- report-generation

Bundle 只是组合和入口，不应把多个 Skill 复制粘贴成一个巨大 Prompt。

## 9. 与 Agent Skills 标准

建议遵循 Agent Skills 的目录和 frontmatter 约定，便于与其他 Agent 生态互操作。MCP Skills Extension 可作为远程发布/读取 Skill 的传输绑定，但需按协议版本和成熟度单独评估，不应成为核心 Skill 存储的唯一方式。
