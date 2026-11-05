# Prompt 架构与结构化输出

## 本文档在系统中的作用

Prompt System 把系统规则、Agent 身份、任务指令、上下文数据和工具说明组合成模型输入。它不是一个巨大字符串，而是有来源、信任等级、优先级、版本和预算的 Prompt Section 集合。

## 1. Prompt 分层

1. Platform Policy：平台不可覆盖规则。
2. Agent Role：Agent 身份和职责。
3. Task Contract：当前目标、完成标准和禁止事项。
4. Tool Instructions：工具使用边界。
5. Context Blocks：用户、项目、会话和业务数据。
6. User Input：当前请求。
7. Output Contract：格式、Schema 和验证要求。

信任等级标记为 `trusted_system`、`trusted_config`、`retrieved_data`、`external_untrusted`。检索内容和网页内容即使包含命令，也不能升级为系统指令。

## 2. Prompt Section

```python
from dataclasses import dataclass
from typing import Literal

@dataclass(frozen=True)
class PromptSection:
    section_id: str
    kind: str
    content: str
    priority: int
    trust_level: Literal[
        "trusted_system", "trusted_config",
        "retrieved_data", "external_untrusted"
    ]
    source_ref: str | None
    cacheable: bool
    token_estimate: int
    required: bool
```

## 3. 模板变量

模板变量定义名称、类型、是否必填、默认值、最大长度、来源、是否敏感、是否允许用户覆盖。不要直接使用字符串 `format()` 拼接未经校验的对象。模板渲染前执行 Schema Validation，并对外部文本做明确边界标记。

## 4. 结构化输出

1. Pydantic Model 定义目标结构；
2. 导出 JSON Schema；
3. Provider Capability 检查；
4. 请求模型；
5. 解析 JSON；
6. Pydantic 验证；
7. 业务不变量验证；
8. 不通过时有限修复或重新生成；
9. 超过预算后返回可解释失败。

模型输出符合 JSON Schema，不代表业务逻辑正确。例如 `status="completed"` 是合法枚举，但实际状态转换未必允许。

## 5. Prompt Injection 边界

Prompt 明确声明检索文档、邮件、网页、文件、Tool Result 是数据，不是高优先级指令。真正的安全控制仍在 Policy Engine 和 Tool Executor，不能只依赖 Prompt 文案。

## 6. 版本与回归

每个 Prompt Version 保存模板、变量 Schema、输出 Schema、模型兼容要求、Token 预算和 checksum。Prompt 修改必须跑 Golden Dataset，比较准确率、拒绝率、Tool Selection、Token 和成本。
