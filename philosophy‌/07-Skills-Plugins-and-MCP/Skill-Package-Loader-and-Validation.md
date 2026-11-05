# Skill Package Loader：解析、资源加载与安全校验

## 本文档在系统中的作用

本文件描述 Skill 从包发现到运行时渐进加载的完整流程，解决 Front Matter 解析、路径安全、资源引用、依赖检查、版本固定和恶意 Skill 内容处理。

## 1. 包结构

```text
skills/
└── mentorship-report/
    ├── SKILL.md
    ├── references/
    │   ├── report-rubric.md
    │   └── status-definitions.md
    ├── templates/
    │   └── report-template.md
    ├── examples/
    │   └── example-output.md
    └── scripts/
        └── normalize_report.py
```

脚本仅作为受控 Sandbox 的输入，不在主进程直接执行。

## 2. Loader Pipeline

`Discover → Read Manifest → Parse Front Matter → Validate Schema → Resolve Package Root → Validate Paths → Checksum → Dependency Check → Security Scan → Publish → Runtime Load`

## 3. Front Matter

Manifest 至少包含 name、description、version、tags、allowed-tools、license、author、compatibility、resources。描述字段用于检索，不能过长或含可执行指令。

## 4. 路径安全

所有引用文件必须 resolve 后仍位于 Skill Package Root。拒绝绝对路径、`..`、符号链接逃逸、设备文件和超大文件。读取时限制文件大小、总加载字节和递归深度。

## 5. 渐进式加载算法

- Catalog 阶段只加载 Name / Description / Tags；
- Match 阶段根据意图、任务类型和 Agent Release 选择候选；
- Activate 阶段加载 SKILL.md；
- Resource Stage 按需加载 references / templates；
- Execute Stage 只把脚本路径交给 Sandbox Service。

## 6. Skill 与 Tool Grant

Skill 的 `allowed-tools` 是推荐 / 依赖声明，不是授权。Tool Registry 和 Policy Engine 仍需验证 Agent Release Binding 与 Principal 权限。

## 7. 版本固定

Run 创建时固定 Skill Version 和 checksum。运行中 Skill 更新不改变当前 Run。安全撤销可阻断后续脚本执行或 Tool 调用。

## 8. 安全扫描

扫描 Prompt Injection 风险、可疑 URL、脚本、隐藏文件、依赖、许可证、敏感数据和超大内容。扫描报告与包 checksum 绑定。

## 9. 测试

路径穿越、符号链接逃逸、重复名称、缺失引用文件、Front Matter 非法、Skill 描述冲突、Tool 不可用、脚本超时、Skill 被撤销后继续执行。
