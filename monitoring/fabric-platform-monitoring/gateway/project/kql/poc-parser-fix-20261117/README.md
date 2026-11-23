# POC Parser 修复工作区（2026-11-17）

> **2026-11-20 更新：Streaming 和 Aggregation 均已在 POC 手工修复并通过新数据功能性回归；仅 Mashup 候选尚未部署。** 本目录为 POC 验证资产，不允许直接部署 PROD。

## 范围与安全边界

- 不修改 Gateway Uploader、Task Scheduler、Eventstream、现有 Parsed Table 或 Update Policy。
- 候选修改仅涉及 3 个现有 Function：`parse_QueryExecutionAggregationReport`、`parse_QueryStreamingReport`、`parse_MashupEvaluationReport`。
- 第一轮尽量保持目标 Schema 的列顺序和类型，显式读取 Raw JSON 中的 `(ms)` / `(byte)` 字段，再转换为目标名称。
- `MashupEvaluationReport.DataSourceKinds` 和 `DataSourcePaths` 无对应原始 Key，暂不从 `DataSource` **捏造含义**；这次修复不代表长期完整保留全部 CSV 字段。

## 子目录：按执行风险排列

| 子目录 | 是否改写 Eventhouse | 用途 |
| --- | --- | --- |
| [dryrun/](dryrun/) | **否（只读）** | `let Candidate = () {...}`，用非空计数初步验证 |
| [schema/](schema/README.md) | **否（只读）** | **已通过 3/3，39/39 列匹配** |
| [value-checks/](value-checks/README.md) | **否（只读）** | 数值/单位检查已通过；[DataSource 内容等价性查询](value-checks/DATASOURCE_README.md) 已通过（235/235） |
| [candidates/](candidates/) | **是（变更 Function）** | Streaming **已部署通过，勿重复运行**；Aggregation **已部署且 Passed**；Mashup 需额外数值只读 Precheck，尚未部署 |
| [rollback/](rollback/) | **是（回滚 Function）** | 以实际部署导出的旧 Body 恢复，必须确认现场版本后使用 |

备份：[15 个实际 Function Body](../../evidence/2026-11-17/DeployedFunctions.full.export.csv) · [142 行 Schema](../../evidence/2026-11-17/DeployedTableSchema.sanitized.csv) · [字段审计](../../docs/SCHEMA_FIELD_MAPPING_AUDIT.md)。

## 验证步骤与当前证据

1. 已完成：3 个 `dryrun/` 查询成功，Aggregation 77 Rows、Streaming 113 Rows、Mashup 91 Rows，具体非空率见 [证据](../../evidence/2026-11-17/DRYRUN_RESULTS.md)。
2. **已完成：** [3 个 Schema Check](schema/README.md) 全部通过，39/39 列匹配，已归档 CSV (8)/(9)/(10)。
3. **数值与单位已检查：** 用户 KQL 共 15 项，13 个数值指标 `ConversionLoss=0` 且 `ChangedByCast=0`（阈值 0.0001）；**DataSource Legacy / Candidate 等价已通过** [只读对照](value-checks/DATASOURCE_README.md)（Aggregation 96/96、Streaming 139/139；Mashup Kinds/Paths Key 不存在）。
4. 通过后，先用本地 MCP 执行 [只读上线前 Preflight](mcp-handoff/README.md)，确认当前部署版本和更新策略；再人工评审 `candidates/`：在 **POC** 中每次只更新一个 Parser，记录精确部署时间、Git Commit SHA、对应 Rollback。
5. 部署后检查新摄入记录、Update Policy、Eventstream 指标和 Ingestion Failures；旧 Parsed null **不会自动修复**，历史回填需单独设计。
6. 全部验证完成后，再把认可的修复应用到 GitHub 的正式 Schema 源码定义；不要未经评估就运行 Jumpstart `update_existing=True`。

## 特别风险

当前 Update Policies 使用 `IsTransactional=true`。如果函数输出与目标表 Schema 不匹配，可能影响 Raw 摄入；仅 Dry-run 的非空验证不足以保证线上数据完整性。Kusto 源码里的 DataSource 标准化表达式已调整输入对象为 `record.DataSource`，需要逐值核对结构。

