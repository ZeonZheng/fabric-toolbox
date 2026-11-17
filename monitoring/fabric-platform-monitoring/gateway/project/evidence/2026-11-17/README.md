# Eventhouse POC 部署基线｜2026-11-17

> **环境：** 单 Cluster / Node POC。来源：用户在 Eventhouse KQL Queryset 及 Remote MCP 的实测导出，加上 GitHub fork 源码。  
> **用途：** 修复前可追溯的 Schema/Function 定义与结构快照；**不是整个 Eventhouse 的物理备份，也不是已部署的修复补丁。**

## 文件清单与来源

| 文件 | 来源 | 具体用途及限制 |
| --- | --- | --- |
| [DatabaseSchema.source.kql](DatabaseSchema.source.kql) | fork 原始 `src/Gateway/Gateway Monitoring.KQLDatabase/DatabaseSchema.kql`，Blob SHA `10e94c1ec71102283f6f93c9ff05da705db77ab8` | **上游源码快照**，不能当作完整的运行时 Eventhouse 导出 |
| [DeployedFunctions.full.export.csv](DeployedFunctions.full.export.csv) | KQL `.show functions` 用户导出 (6) | **实际部署的 15 个 Function Body**，与当时 GitHub 源码逐一对照 15/15 一致 |
| [DeployedParserFunctions.export.csv](DeployedParserFunctions.export.csv) | KQL 用户导出 (5) | 早期单独保存的 3 个 Parser Body；属于上一个更小范围的证据 |
| [DeployedTableSchema.sanitized.csv](DeployedTableSchema.sanitized.csv) | `.show database schema` 用户导出 (7) | **142 行列结构**，包含列名、类型、顺序；已移除 DatabaseName |
| [DeployedUpdatePolicies.sanitized.json](DeployedUpdatePolicies.sanitized.json) | `.show table * policy update` 用户导出 (4) | 14 个表策略记录，去除实体中数据库标识；10 个解析目标有启用策略 |
| [RawFieldInventory.export.csv](RawFieldInventory.export.csv) | 用户 Raw JSON 字段统计导出 (3) | 106 个 logType/字段组合的存在性/非空统计；**不包含实际 Raw 行** |
| [EventhouseOverview.MCP.sanitized.json](EventhouseOverview.MCP.sanitized.json) | 用户通过 Eventhouse Remote MCP 生成的结构 Overview | **12 Tables、15 Functions、10 Update Policies、2 Materialized Views** 以及 Retention、库存行数；去除部分环境标识 |
| [EventhouseOverview.MCP.sanitized.md](EventhouseOverview.MCP.sanitized.md) | 同次 MCP Overview 的 Markdown 说明 | 中文结构总览、数据流、表/列、策略及问题摘要；适合人和 AI 快速查看 |
| [DRYRUN_RESULTS.md](DRYRUN_RESULTS.md) | 用户 KQL 截图转录 | 3 个 Parser Dry-run 的结果统计，未执行部署 |
| [ValueCheck-ParserConversions.csv](ValueCheck-ParserConversions.csv) | 用户导出 (11)，2026-11-17 16:54（UTC+8） | 15 个验证项、13 数值 + 2 DataSource 非空检查；全部 `ConversionLoss=0`，13 个数值项 `ChangedByCast=0`；DataSource 内容仍需进一步验证 |
| [SchemaCheck-QueryExecutionAggregationReport.csv](SchemaCheck-QueryExecutionAggregationReport.csv) | 用户导出 (8)，2026-11-17 16:37（UTC+8） | 13/13 列 IsMatch=true |
| [SchemaCheck-QueryStreamingReport.csv](SchemaCheck-QueryStreamingReport.csv) | 用户导出 (9)，2026-11-17 16:38（UTC+8） | 9/9 列 IsMatch=true |
| [SchemaCheck-MashupEvaluationReport.csv](SchemaCheck-MashupEvaluationReport.csv) | 用户导出 (10)，2026-11-17 16:38（UTC+8） | 17/17 列 IsMatch=true |

## 注意事项

- MCP Overview 采集于 **2026-11-17 14:56 UTC+8**，当时共 156,878 行（各表不同保留期的库存统计，不是 156,878 条去重后的原始事件）。
- `GatewayReports-Raw` 在快照中保留期 **1 天**；Heartbeat 7 天，许多 Parsed 表 3650 天。物理清理可能不是瞬时完成。
- 此快照不包括可恢复数据库存储、认证、所有实时连接设置、Secrets 和原始 Gateway 日志，因此**只能用于配置对照及变更前备份**。
- 原始导出的 SQL/KQL 字段名称、Function Body 保持原文，避免备份被翻译破坏；Markdown 说明采用中文。
- GitHub 分支 `docs/gateway-monitoring-poc-20261112` 是我们工程资产的 Source of Truth；如后续 Eventhouse Function 修改，必须记录变更及回滚版本。

参考：[字段对应审计](../../docs/SCHEMA_FIELD_MAPPING_AUDIT.md) · [Parser 缺陷](../../docs/PARSER_SCHEMA_AUDIT.md) · [当前状态](../../STATUS.md)。
