# 本地 AI / Eventhouse Remote MCP｜Mashup Parser 部署前最后一轮只读核查

> **目标：** 验证尚未部署的 `parse_MashupEvaluationReport` Candidate 对 **TotalBytes / TotalRows / CPU / IO** 等已有字段的转换，结合历史五项 Memory/Time 和 Schema 17/17 检查，判断是否满足进入 **手工 POC Function DDL 评审**的条件。  
> **当前状态：** Streaming 与 Aggregation 都已经在 POC 手工部署并完成新数据功能回归；Mashup **尚未部署**。**本次仅查询，不执行修复。**

## 1. 固定的仓库与版本

- 仓库：https://github.com/ZeonZheng/fabric-toolbox
- **分支：** `docs/gateway-monitoring-poc-20261112`（不得改用 main）。
- 查询文件：[3-mashup-additional-metrics.kql](https://github.com/ZeonZheng/fabric-toolbox/blob/docs/gateway-monitoring-poc-20261112/monitoring/fabric-platform-monitoring/gateway/project/kql/poc-parser-fix-20261117/value-checks/3-mashup-additional-metrics.kql)
- Candidate（**只用于阅读，不得执行**）：[3-parse_MashupEvaluationReport.kql](https://github.com/ZeonZheng/fabric-toolbox/blob/docs/gateway-monitoring-poc-20261112/monitoring/fabric-platform-monitoring/gateway/project/kql/poc-parser-fix-20261117/candidates/3-parse_MashupEvaluationReport.kql)
- 既有 Schema Check（2026-11-17 已通过 17/17）：[3-schema-check ](https://github.com/ZeonZheng/fabric-toolbox/blob/docs/gateway-monitoring-poc-20261112/monitoring/fabric-platform-monitoring/gateway/project/kql/poc-parser-fix-20261117/schema/3-schema-check-parse_MashupEvaluationReport.kql)
- 已验证的 Dry-run：[3-dryrun ](https://github.com/ZeonZheng/fabric-toolbox/blob/docs/gateway-monitoring-poc-20261112/monitoring/fabric-platform-monitoring/gateway/project/kql/poc-parser-fix-20261117/dryrun/3-dryrun-parse_MashupEvaluationReport.kql)
- 项目最新状态：[STATUS.md](https://github.com/ZeonZheng/fabric-toolbox/blob/docs/gateway-monitoring-poc-20261112/monitoring/fabric-platform-monitoring/gateway/project/STATUS.md)

## 2. 本地 MCP 只读执行要求

1. 使用已经连接的 `eventhouse-remote` MCP **只读**工具；先确认目标仍是用户的 **POC Gateway Monitoring KQL Database**。如果无法确认，停止并报告 `Inconclusive`。
2. 从正确分支获取 `3-mashup-additional-metrics.kql`，通过 MCP 执行并保存**每个指标各一行**的原始脱敏汇总，至少包括 `FieldName`、`TargetType`、`Rows`、`RawNonEmpty`、`CandidateNonNull`、`ConversionLoss`、`ChangedByCast`。
3. **必须单独检查 `TotalBytes`**：Preflight 的旧 Parsed `TotalBytes` 在当时 **2797/2797 null**，但 Raw 有此 Key。此轮检验 `tolong(record.TotalBytes)` 对当前 Raw 非空样本可否转换；如果没有当前 Raw 样本，则该项是 `Not verified`，不能因旧结论宣称修复必然成功。
4. 校对 `MaxIODataBytesPerSecond` 等 long 字段转换、`TotalRows`、`AveragePercentProcessorTime`、`MaxPercentProcessorTime`、`AverageIODataBytesPerSecond` 是否有空值转化损失或数值变化。区分 Raw 本来为 null 与本来有值却转为 null。
5. 复查现场 `.show functions` 中 `parse_MashupEvaluationReport` 是否仍为旧版，及其 Update Policy 是否 Enabled、Source=`GatewayReports-Raw`、IsTransactional=true。**绝不修改 Function**。
6. 目前 `DataSourceKinds` / `DataSourcePaths` **不是 103 条历史 Raw 样本中的独立 JSON Key**；这两列在最小 Candidate 中仍可能为空。不要从 `DataSource` 自动推断两列或把全字段覆盖率误报为已通过。记录为**已知数据模型设计限制**，不在本轮偷偷改变目标 Schema。
7. 不重新运行之前已经验证的 DDL、不执行 `.create-or-alter`、`.set`、`.alter`、Rollback、Uploader 修改、Task 启停或任何 PROD 操作。若 MCP 查询工具不支持某命令，报告 `UNSUPPORTED` 而非猜测结果。

## 3. 输出合同：一份中文 Markdown 报告

文件名 `gateway-mashup-precheck-YYYYMMDD-HHmm-SGT.md`。请提供真实文件路径或全文，供用户上传 ChatGPT。

| 部分 | 必需数据 |
| --- | --- |
| A. 环境/执行时间 | POC 依据、MCP 工具、GitHub 分支/Commit、实际 UTC/UTC+8 |
| B. Function / Policy | 旧 Function 是否仍存在、是否未执行 DDL、Policy Enabled/Source/Transactional |
| C. 六项新增数值检查 | 每行 FieldName、Rows、RawNonEmpty、CandidateNonNull、ConversionLoss、ChangedByCast、必要的 null 来源说明 |
| D. TotalBytes 特别复核 | Raw 是否有当前样本、非空到 long 是否成功、为何旧 Parsed 为 null 的原有证据与限制 |
| E. Schema / 既有测试 | 之前 17/17 Schema 及五项 Memory/Time Dry-run 的状态；本轮没重测的明确注明历史证据，不要虚称实时重跑 |
| F. Verdict | `Ready for manual POC patch review` / `Blocked` / `Inconclusive`、证据、数据模型限制、未验证项 |

**判定**：当前 Raw 的必要字段存在、所有本轮有值的字段 `ConversionLoss=0` 且 `ChangedByCast=0`（对于数值比较阈值 0.0001），Policy / Live Function 状态符合预期，才可判 `Ready for manual POC patch review`。缺少足够 Raw 样本或无法查询则 `Inconclusive`；确有转换丢失、类型/Policy 漂移等则 `Blocked`。这里的 Ready **不是授权自动执行 DDL**。

**不要返回** QueryText、真实 DataSource、原始日志、GatewayObjectId、ConnectionId、个人账号、URL 或 token。最终只输出脱敏统计及必要的错误摘要。
