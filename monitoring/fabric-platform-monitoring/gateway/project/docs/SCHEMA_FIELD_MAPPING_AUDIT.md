# Gateway Raw → Eventhouse Parsed｜字段映射审计

> **审计日期：** 2026-11-17；**环境：** POC 单节点；**改动状态：** 仅检查及只读 Dry-run，**未修改 Eventhouse Function**。

已取得全部 15 个实际部署 Function Body 和 142 行数据库 Schema；逐一对照工作分支的上游 `DatabaseSchema.kql`，15/15 Function Body 在忽略空白差异后**一致**。本次发现的映射问题因此属于当前仓库定义本身，而非部署版本漂移。

证据：[15 Function Bodies](../evidence/2026-11-17/DeployedFunctions.full.export.csv) · [142 行 Schema](../evidence/2026-11-17/DeployedTableSchema.sanitized.csv) · [106 组 Raw 字段统计](../evidence/2026-11-17/RawFieldInventory.export.csv) · [Eventhouse Overview](../evidence/2026-11-17/EventhouseOverview.MCP.sanitized.md)。

## 一、类型与字段覆盖

| logType | Raw 已观察字段 | Parsed 表字段 | 关键差异 |
| --- | ---: | ---: | --- |
| ConcurrentOperationAggregationReport | 7 | 7 | 字段名一致；尚未逐值比对 |
| GatewayNodeInfo | 16 | 17 | 目标表额外使用 Envelope 中的 `logDate` |
| MashupEvaluationReport | 17 | 17 | 多个 Byte/Time 字段名不匹配；`DataSourceKinds/Paths` 无相同 Raw Key；`DataSource` / `RequestId` 未按原样保留 |
| OpenConnectionReport | 9 | 9 | `OpenConnectionDuration(ms)` 被 Parser 正确读取，按目标位置输出；尚未发现该项缺陷 |
| QueryExecutionAggregationReport | 13 | 13 | **6 个 Duration 字段**的 Raw Key 是 `(ms)`，Parser 却读取 `_ms_` |
| QueryExecutionReport | 17 | 15 | 时长字段正确读取并按目标位置输出；Raw 独有的 `DataReadingDuration(ms)` / `DataSerializationDuration(ms)` 未保存 |
| QueryStartReport | 8 | 18 | 从 QueryText / EvaluationContext 派生多个字段；展开后行数可能增加 |
| QueryStreamingReport | 12 | 9 | **2 个 Duration 字段**的 Raw Key 错配；Raw 的 `StreamingDelayDuration(ms)`、`StreamingPacketsDuration(ms)`、`StreamingTrackingId` 未保存 |
| SystemCounterAggregationReport | 7 | 7 | 字段名一致；尚未逐值比对 |
| CopyExecutionReport | 当前 Raw 样本中没有 | 7 | 目标表与 Policy 存在，但没有源样本，不能验证 |

## 二、已经用 Raw 与 Parsed 实测确认的问题

| 类型 | Raw 原始字段 | Parsed 目标字段 | 实测 |
| --- | --- | --- | --- |
| QueryExecutionAggregationReport | `AverageQueryExecutionDuration(ms)` | `AverageQueryExecutionDuration_ms_` | Raw 73/73 非空，Parsed 0/2935 非空 |
| QueryStreamingReport | `StreamingDuration(ms)` | `StreamingDuration_ms_` | Raw 81/90 非空，Parsed 0/2346 非空 |
| MashupEvaluationReport | `AverageCommit(byte)` | `AverageCommit byte` | Raw 77/77 非空，Parsed 0/1892 非空 |

另需验证：聚合类型剩余 5 个时长字段、Streaming 的 `SpoolingDiskReadingDuration(ms)`、Mashup 的 WorkingSet / MaxCommit / `TotalProcessorTime(ms)`。

**重要区分：** Parsed **列名**可以不同于 Raw JSON **Key**，前提是 Parser 正确读取 Raw Key 并按目标列顺序/类型投影。例如 `QueryExecutionReport` 的 `QueryExecutionDuration(ms)` 已被正确读取，不能把所有 `_ms_` 列名都误判为错误。

## 三、只读修复候选验证结果

2026-11-17 约 15:40（UTC+8）：

