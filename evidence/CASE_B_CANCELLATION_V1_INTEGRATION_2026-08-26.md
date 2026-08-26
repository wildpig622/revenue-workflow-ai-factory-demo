# CASE_B_CANCELLATION_V1 Integration Evidence

> 本文件是脱敏的 MVP 集成验证摘要，不包含订单、门店或人员明细，不用于证明收益效果。

## 验证概要

- 验证日期：2026-08-26
- Workflow Case：`CASE_B_CANCELLATION_V1`
- 真实样本数量：3 条取消事件
- 数据入口：真实经营 `order_log`
- 验证对象：现有收益经理驾驶舱的端到端决策与人工门禁链路

## 已验证端到端链路

`order_log 真实取消事件 → Context Builder → run_case_b() → Decision → decision_log + todo_items → RiskAlert → Human Gate → strategy_case`

验证结果：

1. 从 2026-08-26 的 `order_log` 读取到 3 条真实取消事件；
2. 3/3 均成功进入 `CASE_B_CANCELLATION_V1` Runtime；
3. 3/3 Decision 均成功写入 `decision_log` 和 `todo_items`；
4. RiskAlert 能展示完整五要素 Decision；
5. Human Gate 的“确认执行”已实测成功；
6. 确认后 `strategy_case` 正确生成，并与对应 `todo_items` 关联；
7. Runtime 去重链路已生效。

## 代表性 Decision

本次 3 条真实样本结果一致：

```text
workflow_case_id: CASE_B_CANCELLATION_V1
reason_code: OBSERVATION_WINDOW_ACTIVE
action: OBSERVE
data_quality: DEGRADED
human_gate: REQUIRED
```

含义：事件已进入取消场景 Runtime，但仍处于观察窗口；系统生成保守的 `OBSERVE` Decision，并进入驾驶舱人工门禁，而不是自动执行价格动作。

## Human Gate 回写结果

- RiskAlert 已成功承载并展示 Decision；
- “确认执行”操作已成功完成；
- 确认结果已生成 `strategy_case`；
- `strategy_case` 与原 `todo_items` 关联正确；
- 本次仅证明人工门禁与动作记录闭环可用，不代表自动调价或收益结果已验证。

## 当前限制

- ✅ Integration / end-to-end workflow 已验证；
- ⚠️ Decision quality 仍在验证中；
- ⚠️ `ops_snapshot` 上下文字段仍在补齐；
- ⚠️ Observation 到期自动重评尚未完成；
- ⚠️ Outcome effectiveness 尚不能宣称已验证；
- ⚠️ 当前仅覆盖 `CASE_B_CANCELLATION_V1`，不是通用 Runtime；
- ⚠️ 本次 3 条样本均处于观察窗口且数据质量为 `DEGRADED`，样本不足以评价价格策略效果。

## 证据结论

`CASE_B_CANCELLATION_V1` 已完成“真实经营数据 + 真实驾驶舱 Human Gate”的 MVP 集成验证。项目已能在现有收益经理驾驶舱中形成：

`Signal / Event → Workflow Decision → Human Gate → Action Record → 后续 Observation`

后续仍需补齐上下文、实现 Observation 到期自动重评，并通过更多分支样本及执行后结果验证 Decision quality 与 Outcome effectiveness。
