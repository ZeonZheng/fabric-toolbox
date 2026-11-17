# DataSource 语义检查（只读）

本检查用于补齐 [数值/单位检查](../value-checks/README.md) 未验证的部分：**修复 Candidate 中的 DataSource 表达式是否与原版 Parser 输出一致**。源代码对照已证明原版先用 `bag_unpack(record)` 提取 DataSource:dynamic，再做 `dynamic_to_json → replace_strings → parse_json`。新版从 `record.DataSource` 直接提取，虽然非空，但仍应验证转换等价性。

- [2-datasource-compatibility.kql](2-datasource-compatibility.kql) 是**单个只读查询**，不显示原始 DataSource、URL、连接串。
- QueryExecutionAggregation、QueryStreaming：比较 Legacy 和 Candidate 的规范化结果（字符串序列化方式）。`DifferentSerialized` 预期为 **0**，`LegacyNonEmpty` 与 `CandidateNonEmpty` 应一致。
- Mashup：只统计 Raw 中 `DataSource` 的存在及 `DataSourceKinds` / `DataSourcePaths` **是否有实际 JSON Key**；暂不自动把 DataSource 强行推导为 Kinds / Paths。
- 若发现 `DifferentSerialized > 0`：先人工审阅 KQL 表达式与输出结构（在受限环境处理敏感信息），不能直接合并 Candidate。
- **限制：** 即使序列化值一致，也不证明 Raw→Parsed 历史数据完整；字符串序列化可能受 dynamic 属性顺序影响，出现差异需进一步判断。不需要将 DataSource 原始值发到聊天或 GitHub。

**当前状态与下一步只看 [STATUS](../../../STATUS.md)**。此脚本已于 **2026-11-17 17:11 UTC+8** 在用户 POC KQL Queryset 执行，**235/235 行内容等价**（DifferenceSerialized=0），详见 [CSV 证据](../../../evidence/2026-11-17/DataSourceCheck-LegacyVsCandidate.csv)。下一步以项目 [STATUS](../../../STATUS.md) 为准。 
