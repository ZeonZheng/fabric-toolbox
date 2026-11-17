# Gateway Parser 缺陷｜发现、根因与复现证据

> 2026-11-17 POC 记录。**已确认字段读取错误；修复尚未部署。** 当前状态与下一步统一以 [STATUS](../STATUS.md) 为准。

## 1. 源端与 Raw 快照

2026-11-16 02:59 UTC，`GatewayReports-Raw` 查询观察到：**9 种 logType / 647 条 JSON Message / 3693 条 CSV Record**。其中 8 类 CSV Report 及独立生成的 `GatewayNodeInfo`。字段统计共 106 个 Type/Field 组合；同类型每条样本都含有相同字段，但空值比例各不相同。**这是当时的 Raw 快照，不是全量无遗漏证明。**

## 2. Raw 值存在，但 Parsed 指标全为 null

Parsed 查询（约 2026-11-17 06:24 UTC）：

| Parsed Table | 总 Rows | 指标有效值 | 检查字段 |
| --- | ---: | ---: | --- |
| QueryExecutionReport | 4595 | 4595 | `QueryExecutionDuration_ms_`（对照正常） |
| QueryExecutionAggregationReport | 2935 | **0** | `AverageQueryExecutionDuration_ms_` |
| QueryStreamingReport | 2346 | **0** | `StreamingDuration_ms_` |
| MashupEvaluationReport | 1892 | **0** | `AverageCommit byte` |

Raw 对应查询（约 06:31 UTC，**不是上表同一历史窗口**）：

| LogType | Raw 字段 | Raw 条数 | 非空 | 示例 |
| --- | --- | ---: | ---: | --- |
| QueryExecutionAggregationReport | `AverageQueryExecutionDuration(ms)` | 73 | 73 | `266536.00` |
| QueryStreamingReport | `StreamingDuration(ms)` | 90 | 81 | `38` |
| MashupEvaluationReport | `AverageCommit(byte)` | 77 | 77 | `261075800.00` |

**根因：** 这三个 Parser 在 `evaluate bag_unpack(log) : (...)` 使用的列名不是当前 Gateway CSV 真正送入 Raw JSON 的字段 Key，导致解析为空值，而不是 Eventstream 随机删掉了这些字段。实际部署 15 个 Function Body 已与 GitHub Schema 原文对照，15/15 一致。

源文件：[上游 Schema 备份](../evidence/2026-11-17/DatabaseSchema.source.kql) · [实际 Functions 导出](../evidence/2026-11-17/DeployedFunctions.full.export.csv) · [Update Policy 快照](../evidence/2026-11-17/DeployedUpdatePolicies.sanitized.json)。

## 3. Update Policy 与 VM 文件观察

- 10 个目标表的 Update Policy 启用，源为 `GatewayReports-Raw`，`IsTransactional=true`。
- 当时 `.show ingestion failures | where OriginatesFromUpdatePolicy == true` 的 24 小时查询为空，但**不能据此认定字段值正确**。
- VM 目录观察到 42 个 Report 文件（40 History 合并文件 + 2 Source），8 种 Report 类型。History 不能用于证明每个原始文件的成功入库。
- `GatewayReports-Raw` 默认保留 1 天，Parsed 多数保留 3650 天；修复 Parser **不会自动更新历史 Parsed null**。

## 4. 验证结果和处置边界

2026-11-17 约 15:40 UTC+8，三组只读 Candidate Dry-run **均成功执行并恢复非空指标**，详情见 [Dry-run 证据](../evidence/2026-11-17/DRYRUN_RESULTS.md)。

后续验收要求：验证 Candidate 与目标表 Schema 完整一致（列名、顺序、类型）；逐项比较 Raw 与 Candidate 的数值及 `TotalProcessorTime(ms)` / `DataSource` 的转换；再审查对事务性 Update Policy 的影响，**只在 POC 中逐个变更**，验证新数据并保留回滚。

其他字段风险和是否长期完整保存原始列，统一记录在 [全字段映射审计](SCHEMA_FIELD_MAPPING_AUDIT.md)。生产化不得仅凭 3 个指标恢复非空就放行。

### 最小复现 KQL（只读）

```kusto
['GatewayReports-Raw']
| where logType in ("QueryExecutionAggregationReport", "QueryStreamingReport", "MashupEvaluationReport")
| mv-expand record = log
| extend RawValue = case(
    logType == "QueryExecutionAggregationReport", todouble(record["AverageQueryExecutionDuration(ms)"]),
    logType == "QueryStreamingReport", todouble(record["StreamingDuration(ms)"]),
    logType == "MashupEvaluationReport", todouble(record["AverageCommit(byte)"]),
    real(null))
| summarize Rows=count(), NonNull=countif(isnotnull(RawValue)), Example=take_anyif(RawValue,isnotnull(RawValue)) by logType
```

**注意：** 不要将完整 Raw 日志、包含 QueryText 的 CSV、Secrets 或环境敏感标识提交到 GitHub。需要回填时单独制定去重、数据保留和历史恢复方案。

关联：[GW-001](../tracking/ISSUES.md)。
