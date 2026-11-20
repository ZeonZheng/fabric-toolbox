# 数值与单位验证（只读）

**状态（2026-11-17 16:54 UTC+8）：** 用户已在 POC Eventhouse 成功运行本查询，结果详见 [ValueCheck-ParserConversions.csv](../../../evidence/2026-11-17/ValueCheck-ParserConversions.csv)。13 个数值指标无转换损失/超阈值变化，2 项 DataSource 仅验证了非空率。

**DataSource 下一项已完成（2026-11-17 17:11 UTC+8）：** Legacy / Candidate 内容对照 Aggregate 96/96 + Streaming 139/139 一致；另有 Mashup Kinds/Paths Raw Key 缺失事实；见 [导出](../../../evidence/2026-11-17/DataSourceCheck-LegacyVsCandidate.csv)。禁止把脚本运行成功当作对实际 Function 变更的授权。

[1-parser-value-validation.kql](1-parser-value-validation.kql) 用**一个查询**对三个 Candidate 涉及的指标作转换风险检查：

- QueryExecutionAggregation：6 个 Duration 的 `(ms)` 原值 → `real / long`；
- QueryStreaming：2 个 Duration 的 `(ms)` 原值 → `long`；
- MashupEvaluation：4 个 Memory `(byte)` 原值 → `real / long`，`TotalProcessorTime(ms)` → `timespan` 的数值回算；
- Aggregation、Streaming 的 `DataSource`：仅检查是否从非空变为空，**不输出数据源信息，也不证明结构/语义完全一致**。

## 字段解释

| 列 | 含义 |
| --- | --- |
| `RecordCount` | 此时间点 Raw 中的对应记录量；会因 1 天保留期变化 |
| `RawNonEmpty` | 原始 JSON Key 非空的记录数 |
| `CandidateNonNull` | 通过候选类型转换后非 null 的记录数 |
| `ConversionLoss` | Raw 非空但候选输出为 null；**必须为 0 或解释** |
| `ChangedByCast` | 浮点/long 转换或 timespan 回算后与原值差异超过 0.0001 的次数；**必须为 0 或解释**。DataSource 该项为 null（未比较值） |
| `SampleRaw` / `SampleCandidate` | 单次抽样，仅便于检查数量级；**来自不同的任意行**，不可直接将两列逐行比较。DataSource 样本隐藏 |

### 使用和局限

在 **POC KQL Queryset** 运行此查询并导出 CSV。若 `ConversionLoss > 0`、`ChangedByCast > 0`、语法错误，先停止并分析；如果均为 0，还需对 `DataSource` 的实际结构做一次有权限的人工比对，而不是只看非空率。

**重要：** 本查询属于无状态聚合检查，不确认上传全量性、不修复已存储的 null，也不证明旧 Parser 的所有其他字段保持不变。`SampleRaw` 和 `SampleCandidate` 是独立抽样，不能将其视作配对结果。数值结果仅为转换合理性验证，仍需在变更前人工复核。

证据归档：[2026-11-17](../../../evidence/2026-11-17/README.md) · 当前状态：[STATUS](../../../STATUS.md)。


## Mashup 部署前检查（2026-11-20）

- [3-mashup-additional-metrics.kql](3-mashup-additional-metrics.kql)：6 个 Raw 数值字段的转换丢失、long 精度、Raw 本来为空的处理。**尚未在用户 Eventhouse 实测**，不得声称通过。
- [MASHUP_PRECHECK_AI_PROMPT.md](MASHUP_PRECHECK_AI_PROMPT.md)：供本地 Eventhouse Remote MCP 的完整执行说明（仓库/分支/URL/中文 Markdown 输出合同）。
- **阻断原则：** 在 `TotalBytes` 等字段确认可安全转换、旧版现场 Function/Policy 未漂移之前，不运行 Mashup Candidate，也不更新正式 KQL Schema。
