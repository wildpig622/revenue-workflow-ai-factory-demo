# 酒店收益策略中枢｜Revenue Strategy Hub

酒店收益策略中枢是一套面向酒店收益管理的 AI 工作流与策略沉淀 MVP。它将真实业务 Case 转化为可复核、可追溯、可验证的 Workflow 与策略资产，并通过质量门禁和 Human Gate 保留业务责任边界。

[打开公开演示](https://wildpig622.github.io/revenue-workflow-ai-factory-demo/) · [查看比赛导览](https://wildpig622.github.io/revenue-workflow-ai-factory-demo/demo.html)

## 核心闭环

`Revenue Case → Workflow / L1–L3 Knowledge → Validator → Human Gate → Action Record → Observation`

公开演示包括：

- Workflow Analyzer 与人工业务复核；
- 候选策略集及来源追溯；
- L1–L3 定价知识结构；
- Structure、Reference、Graph、Branch、Safety、Human Judgment 六类 Validator；
- 业务条件不完整时 `BLOCK`，补齐后重新送检；
- PASS / WARNING / BLOCK 统一 Validation Report。

## 真实驾驶舱 MVP 集成验证

2026-08-26，`CASE_B_CANCELLATION_V1` 已使用现有收益经理驾驶舱完成真实经营数据的端到端 MVP 集成验证：

`order_log 真实取消事件 → Context Builder → run_case_b() → Decision → decision_log + todo_items → RiskAlert → Human Gate → strategy_case`

本次验证事实：

- 从当日 `order_log` 读取 3 条真实取消事件，3/3 均成功进入 Case B Runtime；
- Decision 均成功写入 `decision_log` 和 `todo_items`；
- RiskAlert 能展示完整五要素 Decision；
- Human Gate“确认执行”已实测成功；
- 确认后 `strategy_case` 正确生成并与 `todo_items` 关联；
- Runtime 去重链路已生效。

当前 3 条真实样本均命中：

```text
reason_code: OBSERVATION_WINDOW_ACTIVE
action: OBSERVE
data_quality: DEGRADED
```

因此，目前能够确认的是：

- ✅ Integration / end-to-end workflow 已验证；
- ⚠️ Decision quality 仍在验证中；
- ⚠️ `ops_snapshot` 上下文字段仍在补齐；
- ⚠️ Observation 到期自动重评尚未完成；
- ⚠️ Outcome effectiveness 尚不能宣称已验证。

完整状态请查看 [Runtime Readiness](RUNTIME_READINESS.md)，脱敏验证摘要请查看 [Integration Evidence](evidence/CASE_B_CANCELLATION_V1_INTEGRATION_2026-08-26.md)。

## 产品定位

本项目不是自动调价器，也不以 AI 替代收益经理。它已作为现有收益经理驾驶舱的决策层接入真实工作流，当前形成：

`Signal / Event → Workflow Decision → Human Gate → Action Record → 后续 Observation`

Runtime 仍严格限定于 `CASE_B_CANCELLATION_V1`。通用 Runtime、多 Case 调度、Observation 到期自动重评和 Outcome effectiveness 仍属于后续工作。

## 数据与安全边界

- 公开仓库和演示不包含真实订单、门店、人员或经营明细；
- 真实驾驶舱验证仅以脱敏汇总形式记录；
- PASS 只表示 Workflow 通过当前质量门禁，不表示策略最优或收益效果已验证；
- Runtime 不自动确认、不自动调价，所有动作仍经过 Human Gate。

## 文档

- [CASE_B_CANCELLATION_V1 Runtime MVP](CASE_B_RUNTIME_MVP.md)
- [Runtime Readiness](RUNTIME_READINESS.md)
- [2026-08-26 Integration Evidence](evidence/CASE_B_CANCELLATION_V1_INTEGRATION_2026-08-26.md)
