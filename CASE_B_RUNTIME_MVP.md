# CASE_B_CANCELLATION_V1 Portable Runtime MVP

这是只服务于 Case B 取消场景的纯 Runtime 模块，不是通用 Runtime。模块不访问网络、数据库、驾驶舱或工作区文件，不生成 `decision_log`、`todo_items`、任务 ID 或状态记录。

唯一业务调用入口：

```python
from revenue_validator.case_b_runtime import run_case_b

decision = run_case_b(workflow_context)
```

- 输入：普通 `dict` 或 `WorkflowContext`
- 输出：严格 Decision `dict`
- `order_log.change_rooms >= 0` 时返回 `None`
- 仅使用 Python 标准库

## 固定契约

模块内固定提供：

- `WORKFLOW_CONTEXT_JSON_SCHEMA`
- `DECISION_JSON_SCHEMA`
- `WorkflowContext`
- `run_case_b`

CatDesk 独立交付包已经包含对应的 JSON Schema 文件、字段分级表和历史 Case：

[deliverables/CatDesk_CASE_B_Runtime](deliverables/CatDesk_CASE_B_Runtime)

## CatDesk 接入边界

CatDesk 负责：

1. 构建真实 `WorkflowContext`。
2. 使用 `(workflow_case_id, order_log.event_id)` 或现有唯一键规则去重。
3. 调用 `run_case_b()`。
4. 按自身 Schema 落库。
5. 按现有 `todo_items.id` 规则生成 RiskAlert。
6. 完成 Human Gate、状态流转、事务和重试。

Runtime 不执行上述外部行为，不自动确认，不生成 `strategy_case`，不自动调价。

## 真实驾驶舱集成状态

截至 2026-08-26，纯 Runtime 模块的职责边界没有改变，但它已由现有收益经理驾驶舱完成端到端接入：

`order_log 真实取消事件 → Context Builder → run_case_b() → Decision → decision_log + todo_items → RiskAlert → Human Gate → strategy_case`

本次读取并处理 3 条真实取消事件，三条 Decision 均完成落库、RiskAlert 展示、Human Gate 确认和 `strategy_case` 回写关联；真实事件去重链路也已生效。三条样本均命中 `OBSERVATION_WINDOW_ACTIVE`，输出 `OBSERVE`，且 `data_quality = DEGRADED`。

这将 `CASE_B_CANCELLATION_V1` 的 Readiness 从“本地/模拟可运行”提升为“真实经营数据 + 真实驾驶舱 Human Gate 已完成 MVP 集成验证”。它不改变以下事实：

- Decision quality 仍在验证中；
- `ops_snapshot` 上下文字段仍在补齐；
- Observation 到期自动重评尚未完成；
- Outcome effectiveness 尚不能宣称已验证；
- 这仍不是通用 Runtime，也不自动调价。

完整状态矩阵见 [RUNTIME_READINESS.md](RUNTIME_READINESS.md)，本次脱敏证据摘要见 [evidence/CASE_B_CANCELLATION_V1_INTEGRATION_2026-08-26.md](evidence/CASE_B_CANCELLATION_V1_INTEGRATION_2026-08-26.md)。

## 已固定的业务边界

- 仅 `change_rooms < 0` 触发
- 仅 D+0~D+2 主动判断，D+3 以上 `OBSERVE`
- 早/晚观察窗口分别默认 60/30 分钟
- Pickup 恢复优先 `HOLD`
- 库存压力或 Pickup 缺失时保守 fallback
- 晚取消＋库存压力高＋Pickup 停滞才进入价格调整候选
- 门店原因取消降低主动降价优先级
- Hard Price Floor 不得突破
- 业主不降价约束命中时转 `ESCALATE`
- 不输出未经数据支持的具体价格

## 已验证 Case

现有 Case B 规范回放命中：

`PRICE_CANDIDATE_OWNER_CONSTRAINT_ESCALATE`

最终 Decision 为 `ESCALATE`：不直接降价，将库存风险、Pickup 停滞和业主约束提交驾驶舱 Human Gate。

输入和完整输出位于 CatDesk 交付包的 `examples/` 目录。
