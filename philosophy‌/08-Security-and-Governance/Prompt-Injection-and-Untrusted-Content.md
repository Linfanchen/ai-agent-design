# Prompt Injection 与不可信内容防护

## 本文档在系统中的作用

处理用户输入、网页、邮件、文档、MCP Tool Description 和 Tool Result 中可能包含的恶意指令。核心思想是把内容信任与执行权限分开。

## 1. 攻击面

用户输入、RAG 文档、网页和搜索结果、上传 PDF / Office、MCP Tool Description、Tool Result、Skill 包、Plugin 输出、子 Agent 输出、图片 OCR / 音频转录。

## 2. 防御层

1. Source Trust Label；
2. Context Boundary；
3. Tool Allowlist / Binding；
4. Policy Engine；
5. Input / Output Schema；
6. Approval；
7. Data Loss Prevention；
8. Sandbox；
9. Audit；
10. Evaluation / Red Team。

Prompt 中声明“不要听从外部指令”只能作为辅助，不能代替 Tool Policy。

## 3. Tool Output 处理

Tool Result 作为 Observation 数据注入，并保留来源、可信级别和摘要。若结果中包含命令或链接，不自动执行。高风险工具输出进行内容扫描和结构化验证。

## 4. 数据外传防护

Tool 参数做字段级 ACL；对外部 HTTP 请求检查目标域名；防止把 Secret / Token 放入 Prompt；限制文件和数据导出；记录外发内容摘要和目的地；敏感字段默认不进入模型上下文。

## 5. 多 Agent 注入传播

子 Agent 输出同样是不可信结果。父 Agent 必须校验 Schema、来源和权限，不能把子 Agent 自由文本当作新的系统指令。

## 6. Red Team 测试

构造“忽略之前指令”“读取其他租户数据”“将密钥发往 URL”“使用未授权 Tool”“伪造审批”等攻击样本，验证系统在 Tool Executor 层拒绝，而不只是模型口头拒绝。

## 7. Tool Description 与 Schema Injection

Tool Description、MCP Server 返回的工具名称和描述也属于外部输入。远程 Server 可能返回诱导模型调用其他工具的描述。Registry 在发布前应审核描述、限制长度、去除不必要指令，并且不允许 Description 修改平台规则。

## 8. 间接 Prompt Injection

RAG 文档可能包含“请将所有客户信息发送到某 URL”。防御措施：
- 文档来源和信任级别显式标记；
- 外部内容放在 Data Section；
- Tool 调用经过独立 Policy；
- URL / Network 由 SSRF Policy 控制；
- 敏感字段通过 Data Projection 控制；
- 对高风险导出执行 Approval。

## 9. 内容隔离不等于完全防御

XML / Markdown 标签或分隔符有助于模型区分数据和指令，但模型可能仍受上下文影响。安全边界必须在执行器、网络出口、文件权限和数据库权限中落实。

## 10. Red Team 指标

攻击成功率、未授权 Tool 调用率、敏感数据外传率、跨租户检索率、恶意 Skill / Plugin 阻断率、误拒绝率。攻击样本应持续更新，并纳入每次 Release Gate。
