# POC Aggregation Parser｜第二轮单函数修复指南

> **状态：尚未修改 Aggregation Function。** Streaming 已在 POC 用新数据 32/32 验证通过。**请仅在确认是 POC 的环境操作**；本地 MCP 只能做只读检查。

## 你手动执行（三步）

1. **确认 POC / 回滚：** 进入 [Fabric](https://app.fabric.microsoft.com/) → POC Workspace → Gateway Monitoring Eventhouse → KQL Queryset，确认数据库是 POC。先打开 [Aggregation 旧版 Rollback](https://github.com/ZeonZheng/fabric-toolbox/blob/docs/gateway-monitoring-poc-20261112/monitoring/fabric-platform-monitoring/gateway/project/kql/poc-parser-fix-20261117/rollback/1-rollback-parse_QueryExecutionAggregationReport.kql)，**不要运行**。若现场 Function 与原基线不同，先停止。
2. **执行唯一写命令：** 打开 [Aggregation Candidate](https://github.com/ZeonZheng/fabric-toolbox/blob/docs/gateway-monitoring-poc-20261112/monitoring/fabric-platform-monitoring/gateway/project/kql/poc-parser-fix-20261117/candidates/1-parse_QueryExecutionAggregationReport.kql)，点击 GitHub 的 **Copy raw contents**，粘贴到已确认 POC 的 Queryset。核对对象仅是 `parse_QueryExecutionAggregationReport`，点击 **Run 一次**。如有错误，停下保留原始错误，不再继续。
3. **记录实际 UTC：** 在新查询中运行 `print RecordedAfterPatchUTC = now()`；将真实 UTC 时间和 DDL 执行成功/失败信息发给本地 AI，使用 [Aggregation Postcheck Prompt](AGGREGATION_POSTCHECK_AI_PROMPT.md) 自动执行下面 3 项只读检查。

## 验收说明

- 每次只改一个 Function；如果 Aggregation 新数据还没有出现则为 `Inconclusive`，不能认定 Pass 或 Fail。
- **6 个 Duration** 中原始 Raw 本身可能为空；之前 POC Dry-run 的 QueryDuration 三项 77/77 非空，DataProcessingDuration 三项 56/77 非空。不能把后者正常空值当 Parser 故障。应逐字段对比 Raw 值和 Parsed 新记录。
- 若出现相关 Policy failures 或明确的字段错误，停止后续修改，由用户在 POC **人工**评估执行 Rollback。旧历史 null 不自动回填。
- Streaming 已部署通过，**不要重跑** Streaming Candidate；Mashup 暂不修改，PROD 不动。
