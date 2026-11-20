# POC Parser｜受控逐项修复 Runbook

> **状态：待用户人工执行，尚未进行 Function DDL。** 首轮只针对 `parse_QueryStreamingReport`。适用 POC Eventhouse；**不能用于 PROD**。  
> **证据：** [2026-11-20 Preflight](../../../evidence/2026-11-20/PRE_DEPLOYMENT_REVIEW.md) = 允许人工评审而非自动部署。  
> **确认日期：** 2026-11-20（Asia/Singapore）。

## Step 0 — 变更前检查（不能跳过）

1. 在 Eventhouse UI **亲自确认** Workspace 和 Eventhouse 是 POC（MCP Preflight 尚缺 Workspace/Capacity 直接证据），不是 PROD。
2. 保存当前 `parse_QueryStreamingReport` 的现场 Body（`.show functions`），与 [基线](../../../evidence/2026-11-17/DeployedFunctions.full.export.csv) 匹配。确认 [回滚脚本](../rollback/2-rollback-parse_QueryStreamingReport.kql) 已在手边。
3. 在 GitHub 当前工作分支取 [候选脚本](../candidates/2-parse_QueryStreamingReport.kql)；复核只改一个 Function，未改目标表及 Update Policy。提交/记录 Commit SHA 和**实际执行 UTC**。不要让 MCP 自动执行写操作。
4. 执行前保留 [Schema Check](../schema/2-schema-check-parse_QueryStreamingReport.kql) 与 [Read-only Dry-run](../dryrun/2-dryrun-parse_QueryStreamingReport.kql) 的既有通过证据；如运行环境或目标表改变，需重新测试。

## Step 1 — 受控 POC 部署（需用户人工批准）

仅由用户在 **确认的 POC KQL Queryset** 手动运行 **一次**：
[2-parse_QueryStreamingReport.kql](../candidates/2-parse_QueryStreamingReport.kql)。

预期命令为 **`.create-or-alter function`**，不会主动改表或 Update Policy，但有事务性 Policy 运行风险。不要在测试期间同时改 Aggregation/Mashup，也不要更新 Jumpstart。

执行后：
- 立即用 `.show functions | where Name == "parse_QueryStreamingReport"` 检查生效 Body。
- 重新确认 Update Policy 仍 Enabled、Source=`GatewayReports-Raw`、IsTransactional=true。
- 记录成功/失败及准确 UTC 时间，保持 Snapshot 和原始文件。
- 有错误立刻停止，按 Step 3 回滚。

## Step 2 — 只读回归（必须有修复后**新摄入**的样本）

1. 将 [01-streaming-postdeploy-summary.kql](01-streaming-postdeploy-summary.kql) 内的 `PatchUtc` 替换为**实际执行 UTC 时间**后运行，保存 CSV。**如果无新数据：状态为 Inconclusive；不能判定部署成功。**
2. 运行 [02-streaming-postdeploy-keycheck.kql](02-streaming-postdeploy-keycheck.kql)（同一 `PatchUtc`），检查 Raw 实际字段和 Parsed 的修复后指标，按 RequestId + QueryTrackingId 做样本关联。该关联**不是源文件级全量对账**。
3. 运行 [03-streaming-ingestion-failures.kql](03-streaming-ingestion-failures.kql)，检查是否产生 Update Policy failures；同时看 Uploader / Eventstream 是否仍持续。
4. 接受条件：新摄入 Raw 有可解析指标；相应 Parsed 的时长字段出现非 null 且对应数值合理；已观察到至少一批**部署后新事件**；没有新出现的相关 ingestion failures。允许 Raw 中原本空值继续对应 null。
5. **旧 Parsed 行保持 null 不构成修复失败**，因为修改 Function 不会自动回填历史。

## Step 3 — 回滚条件与动作（人工）

任一情况停止：DDL 报错、目标 Schema 差异、相关摄入失败、修复后有值的 Raw 却持续得到 null/错误 Parsed、非预期字段/类型回归。人工重新核实 POC 后执行 [2-rollback-parse_QueryStreamingReport.kql](../rollback/2-rollback-parse_QueryStreamingReport.kql)，然后复核现场 Body 和异常是否停止。

**重要：** 回滚函数只恢复今后数据的解析代码，不自动恢复已失败或未摄入的事件；Eventstream 重试/缓存、Raw 短保留期、历史 backfill 均需单独检查。

## Step 4 — 后续顺序（不在本轮同时执行）

`QueryExecutionAggregationReport` → `MashupEvaluationReport`。Mashup 部署前必须先执行 [额外数值检查](../value-checks/3-mashup-additional-metrics.kql)，尤其是 `TotalBytes`（在 2026-11-20 的旧 Parsed 全部 null）。

**不要把此次 POC 的修复通过等同于 PROD Ready。** 端到端文件/记录审计、重试幂等和 Raw 所有字段长留存仍未完成。
