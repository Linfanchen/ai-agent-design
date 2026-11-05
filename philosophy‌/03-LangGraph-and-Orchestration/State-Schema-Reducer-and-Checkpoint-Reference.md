# LangGraph State Schema、Reducer 与 Checkpoint 参考

## 本文档在系统中的作用

提供 Graph State 的设计样例，并明确哪些字段可以进入 Checkpoint，哪些应由 Runtime Context 或数据库管理。LangGraph State 是工作流状态，不是全系统的万能数据容器。

## 1. State 分区

推荐 State 分为：
- `request`：任务输入引用；
- `plan`：结构化计划；
- `entities`：已解析的稳定业务 ID；
- `observations`：工具结果引用；
- `pending_action`：待执行动作摘要；
- `approval`：审批引用和状态；
- `messages`：模型交互消息；
- `errors`：可恢复错误；
- `final`：最终结果引用。

Tenant、Principal、权限对象、数据库 Session、HTTP Client 和 Secret 不进入 State。

## 2. TypedDict 样例

```python
from typing import TypedDict, Annotated
import operator

class Observation(TypedDict):
    tool_execution_id: str
    status: str
    result_ref: str | None
    evidence_ref: str | None

class AgentState(TypedDict, total=False):
    request_ref: str
    task_type: str
    plan: list[dict]
    resolved_entities: dict[str, str]
    observations: Annotated[list[Observation], operator.add]
    errors: Annotated[list[dict], operator.add]
    approval_id: str | None
    final_result_ref: str | None
```

`operator.add` 适合追加不可变记录。对于“当前状态”字段，不应随意用 append reducer；应由单一节点负责更新，或使用明确的合并函数。

## 3. Reducer 设计原则

Reducer 需要确定性、可测试和可解释：
- 追加型：仅适用于不可变 Event / Observation；
- 覆盖型：只允许明确的单一写入者；
- 集合并集：适合无序 ID 集合，但要定义排序；
- 冲突型：不同分支写同一业务字段时返回冲突，而不是随机覆盖。

并行 Graph Branch 的完成顺序不应改变业务结论。

## 4. Graph Config

`thread_id` 标识跨 Run 的 Graph 连续性；`run_id` 标识当前一次任务。建议将二者分开存储，并通过可信 Runtime Config 注入。客户端不能任意指定其他 Tenant 的 `thread_id`。

## 5. Checkpoint 与外部存储

Checkpoint 保存恢复所需状态；业务文件保存到 Artifact Store；大段 Tool Result 保存到受控 Result Store；Audit 保存不可变执行事实。State 只保存引用和必要的小型结构化数据。

## 6. State Migration

每个 State Schema 有整数或语义版本。迁移流程：
1. 读取旧 Checkpoint Version；
2. 校验该版本是否支持；
3. 按显式迁移函数转换；
4. 校验新 Schema；
5. 写入新版本；
6. 保留迁移事件和错误。

不要用“字段不存在就默认空字符串”的方式掩盖字段语义变化。

## 7. 恢复一致性

恢复 Graph 前：
- 确认 Run 仍可继续；
- 确认 Tenant / Principal 授权仍有效；
- 确认 Release 和依赖版本仍可执行；
- 检查未完成 Tool Execution；
- 检查 Approval 是否过期 / 撤销；
- 检查预算和 Deadline；
- 再读取 Checkpoint。

## 8. 测试

Reducer 顺序稳定、Checkpoint 序列化、旧 State Migration、非法 Thread 访问、并行 Branch 冲突、Interrupt Resume、恢复时权限撤销和 Tool UNKNOWN。
