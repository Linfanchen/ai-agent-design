# Tool Registry：注册、Schema、绑定与版本

## 本文档在系统中的作用

Tool Registry 管理平台允许 Agent 调用的能力定义和版本。它回答系统有哪些工具、工具需要什么参数、谁提供实现、哪些 Agent 可以使用、当前版本是否可用，但不直接执行工具。

## 1. Tool 的五个概念

- Tool Definition：逻辑能力身份，如 `mentorship.learner.search`。
- Tool Version：输入输出 Schema、描述和行为契约的不可变版本。
- Tool Handler：Python 具体实现。
- Tool Binding：某 Agent Release 对 Tool Version 的授权引用。
- Tool Execution：一次真实调用记录。

不要将 Definition、Handler 和 Execution 混为一个 Python 函数。

## 2. 注册方式

Built-in Registry 由代码显式注册核心工具；Plugin Registry 通过 Plugin SDK 注册扩展工具；MCP Registry 将外部 MCP Tool 作为 Remote Binding；Tenant Registry 由租户配置启用已审核工具。

不允许用户通过 API 提交任意 Python 文件并在主进程 import 执行。

## 3. Tool Definition 字段

tool_id、tool_key、display_name、description、domain、operation、input_schema、output_schema、risk_level、side_effect_class、timeout_seconds、max_retries、idempotency_mode、required_permissions、approval_policy_ref、handler_ref、version、status。

## 4. Schema 来源

优先使用 Pydantic V2 Model 作为 Python Input / Output Contract，导出 JSON Schema 给模型。Schema 限制额外字段、长度、枚举、数值范围和嵌套深度。复杂业务不变量由 Domain Validator 执行。

## 5. Handler Registry

```python
from typing import Protocol

class ToolHandler(Protocol):
    async def execute(self, args, context) -> "ToolResult": ...

class ToolRegistry:
    def __init__(self):
        self._handlers = {}

    def register(self, tool_key: str, version: str, handler: ToolHandler):
        key = (tool_key, version)
        if key in self._handlers:
            raise ValueError(f"duplicate tool binding: {key}")
        self._handlers[key] = handler

    def resolve(self, tool_key: str, version: str) -> ToolHandler:
        try:
            return self._handlers[(tool_key, version)]
        except KeyError:
            raise ToolBindingUnavailable(tool_key, version)
```

真实实现需在启动时验证 Handler 与 Manifest 的版本、Schema 和能力声明。

## 6. 注册生命周期

`Discovered → Validated → Registered → Published → Bound → Invokable → Deprecated / Revoked`

注册不等于发布；发布不等于 Agent 获得使用权限；Binding 不等于每次 Invoke 都通过 Policy。

## 7. 版本规则

Tool Definition ID 稳定；Tool Version 不可变；Handler 部署版本必须与 Tool Version 匹配；Breaking Schema 变更创建新 Major Version；旧版本可在兼容期内运行；安全撤销可立即禁用；Run 固定 Tool Version，但 Invoke 时仍检查 Revocation。

## 8. Search / Describe / Invoke

Search 返回简短摘要；Describe 返回完整 Schema 和风险信息；Invoke 接受 Tool Version、Arguments、Correlation ID、Idempotency Key。模型不能指定 Handler Import Path。

## 9. 测试

每个 Tool Version 有 Schema Test、Handler Contract Test、Permission Test、Idempotency Test、Timeout Test、Output Validation Test 和 Regression Case。

## 10. Tool Metadata 的分层

建议区分：
- Definition：稳定 Tool Key；
- Contract Version：输入输出 Schema 和副作用声明；
- Handler Build：实际部署代码构建版本；
- Binding：Agent Release / Tenant 的可用关系；
- Runtime Availability：当前健康状态；
- Execution Record：真实调用事实。

Handler Build 与 Contract Version 不必一一对应，但必须有兼容声明。若 Handler 行为改变了副作用语义或输出结构，应发布新的 Contract Version。

## 11. Tool Schema 设计原则

- 字段名面向业务，而不是数据库列名；
- Description 说明含义和边界，不塞入全部业务文档；
- Enum 尽量使用稳定值；
- 日期 / 时间明确时区；
- ID 字段说明 ID 类型；
- 不把自由 SQL / URL / Python Code 当普通参数；
- 对分页、批量和字符串长度设置上限；
- `extra="forbid"` 防止模型悄悄多传字段。

## 12. Registry 启动验证

应用启动或 Release 加载时验证：
- Handler 已注册；
- Pydantic Input / Output Model 可解析；
- 导出的 JSON Schema 与登记 Schema Hash 一致；
- Tool Version 状态 Published；
- 所需权限定义存在；
- Timeout / Retry / Idempotency 配置合理；
- 依赖服务健康或有降级策略。

## 13. Registry 与 Discovery 的缓存

Tool Metadata 可按 Version 缓存；Binding / Permission 需按 Tenant / Agent Release 缓存并设置失效机制。缓存不能跨租户共享授权结果。Tool 被 Revoked 时需发布失效事件。

## 14. 工具注册审计

记录谁创建、谁审核、谁发布、哪些 Agent Release 绑定、何时撤销、变更了哪些 Schema 和权限。Tool Version 不允许原地修改已发布内容。
