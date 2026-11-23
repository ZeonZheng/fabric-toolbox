# Mashup Parser 手工部署记录｜2026-11-23

> **证据来源：** 用户在聊天中确认已执行 POC Candidate，提供真实 `RecordedAfterPatchUTC` 及现场 Function 导出 `export (13).csv`。只记录脱敏审计摘要；**未将环境专属 ID 或完整导出上传至 GitHub**。

## 现场观察

- 用户确认在 **Gateway Monitoring POC** 执行过 Mashup Candidate DDL；当前仍需部署后 Eventhouse Postcheck，不能直接判 P1 功能 Passed。
- **RecordedAfterPatchUTC：** `2026-11-23T02:39:15.4152749Z`（SGT `2026-11-23 10:39:15.4152749`）。这个时间是部署**后**执行 `print now()` 的结果，用作保守 Postcheck 起点。
- 用户上传的 `export (13).csv` 提供 `.show functions` 风格的 `Name/Parameters/Body/Folder/DocString` 一行记录：`parse_MashupEvaluationReport`、`()`、`UpdatePolicyFunctions`、`Function to extract data from MashupEvaluationReport`。
- 导出 Body 使用 `mv-expand record = log` 和显式 `project`，含 `record["AverageCommit(byte)"]`、`record["AverageWorkingSet(byte)"]`、`record["MaxCommit(byte)"]`、`record["MaxWorkingSet(byte)"]`、`record["TotalProcessorTime(ms)"] * 1ms`；与预备的 P1 Candidate 逻辑一致。**尚未获得 Patch 后 Policy、失败计数、新数据对账结果。**
- 三个非必需指标 TotalBytes/TotalRows/MaxIO 仍 Deferred（GW-009），它们保持原有转换逻辑，不当成修复成功。
- **仅确认用户报告的 POC Function 修改**；未授权或开展 PROD、Schema、Update Policy、Uploader、Scheduled Task 的修改。

## 当前下一步

交给本地 Eventhouse Remote MCP **只读执行** [Mashup Runbook](../../kql/poc-parser-fix-20261117/controlled-rollout/MASHUP_RUNBOOK.md) 中：

1. [10-mashup-postdeploy-summary.kql](../../kql/poc-parser-fix-20261117/controlled-rollout/10-mashup-postdeploy-summary.kql)
2. [11-mashup-postdeploy-keycheck.kql](../../kql/poc-parser-fix-20261117/controlled-rollout/11-mashup-postdeploy-keycheck.kql)
3. [12-mashup-postdeploy-ingestion-failures.kql](../../kql/poc-parser-fix-20261117/controlled-rollout/12-mashup-postdeploy-ingestion-failures.kql)

三份查询中的 `2099-01-01T00:00:00Z` 占位时间改成 **`2026-11-23T02:39:15.4152749Z`**；其余保持只读，输出聚合结果及执行错误；核实 Live Body / Policy。若无新 Raw/Parsed 行，应判为 **Inconclusive**；出现与新 Function 关联的 ingestion failure 或持续数值不匹配，进入回滚评审（回滚需用户在 POC 手工执行，且不能自动补回曾失败的批次）。

**尚未完成 Postcheck；不能标记 Mashup POC Passed 或 GW-001 Closed。**
