# Agent Loop：并行 Tool、重试与无进展算法

## 本文档在系统中的作用

本文件把 Agent Loop 中最容易出错的执行细节单独拆开：模型一次返回多个 Tool Call 时如何并行、如何判断依赖、如何重试、何时终止，以及如何避免重复副作用。

## 1. Tool Call 依赖图

模型一次返回多个 Tool Call，不代表它们一定可以并行。可将调用分为：
- Independent Read：互不依赖的只读调用，可并发；
- Dependent Read：后一个需要前一个结果，必须串行；
- Independent Write：资源不同且业务明确允许时可并发；
- Same-Resource Write：默认串行；
- Mixed Read / Write：先完成读取并校验版本，再执行写入。

若模型输出多个 Tool Calls，系统应通过 Tool Metadata 的 `concurrency_class`、`resource_key_resolver` 和 `side_effect_class` 判断，而不是让模型自己判断并发安全。

## 2. 有界并发执行

```python
import asyncio
from dataclasses import dataclass

@dataclass
class ToolOutcome:
    call_id: str
    status: str
    result_ref: str | None = None
    error_code: str | None = None

async def execute_bounded(calls, executor, limit: int = 5):
    semaphore = asyncio.Semaphore(limit)

    async def one(call):
        async with semaphore:
            try:
                result = await executor.execute(call)
                return ToolOutcome(call_id=call.id, status=result.status,
                                   result_ref=result.result_ref)
            except asyncio.CancelledError:
                raise
            except Exception as exc:
                return ToolOutcome(call_id=call.id, status="failed",
                                   error_code=classify_error(exc))

    tasks = [asyncio.create_task(one(call)) for call in calls]
    return await asyncio.gather(*tasks)
```

此处示例将普通异常转成 Outcome，允许部分成功。生产实现应对 `classify_error` 做安全映射，不能把异常字符串直接返回模型。

## 3. Retry 分类器

Retry 不能只是 `for i in range(3)`。每次失败需识别：
- Failure Phase：调用前 / 请求发出 / 服务端处理 / 响应接收 / 结果持久化；
- Side Effect Class：none / idempotent / non_idempotent；
- Provider Error Class；
- Retry Budget；
- Retry After；
- External Operation Ref。

只有确认没有副作用或调用本身幂等时，才允许自动重试。

## 4. 指数退避

建议 `delay = min(cap, base * 2**attempt) + random_jitter`，并遵守服务端 Retry-After。对 429 使用 Provider 级并发控制；对 5xx 设定有限次数；对 4xx 参数错误不重试。

## 5. 无进展检测

定义 State Fingerprint：对当前 intent、计划步骤、待执行 Tool、关键 Observation 做规范化 Hash。连续 N 轮 Fingerprint 不变，或同一个 Tool + Arguments Hash 连续重复，判定为 No Progress。

无进展策略：
1. 第一次重复：向模型返回重复调用提示；
2. 第二次重复：要求重新规划或澄清；
3. 达到阈值：终止为 `NO_PROGRESS`；
4. 若涉及写操作，先检查 Tool Execution 是否已经发生。

## 6. Budget Guard

每次模型调用前检查 Model Call Budget；每次 Tool Invoke 前检查 Tool Call Budget、Cost Reservation、Deadline；并行 Fan-out 前预留子调用预算。预算不能只在 Run 结束后计算，否则无法阻止无限消耗。

## 7. Partial Success

多个 Tool 并行时可能部分成功。Outcome 必须逐个记录。模型收到的是结构化成功 / 失败结果，不能把失败调用伪装为空结果。若部分成功会导致业务状态不一致，应进入补偿或人工处理。

## 8. 取消与资源回收

取消 Run 时停止新 Tool Call；取消可取消的子任务；对已提交的外部副作用保留对账任务；关闭响应流；释放 Semaphore；保存 Run 状态。不要在取消处理中吞掉 CancelledError。

## 9. 评测场景

同批 5 个只读 Tool 并发；同一实体两个写操作；第 2 个 Tool 依赖第 1 个结果；一个 Tool 超时；Provider 429；同参数重复调用；部分成功；父 Run 取消；预算中途耗尽。