Microsoft 参考：[Update Policy](https://learn.microsoft.com/en-us/kusto/management/update-policy)。

**唯一最新进度：[STATUS.md](../../STATUS.md)**。避免在脚本说明里重复维护项目当前阶段。


## 2026-11-20：MCP Preflight + 受控修复交接

- [Preflight 评审与原始脱敏报告](../../evidence/2026-11-20/README.md)：4/4 通过、3/3 Function SHA256 一致，允许进入人工 Review，**不授权自动部署**。
- [受控 Streaming 首轮 Runbook](controlled-rollout/README.md)：POC UI 确认、单 Function DDL、部署后只读新记录/失败监测、回滚。
- [Mashup 新发现补充只读检查](value-checks/3-mashup-additional-metrics.kql)：`TotalBytes` 等此前未全部覆盖指标，**尚未执行**。

**实际运行的 POC Functions 仍是旧版；历史 null 未修复。** 不要并行执行三个 Candidate，PROD 不受本轮变更影响。


## 2026-11-20 Streaming POC 首轮修复验收

- 实际现场 `parse_QueryStreamingReport` 与 Candidate Body 逐字节一致；修改后 **32/32 Parsed 两个 Duration 都非空**，Raw 匹配与数值对比 **32/32 一致、0 mismatch**，未观察到 Update Policy 失败。
- 旧 Parsed 2675 行仍 null，历史不自动修复；功能验证窗口约 25 分钟，不等于长时间稳定性。
- 已将这一个已通过 POC 验证的 Function 同步到 fork 正式 `src/Gateway/Gateway Monitoring.KQLDatabase/DatabaseSchema.kql`，避免下次部署从旧源码覆盖修复。保留原版 `evidence/2026-11-17/DatabaseSchema.source.kql`。
- [完整证据](../../evidence/2026-11-20/STREAMING_FOLLOWUP_REVIEW.md) · [下一个 Aggregation Runbook](controlled-rollout/AGGREGATION_RUNBOOK.md)。


## 2026-11-20 Aggregation 功能性回归与 Mashup 门槛

- Aggregation 2026-11-20 15:47–15:51 Follow-up 已通过：Raw 2 Envelope / 7 行、Parsed 7 行；六列 Raw/Parsed 非空计数相同，复合 Key 2/2 命中、非空值 0 mismatch。DataProcessing 三项仅 1/7 有值是 Raw 自身空值，属于预期。现场 Body 与 Candidate 一致、Policy 正常。**不是 Exactly-once 或 7/7 逐行匹配保证**。
- 已将该已验证 Function 同步 fork 正式 `DatabaseSchema.kql`，与早先 Streaming 同步一起保留；旧版源 [2026-11-17 baseline](../../evidence/2026-11-17/DatabaseSchema.source.kql)。
- Mashup 本轮**未运行任何 DDL**，先让本地 MCP 按 [Mashup Precheck Prompt](value-checks/MASHUP_PRECHECK_AI_PROMPT.md) 运行 [六项额外字段检查](value-checks/3-mashup-additional-metrics.kql)，根据结果再决定单函数手动修复。


## Mashup Blocked（2026-11-20 Precheck）

- 本轮只读证实 `TotalBytes` 243/243、`TotalRows` 34/34 的数组文本及 `MaxIODataBytesPerSecond` 127/127 的小数文本在现行 Candidate 中均转换为 null。**原 Parsed 也已有同样缺陷，不是已部署 Candidate 的回归。**
- 两个数组的多元素语义未知；`MaxIO` 目标表是 long 但原始有小数。保留原 Candidate **仅作缺陷复现/审计，不得执行**，也不改 fork 正式 KQL Schema。详细见 [独立评审](../../evidence/2026-11-20/MASHUP_PRECHECK_REVIEW.md) 与 [只读诊断](value-checks/MASHUP_SHAPE_README.md)。


## 2026-11-23 Mashup Shape Follow-up

- 04/05/06 已在 POC 通过（04/06 KQL 的 `floor(x)` 语法错误已在 GitHub 只读版本中修为 `floor(x,1.0)`），数组元素类型转换可行，**但语义仍未知**。
- [Nov23 独立技术评审](../../evidence/2026-11-23/MASHUP_SHAPE_FOLLOWUP_REVIEW.md) 及 [设计范围选项](../../docs/MASHUP_DESIGN_OPTIONS.md)：在业务消费和字段定义明确前不得修改、执行 `candidates/3-parse_MashupEvaluationReport.kql`，不得变更目标 Schema/Policy/PROD。