| Candidate | Raw Rows | 非空结果 |
| --- | ---: | --- |
| Aggregation | 77 | 3 个 Query Duration 各 77；3 个 Processing Duration 各 56 |
| Streaming | 113 | 两个 Duration 各 104 |
| Mashup | 91 | Memory / Processor 共 5 个指标各 91 |

这证明读取 Raw Key 的方向正确；**没有**证明列序、类型、数值单位、DataSource、Update Policy 运行、历史回填正确。

## 四、设计和剩余风险

1. 必须决定 PROD 需要**仅用于报表的标准化 Schema**，还是需要长期保存 Gateway CSV 的**所有原始字段**。目前 Parsed 不保留全部字段，而 Raw 仅保留 1 天。
2. `MashupEvaluationReport.DataSourceKinds/Paths` 不存在对应 Raw Key，不能未经数据语义核实就把 `DataSource` 强行拆成它们。
3. `TotalProcessorTime(ms)` 要明确从数值毫秒转换为目标 `timespan`，含空值和单位测试。
4. `QueryStartReport` 会按 EvaluationContext 派生多行，不能一概要求 Raw/Parsed 记录数一比一。
5. MCP Overview 显示 12 个物理表、15 函数、10 个启用的事务性 Update Policy、2 个健康 MV。这是**瞬时结构快照**，不证明历史数据完整。

## 五、验收前置条件

- 运行 [3 个只读 Schema Check](../kql/poc-parser-fix-20261117/schema/README.md)，逐列匹配名称、顺序、类型。
- 对照 Raw 原值与 Candidate 数值、单位、DataSource；如果有偏差先修 KQL。
- 然后才考虑 POC `.create-or-alter function`，逐个上线/观察，保留 [Rollback](../kql/poc-parser-fix-20261117/README.md)。现有 Parsed 历史 null 不会自动修复。

对问题的完整复现见 [PARSER_SCHEMA_AUDIT.md](PARSER_SCHEMA_AUDIT.md)。对应 Issue：[GW-001](../tracking/ISSUES.md)。


## 2026-11-17 17:11（UTC+8）— DataSource 语义验证补充

- Aggregation 96 条 Raw、Streaming 139 条 Raw：Legacy Parser 与 Candidate 的 DataSource 规范化值 **235/235 内容完全一致**（`DifferentSerialized=0`）；动态类型分别观察到 `array` 和 `string`。不是仅仅非空率一致。
- Mashup 103 条 Raw 中 `DataSource` 字段 103/103 非空，值的 JSON 类型为 `string`；`DataSourceKinds` 和 `DataSourcePaths` 均 **0/103** 出现在原始 JSON Key 中。目标表仍保留这些字段但在本次 Candidate 中不推断值，长期数据语义修复属于 [GW-005](../tracking/ISSUES.md)。
- 只读检查结果：[CSV](../evidence/2026-11-17/DataSourceCheck-LegacyVsCandidate.csv)。**三个候选的 Dry-run、Schema、数值和适用的 DataSource 等价检查已通过；现场 Function 仍未修改。** 接下来应确认当前部署/Policy 没有漂移，再安排受控 POC 修改。


## 2026-11-23：Mashup 数组与小数新证据

- [Nov23 Shape Follow-up](../evidence/2026-11-23/MASHUP_SHAPE_FOLLOWUP_REVIEW.md)：TotalBytes 44/44、TotalRows 35/35 数组内元素可转 long；同现 6/6 数组等长，不代表元素业务上可求和/按 index 匹配。MaxIO 12/14 含小数，目标 long 不可保真。
- 之前 Mashup `17/17 Schema Check` 只验证 Candidate 投影列结构，**不保证** TotalBytes/Rows/MaxIO 的数值语义；候选 DDL 仍被禁止，现场 Function 仍旧版。
- 经 GitHub fork 的 Fabric Report、Semantic Model TMDL、KQL Dashboard 可读源码检查，暂未发现这三个字段的直接引用；这不证明已部署报表/二进制 PBIT 或外部消费者不依赖。
- 后续 [业务范围与技术设计评审](MASHUP_DESIGN_OPTIONS.md) 替代反复的 Raw 数值统计；是否将这三字段纳入 P1 必须是明确的范围决定。
