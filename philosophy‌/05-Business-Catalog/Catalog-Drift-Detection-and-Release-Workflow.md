# Business Catalog：版本漂移检测与发布流程

## 本文档在系统中的作用

Catalog 与业务源码、API Schema、数据库实体持续变化。Drift Detection 用于识别 Agent 仍引用旧能力、字段或权限定义的情况，防止“目录看起来可用，实际 API 已变化”。

## 1. Snapshot 内容

Catalog Snapshot 应包含：
- Git Commit；
- OpenAPI / Python Scanner Version；
- Entity / Attribute / Relationship；
- Capability / Input / Output Schema；
- API Binding；
- Permission Mapping；
- Business Rule Reference；
- Source File / Symbol / Line；
- Checksum；
- Review Status。

Snapshot 发布后不可变。

## 2. Drift 类型

| Drift | 示例 | 风险 |
|---|---|---|
| Removed Route | API 删除 | 执行失败 |
| Path / Method Changed | POST 改 PATCH | 调用错误 |
| Input Breaking Change | 必填字段新增 | 参数不兼容 |
| Output Breaking Change | 字段改名 | 结果解析错误 |
| Entity Change | 主键或关系变化 | 实体解析错误 |
| Permission Change | 新增审批 | 越权风险 |
| Semantic Change | status 含义变化 | 业务决策错误 |
| Source Deleted | 代码模块移除 | Binding 失效 |

## 3. Drift Pipeline

`Scan New Commit → Compile Candidate Snapshot → Diff Old / New → Classify Severity → Run Contract Tests → Human Review → Publish → Notify Agent Releases → Re-Evaluate`

## 4. 兼容等级

- Patch：描述、示例修正，不改变语义；
- Minor：新增可选字段或能力；
- Major：移除 / 改变字段、权限、业务含义或副作用；
- Revocation：安全撤销，立即阻断旧 Binding。

## 5. Agent Release 兼容性

每个 Agent Release 固定 Catalog Snapshot。新 Snapshot 发布后，生成兼容报告：
- 所引用 Capability 是否存在；
- Tool Schema 是否兼容；
- 权限是否改变；
- 业务规则是否变化；
- 是否需要重新评测；
- 是否必须阻断旧 Release。

## 6. 灰度发布

先在测试 Tenant 或 Canary Agent Release 使用新 Snapshot；观察 Tool Error、参数校验失败、实体解析、权限拒绝和业务结果。高风险写 Capability 不建议自动升级 Snapshot。

## 7. 回滚

Catalog Snapshot 回滚只能恢复语义目录版本，不能撤销已经发生的业务写操作。回滚前检查业务 API 兼容性和外部副作用。安全撤销应优先于回滚旧版本。

## 8. 测试

删除路由、字段类型变化、权限新增、状态枚举变化、别名变更、数据库关系变化、旧 Agent Release 继续执行和紧急 Revocation。
