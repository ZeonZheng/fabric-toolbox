# Gateway Monitoring｜2026-11-20 POC Preflight 独立评审

> **证据执行日期：** 2026-11-20 10:36–10:47（Asia/Singapore）  
> **复核日期：** 2026-11-20（Asia/Singapore）  
> **环境：** 报告所称 POC Eventhouse，运行身份依据是单节点/表结构/函数集等间接证据，**缺少 Workspace/Capacity 直接证明**。  
> **报告来源：** 本地 AI 使用 `eventhouse-remote.executeQuery`，用户上传 [脱敏报告](gateway-preflight-20261120-1036-SGT.sanitized.md)。ChatGPT 没有通过用户本地 MCP 直接运行本报告的查询。  
> **判定：** **Preflight Passed for manual patch review**, **NOT yet approved for Function DDL**。

## A. 四项证据

| 项 | 报告结果 | 复核判断 |
| --- | --- | --- |
| 01 Function Drift | 3/3 Body 严格 SHA256 一致（Length 1111 / 748 / 695） | 本次三个目标 Parser 无现场版本漂移 |
| 02 Update Policies | 3/3 Policy 均 Enabled、Source=`GatewayReports-Raw`、IsTransactional=true；全库 10 条 | 配置符合旧基线；Transactional 使错误函数存在源端摄入失败风险 |
| 03 Raw/Parsed | 三种日志 6/6 层有记录，8 类 Raw 类型继续产生数据，单节点 Heartbeat | 摄入链路在检查时仍有活动；Retention / 时间字段不同，**不证明无丢失/无重复** |
| 04 Ingestion Failures | 最近 2h、24h、保留期内 `0 observed`；FailedIngestions 14 天保留 | 命令正常执行；不排除静默 null 字段问题 |

## B. 新发现 / 需澄清

1. **GW-001 现场仍在复现：** Aggregation 6 个 Duration 字段、Streaming 2 个 Duration 字段，在旧 Parsed Table 中均 **100% null**；11 月 20 日新数据仍存在。已确认 Raw 包含括号命名的原始 Key；本分支 Candidate 使用显式 `record["...(ms)"]` 读取，之前 Dry-run + Schema + 数值和 DataSource 一致性检查通过。
2. **GW-001 扩大 Mashup 验证范围：** 报告中 `MashupEvaluationReport.TotalBytes` **2797/2797 null**，但 Raw 键存在；原版 `bag_unpack` 用 `TotalBytes:long`，Raw 字段是 string。候选使用 `tolong(record.TotalBytes)`，**合理但尚未单独量化 ConversionLoss / NonNull**。此外 `TotalRows`、`AveragePercentProcessorTime`、`MaxPercentProcessorTime`、`AverageIODataBytesPerSecond`、`MaxIODataBytesPerSecond` 需检查新候选对已有字段有无回归。
3. **GW-005 Streaming 标准 Parsed schema 丢失 3 个 Raw 字段：** `StreamingTrackingId`、`StreamingDelayDuration(ms)`、`StreamingPacketsDuration(ms)`（报告仅 1 个 envelope 抽样）；本次最小 POC Function 修复**不改 Schema**，另设缺口验收。
4. **Folder 差异：** 报告 B 节称 `parse_QueryStartReport_csv` live folder=`Parsers`，baseline=`UpdatePolicyFunctions`，但另一节称 15/15 Folder 一致。此处存在报告内部矛盾，须保留为“元数据差异待复核”，**不影响本次 3 个目标 Parser 的 Body 3/3 相符**。
5. **POC 身份：** 缺少 workspace/capacity 直接核验。**执行任何 DDL 之前必须由用户在 Eventhouse UI 明确确认目标是 POC**。
6. **性能与范围：** Candidate 采用 `mv-expand record=log | project ...` 显式转换，与旧 `bag_unpack` 实现不同；Dry-run 能执行、Schema 通过，但真实流量成本与重入性尚需 POC 观察。Patch 只作用于**新摄入**数据，历史 null 不会自动回填。

## C. 决策与控制

- **可进入人工 Review（通过）**：3/3 Function Baseline、3/3 Transactional Update Policies、Raw/Parsed 活跃、0 reported failures；加上 2026-11-17 dryrun/schema/值/DataSource 等价证据。
- **未执行任何线上修改：** 不修改 Uploaders、Eventstream、Policy、PROD，未运行三个 Candidate 或 Rollback。
- **最小风险的试点顺序：** `QueryStreamingReport` → `QueryExecutionAggregationReport` → `MashupEvaluationReport`。每次只在确认是 POC 后，人工执行一个 Function DDL，验证新摄入数据和相关故障，再决定下一个。Mashup 发布前必须先额外验证 `TotalBytes` 等非原先测试范围内的字段。
- **保留既有 Schema：** Streaming 三个 Raw-only 字段和 Mashup DataSourceKinds/Paths 另做长期保留/Schema 演进评估，不进入此次最小修复。
- **控制措施：** 变更时间、Git Commit、执行者和 Function Body SHA256 要记录；修改后确认新生效版本，验证指标非空且数值合理；检查 `.show ingestion failures`。一旦出现失败或异常立即使用本地备份的原 Body 回滚，之后再分析历史回填。
- **绝不能声称 PROD Ready**：源文件级 Upload Manifest、断点续传、去重、长时间运行、节点隔离、保留与安全审查等均未完成。

本报告的来源标签与字段级结论来自用户上传 Preflight；没有直接对用户 Eventhouse 现场进行第二次查询。参考：[Current Status](../../STATUS.md)、[GW-001/GW-005](../../tracking/ISSUES.md)。
