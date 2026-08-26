# CatDesk 接入包：CASE_B_CANCELLATION_V1

这是 Case B 取消场景的最小可移植 Runtime。CatDesk 只需完成：

`构建真实 WorkflowContext → 调用 run_case_b → 按现有规则去重、落库和生成 RiskAlert`

模块不连接数据库、不写文件、不生成 `todo_items.id`、不维护状态，也不修改 CatDesk。

> 集成状态更新（2026-08-26）：CatDesk 已使用 3 条真实 `order_log` 取消事件完成 Context Builder、Decision 落库、RiskAlert、Human Gate 确认及 `strategy_case` 回写的 MVP 端到端验证。纯 Runtime 模块边界与业务规则没有改变。完整状态见项目根目录 `RUNTIME_READINESS.md` 和 `evidence/CASE_B_CANCELLATION_V1_INTEGRATION_2026-08-26.md`。

## 1. 唯一业务调用入口

把 `case_b_runtime.py` 放进 CatDesk 可导入目录：

```python
from case_b_runtime import run_case_b

decision = run_case_b(workflow_context)
```

- 输入：普通 Python `dict`，结构见 `schemas/workflow_context.schema.json`
- 输出：严格 Decision `dict`，结构见 `schemas/decision.schema.json`
- 如果 `order_log.change_rooms >= 0`，未触发 Case B，返回 `None`
- 不依赖第三方 Python 包，仅使用标准库，Python 3.10+

## 2. Runtime 必填字段

以下字段决定 Runtime 能否识别事件与计算 D+N：

| 字段 | 类型 | 用途 | 缺失结果 |
|---|---|---|---|
| `order_log.change_rooms` | number | `< 0` 才触发取消 Case | 抛出输入错误 |
| `order_log.target_date` | `YYYY-MM-DD` | 计算 D+0~D+N | 触发后缺失则输入错误 |
| `order_log.cancellation_time` | ISO 8601 datetime | 判断取消日期和早晚窗口 | 触发后缺失则输入错误 |
| `order_log` | object | 取消事件容器 | Schema 不通过 |
| `ops_snapshot` | object | 运营快照容器，可以为空对象 | Schema 不通过 |

## 3. CatDesk 集成必填字段

这些字段不阻断纯 Decision，但 CatDesk 在正式落库/建 RiskAlert 前应保证存在：

| 字段 | 用途 | Runtime 缺失行为 |
|---|---|---|
| `order_log.event_id` | CatDesk 使用 `(workflow_case_id, event_id)` 去重 | 加入 `missing_context`，Decision 仍生成 |
| `order_log.store_id` 或 `ops_snapshot.store_id` | 关联门店 | 加入 `missing_context` |
| `ops_snapshot.store_full_name` | RiskAlert 使用看板一致的门店全称 | 加入 `missing_context`；Runtime 不生成待办 |

建议 CatDesk 对 `(workflow_case_id, order_log.event_id)` 建唯一约束；如果源事件暂时没有 `event_id`，应在 Context 构建层先生成稳定事件键。

## 4. 可选字段及 fallback

| 字段 | 正常用途 | 缺失时 fallback |
|---|---|---|
| `ops_snapshot.as_of` | 计算取消后已观察分钟数 | 使用取消时间，等同刚进入观察窗口 |
| `order_log.cancellation_reason` | 识别门店原因取消 | 记录缺失，不应用门店原因降级信号 |
| `order_log.pickup_after_cancel` | 判断 Pickup 恢复/停滞 | 高库存压力时转 `OBSERVE` |
| `order_log.pickup_observation_minutes` | 补充实际 Pickup 观察时长 | 使用 `as_of - cancellation_time` |
| `ops_snapshot.inventory_pressure_high` | 直接判断库存压力 | 尝试读取 `inventory_status` |
| `ops_snapshot.inventory_status` | 使用上游库存压力分类 | 两种压力字段都缺时转 `OBSERVE` |
| `ops_snapshot.occ` | 判断依据与审计 | 记录 `current_occ` 缺失，不单独中止 |
| `ops_snapshot.sold` | 判断依据与审计 | 记录 `current_sold` 缺失，不单独中止 |
| `ops_snapshot.remaining` | 取消后剩余库存依据 | 记录缺失；压力标签存在时仍可判断 |
| `rms_daily.current_price` | 价格安全检查和依据 | 不生成具体价格，记录缺失 |
| `rms_daily.hard_price_floor` 或 `store_constraints.hard_price_floor` | Hard Price Floor | 需要价格调整时转 `ESCALATE` |
| `rms_daily.market_price_band` | 市场价格带辅助 | `DEGRADED`，不阻断 |
| `comp_price_monitor.competitor_prices` | 竞品辅助信息 | `DEGRADED`，不阻断 |
| `store_constraints.owner_no_price_decrease` | 业主不降价约束 | 默认未命中，但记录约束源缺失 |
| `store_constraints.owner_min_price` | 业主最低价约束 | 不写具体最低价 |
| `store_constraints.notes` | 附加约束说明 | 空列表 |

可配置参数及默认值：

- `late_cancel_cutoff = 18:00`
- `early_observation_minutes = 60`
- `late_observation_minutes = 30`

## 5. 已固定的 Case B 行为

- D+3 以上直接 `OBSERVE`
- 较早取消且仍在观察窗口：`OBSERVE`
- Pickup 已恢复：优先 `HOLD`
- 库存压力不明显：`HOLD`
- 库存压力或 Pickup 缺失：保守 `OBSERVE`
- 晚取消＋库存压力高＋Pickup 停滞：进入 `PRICE_ADJUST` 候选
- 门店原因取消：降低主动降价优先级并转人工
- Hard Price Floor 缺失或当前价低于底线：`ESCALATE`
- 业主不降价：不直接降价，`ESCALATE`
- 不输出未经数据支持的具体价格
- 所有 Decision 由驾驶舱 Human Gate 处理；Runtime 不自动确认

## 6. 已验证历史 Case

- 输入：`examples/verified_history_context.json`
- 验证结果：`examples/verified_history_result.json`
- 命中分支：`PRICE_CANDIDATE_OWNER_CONSTRAINT_ESCALATE`
- 最终动作：`ESCALATE`
- 原因：D0 晚取消、库存压力高、Pickup 停滞，但命中业主不降价约束
- 数据质量：`DEGRADED`，缺竞品、OCC、sold、Hard Price Floor 和价格带

样例中来自既有 Case B 规范的事实与仅用于回放的补充字段，已经在 `fixture_metadata` 中分开标注。

验证：

```powershell
python test_runtime.py
```

预期输出：

```text
PASS: CASE_B_CANCELLATION_V1 verified history replay
```

## 7. CatDesk 落库职责

Runtime 返回 Decision 后，CatDesk 负责：

1. 使用现有唯一键规则检查是否重复。
2. 按 CatDesk 自身 Schema 写入 Decision。
3. 使用 CatDesk 现有 `todo_items.id` 规则生成 RiskAlert。
4. 使用 `ops_snapshot.store_full_name` 展示门店全称。
5. 设置 CatDesk 自身的待审状态并进入 Human Gate。
6. 管理事务、并发、重试、失败补偿和状态流转。

Runtime 不包含以上外部行为。

## 8. 文件清单

- `case_b_runtime.py`：唯一 Runtime 模块
- `schemas/workflow_context.schema.json`：固定输入 Schema
- `schemas/decision.schema.json`：固定输出 Schema
- `examples/verified_history_context.json`：已验证 Case 输入
- `examples/verified_history_result.json`：命中分支与期望 Decision
- `test_runtime.py`：独立冒烟测试
- `README.md`：本接入说明
