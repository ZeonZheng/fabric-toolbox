# POC Parser 修复工作区（2026-11-17）

> **注意：目前所有 Patch 仅存在于 GitHub，未执行 Eventhouse Function DDL。** 本目录只用于单节点 POC 的验证与修复候选，不允许直接部署 PROD。

## 范围与安全边界

- 不修改 Gateway Uploader、Task Scheduler、Eventstream、现有 Parsed Table 或 Update Policy。
- 候选修改仅涉及 3 个现有 Function：`parse_QueryExecutionAggregationReport`、`parse_QueryStreamingReport`、`parse_MashupEvaluationReport`。
- 第一轮尽量保持目标 Schema 的列顺序和类型，显式读取 Raw JSON 中的 `(ms)` / `(byte)` 字段，再转换为目标名称。
- `MashupEvaluationReport.DataSourceKinds` 和 `DataSourcePaths` 无对应原始 Key，暂不从 `DataSource` **捏造含义**；这次修复不代表长期完整保留全部 CSV 字段。

## 子目录：按执行风险排列

| 子目录 | 是否改写 Eventhouse | 用途 |
| --- | --- | --- |
| [dryrun/](dryrun/) | **否（只读）** | `let Candidate = () {...}`，用非空计数初步验证 |
| [schema/](schema/README.md) | **否（只读）** | 用 `getschema` 对比 Candidate 与目标表列名/顺序/数据类型 |
| [candidates/](candidates/) | **是（变更 Function）** | 每个文件含 `.create-or-alter function`；**现在禁止运行** |
| [rollback/](rollback/) | **是（回滚 Function）** | 以实际部署导出的旧 Body 恢复，必须确认现场版本后使用 |

备份：[15 个实际 Function Body](../../evidence/2026-11-17/DeployedFunctions.full.export.csv) · [142 行 Schema](../../evidence/2026-11-17/DeployedTableSchema.sanitized.csv) · [字段审计](../../docs/SCHEMA_FIELD_MAPPING_AUDIT.md)。

## 验证步骤与当前证据

1. 已完成：3 个 `dryrun/` 查询成功，Aggregation 77 Rows、Streaming 113 Rows、Mashup 91 Rows，具体非空率见 [证据](../../evidence/2026-11-17/DRYRUN_RESULTS.md)。
2. **待执行：** 运行 [3 个 schema 查询](schema/README.md)，核对所有 `ColumnOrdinal`、`ColumnName`、`ColumnType` 的 `IsMatch`；任意不一致立即停止。
3. 待执行：对照真实 Raw JSON 和 Candidate 的代表性值、`DataSource` 结构、Numeric Precision、`TotalProcessorTime(ms)` → timespan。
4. 通过后才评审 `candidates/`：在 **POC** 中每次只更新一个 Parser，记录精确部署时间、Git Commit SHA、对应 Rollback。
5. 部署后检查新摄入记录、Update Policy、Eventstream 指标和 Ingestion Failures；旧 Parsed null **不会自动修复**，历史回填需单独设计。
6. 全部验证完成后，再把认可的修复应用到 GitHub 的正式 Schema 源码定义；不要未经评估就运行 Jumpstart `update_existing=True`。

## 特别风险

当前 Update Policies 使用 `IsTransactional=true`。如果函数输出与目标表 Schema 不匹配，可能影响 Raw 摄入；仅 Dry-run 的非空验证不足以保证线上数据完整性。Kusto 源码里的 DataSource 标准化表达式已调整输入对象为 `record.DataSource`，需要逐值核对结构。

Microsoft 参考：[Update Policy](https://learn.microsoft.com/en-us/kusto/management/update-policy)。

**唯一最新进度：[STATUS.md](../../STATUS.md)**。避免在脚本说明里重复维护项目当前阶段。
