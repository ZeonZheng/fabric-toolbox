# Parser 只读 Dry-run 结果｜2026-11-17

**时间：** 约 2026-11-17 07:40 UTC / 15:40 Asia/Singapore  
**来源：** 用户在 POC Eventhouse KQL Queryset 运行后提供的 3 张截图，以下数据由截图转录，**非完整 CSV 导出**。  
**运行变更：** 仅 SELECT / Query，**未执行 Function DDL、未更改 Update Policy**。

| 解析类型 | Dry-run Raw Rows | 经 Candidate 读取后非空的字段 |
| --- | ---: | --- |
| QueryExecutionAggregationReport | 77 | Query Duration 的 Average/Max/Min：**各 77**；Data Processing Duration 的 Average/Max/Min：**各 56** |
| QueryStreamingReport | 113 | SpoolingDiskReadingDuration、StreamingDuration：**各 104** |
| MashupEvaluationReport | 91 | AverageCommit、AverageWorkingSet、MaxCommit、MaxWorkingSet、TotalProcessorTime：**各 91** |

**已经证明：** 三个只读 KQL 候选可以执行，Raw JSON 的相关字段能够读出预期非空值，与字段名不匹配的根因判断一致。

**尚未证明：** Candidate 与目标表列名/顺序/类型相同、所有原始数值与单位相等、`DataSource` 转换正确、`TotalProcessorTime(ms)` 的 timespan 正确、事务性 Update Policy 接受新函数、历史 Parsed null 能回填、文件传输无丢失/重复。

## Schema Check 已通过（2026-11-17 16:37～16:38 UTC+8）

| CSV | 目标 | 通过 |
| --- | --- | ---: |
| [export (8)](SchemaCheck-QueryExecutionAggregationReport.csv) | Aggregation | 13/13 |
| [export (9)](SchemaCheck-QueryStreamingReport.csv) | Streaming | 9/9 |
| [export (10)](SchemaCheck-MashupEvaluationReport.csv) | Mashup | 17/17 |

**39/39 列全部一致**，列序从 0 连续到末列。此验证不证明已修改运行中 Parser。

## 数值与时间单位验证（2026-11-17 16:54 UTC+8）

[用户导出的完整 CSV](ValueCheck-ParserConversions.csv) 包含 15 项：Aggregation 6 个 Duration、Streaming 2 个 Duration、Mashup 4 个 Memory 与 1 个 timespan、Aggregation/Streaming 各 1 个 DataSource 非空率。

- Raw 记录总计 **338 条**（96 + 139 + 103，均在此次 Raw 保留窗口内）。13 个数值指标对应 **1,276 次非空转换**：全部 `ConversionLoss=0`、`ChangedByCast=0`（比较阈值 `>0.0001`）。
- DataSource 共 **235 次非空检查**，输入输出均非空；`ChangedByCast` 为 **空值（未执行值比较）**，绝不能视作语义已验证。
- Mashup 样本 `156 ms → 00:00:00.1560000`，103 条 Raw 的 `TotalProcessorTime` 毫秒回算未出现超过阈值的差异。
- `SampleRaw` 与 `SampleCandidate` 分别任意抽样，**不是配对行值**。非空率/阈值通过不等于全部数值精确无误，也未完成 Legacy vs Candidate 的 DataSource 内容等价验证。

## DataSource Legacy / Candidate 等价性验证（2026-11-17 17:11 UTC+8）

[用户导出的完整 CSV](DataSourceCheck-LegacyVsCandidate.csv) 包含三种类型：Aggregation 96/96，Streaming 139/139，Legacy 与 Candidate 的 `LegacyNonEmpty`、`CandidateNonEmpty` 均等于原始非空数，`DifferentSerialized=0`；动态类型集合均含 `array` 和 `string`。Mashup 原始 `DataSource` 103/103 非空、类型 `string`，但 `DataSourceKinds` 和 `DataSourcePaths` 独立 Raw Key 均为 0/103。此检查不包含这些 Mashup 缺失字段的语义转换。

**下一关口：** 在实际 Function 部署前，由本地 [Eventhouse MCP 只读 Preflight](../../kql/poc-parser-fix-20261117/mcp-handoff/README.md) 重新核查现场 Function、Policy、摄入状况和近期错误；仍需人工授权才可进行单个 POC DDL。


[相关测试脚本](../../kql/poc-parser-fix-20261117/README.md) · [GW-001](../../tracking/ISSUES.md) · [STATUS](../../STATUS.md)。
