# Mashup 12 非目标列 Legacy–Candidate 兼容性 Gate｜2026-11-23

> **证据来源：** 用户在聊天中转交的本地 Eventhouse Remote MCP 执行结果（POC `Gateway Monitoring`）。结果为**用户提供的运行摘要**，非本助手直接调用 MCP 的查询输出；查询原文据用户确认未修改、没有 DDL。

## 本次只读结果

查询：[7-mashup-legacy-candidate-compatibility.kql](../../kql/poc-parser-fix-20261117/value-checks/7-mashup-legacy-candidate-compatibility.kql)；GitHub 工作分支 `docs/gateway-monitoring-poc-20261112`。

| LegacyRows | CandidateRows | DifferingSignatureGroups | UnmatchedRows | Result | 错误 |
| ---: | ---: | ---: | ---: | --- | --- |
| 36 | 36 | 0 | 0 | **PASS** | 无 |

这意味着在 Raw 1 天保留窗口内的 36 行样本，旧 `bag_unpack` 与新显式投影对**12 个非目标列**的组合值与重复计数均一致。此前 [最小 Precheck](MASHUP_MINIMAL_PRECHECK_REVIEW.md) 则验证五个 Memory/Time 目标字段：四项 33/33、ProcessorTime 36/36，无观察到转换损失，Schema 17/17。

## 独立判断

**已满足当前 POC 单函数手工部署前的只读验证 Gate**；可进入**受控手工部署**，但**不是已经部署，更不是功能性通过**。新 Function 的函数体与 2026-11-20 已标为 Blocked 的历史 Candidate 相同：改变的是 ADR-009 的验收范围以及新补充的兼容性证据。

### 限制

- 样本只有 36 行，且只覆盖 Raw 当前保留窗口，不能保证将来所有记录/异常载荷都兼容。
- 本次使用哈希后的组合字段及**多重计数**比较；结果为 0 mismatch，但不能当成上游日志 Collector Exactly-once 的证据。
- `TotalBytes`、`TotalRows`、`MaxIODataBytesPerSecond` 仍按 [GW-009](../../tracking/ISSUES.md) 延期；原有 null 不是已经修复。
- 实际部署前还必须确认正确 POC Workspace/Database、Live Function 与本次基线无漂移、事务性 Policy/回滚可用。
- 部署后的 Raw/Parsed 新数据、关键五项数值、Policy failures 必须重新验收；无新数据 = Inconclusive。
- **目前尚未执行 Function DDL，未更改 Table Schema / Update Policy / Uploader / PROD。**

## 下一动作

按 [Mashup 手工部署 Runbook](../../kql/poc-parser-fix-20261117/controlled-rollout/MASHUP_RUNBOOK.md) 执行；本地 AI 只负责 MCP 只读 postcheck，不自动执行 DDL 或回滚。
