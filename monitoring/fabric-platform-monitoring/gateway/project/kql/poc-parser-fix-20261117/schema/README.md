# POC Parser｜Schema 兼容性检查（只读）

> **本目录的三个 Query 不修改 Eventhouse。** 不通过前禁止运行 `../candidates/` 中的 Function DDL。

**目的：** 通过 `getschema` 对比 Candidate() 与现有 Parsed Table 的列定义，包括 `ColumnOrdinal`（顺序）、`ColumnName`（字段名）、`ColumnType`（类型）。脚本使用 `join kind=fullouter`，因此某一侧缺少字段也能显示出来。

| 类型 | 查询 |
| --- | --- |
| QueryExecutionAggregationReport | [Schema Check 1](1-schema-check-parse_QueryExecutionAggregationReport.kql) |
| QueryStreamingReport | [Schema Check 2](2-schema-check-parse_QueryStreamingReport.kql) |
| MashupEvaluationReport | [Schema Check 3](3-schema-check-parse_MashupEvaluationReport.kql) |

**预期：** 每行 `IsMatch = true`，且字段数、列顺序与现有目标表一致。若有 `false`、null、缺行或错误，**不要部署**，应先修复 Candidate。

注意这只是类型兼容检查。仍须用 Raw 样本比较实际数值、DataSource 解析和 `TotalProcessorTime(ms)` → timespan 的单位；只读 Dry-run 通过不代表这些自动正确。

执行证据保存到：[DRYRUN_RESULTS.md](../../../evidence/2026-11-17/DRYRUN_RESULTS.md)。当前下一步以 [STATUS](../../../STATUS.md) 为准。

官方文档：[Kusto Update Policy](https://learn.microsoft.com/en-us/kusto/management/update-policy)。
