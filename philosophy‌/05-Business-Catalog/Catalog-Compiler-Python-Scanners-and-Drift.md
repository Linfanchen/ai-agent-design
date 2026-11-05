# Business Catalog Compiler：Python 扫描器与漂移检测

## 本文档在系统中的作用

Catalog Compiler 把技术资产编译成可查询的业务目录。它不是运行时 Agent 的一部分，而是 CI / Indexing / Admin 工作流中的构建系统。

## 1. 扫描输入

### OpenAPI Scanner
提取 Path、Method、OperationId、Tags、Request / Response Schema、Security、Deprecated、Error Response。

### Python Source Scanner
基于 AST 和框架适配器识别 FastAPI Router、Pydantic Request / Response Model、Dependency、Permission Decorator、Service Call、SQLAlchemy Model、Domain Enum。静态扫描不能执行被扫描代码。

### PostgreSQL Scanner
提取 Schema、Table、Column、Comment、Primary Key、Foreign Key、Check Constraint、Enum、Index。生产扫描账号使用只读权限。

### Documentation Scanner
提取术语、业务流程、状态机、角色说明和接口示例。文档内容默认视为待审核知识。

## 2. 编译流程

`Discover → Parse → Normalize → Link → Infer Candidate → Validate → Human Review → Publish Snapshot`

- Discover：发现资产。
- Parse：解析原始结构。
- Normalize：统一命名、类型和 ID。
- Link：关联 API、DTO、Entity、Table、Permission。
- Infer：生成业务描述和候选 Capability。
- Validate：检查 Schema、关系、引用。
- Human Review：审核风险、语义和权限。
- Publish：生成不可变 Snapshot。

## 3. Python AST 扫描注意事项

不执行目标模块；不调用任意 import hook；识别装饰器时允许配置别名映射；对动态路由、运行时注册和反射代码标记 unresolved；Source Location 保存文件路径、行号和 Git Commit；区分 source 与 generated；解析失败不静默跳过。

## 4. Schema Normalization

将 Pydantic / OpenAPI / SQL 类型映射到统一 Schema Type：string、integer、number、boolean、object、array、enum、date / datetime、identifier、reference。保留原始 Schema 作为 evidence，规范化 Schema 作为检索和比较对象。

## 5. 漂移检测

比较 API path / method / operationId、字段新增 / 删除 / 类型变化、Required 变化、Enum 新增 / 删除、Permission 变化、Entity Relationship 变化、Capability Binding 失效、Python Handler 不存在、Catalog 描述与代码签名不一致。

分级：compatible、potential_breaking、breaking、security_critical。

## 6. 影响图

建立有向依赖：`API → DTO → Entity → Capability → Tool Version → Agent Release → Evaluation Case`。当某字段从 optional 变 required，Compiler 应列出受影响 Tool、Skill、Agent Release 和评测用例。

## 7. 发布策略

Catalog Snapshot 不可变；未通过审核的 Breaking Change 不得进入生产。生产 Run 固定 Snapshot。安全关键变更可通过 Revocation Overlay 即时阻断旧 Binding。

## 8. 自动化测试

准备 FastAPI / Pydantic / SQLAlchemy Fixture 项目，验证扫描器识别路由、Schema、权限、关系和行号。再人为修改 DTO / Enum / API 路径，验证 Drift Detector 能识别变更。
