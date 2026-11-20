# Gateway Monitoring｜当前状态

> **更新：** 2026-11-20 15:51（Asia/Singapore）｜**阶段：** P1 Parser 数据正确性｜**环境：** 单节点 POC｜**PROD：** 尚未批准

| 工作项 | 状态 | 实测证据与验收边界 |
| --- | --- | --- |
| Gateway → Fabric → Report | **功能 POC 已验证** | 8 类 Gateway Report Raw 在 2026-11-20 15:47–15:51 的窗口内均有 Patch 后活动；未验收不丢失/不重复 |
| **QueryStreamingReport Parser** | **已部署，POC 功能回归 Passed** | 新 Parsed **32/32** 两个 Duration 均非空且与 Raw **32/32 匹配**；0 mismatch、0 observed failures |
| **QueryExecutionAggregationReport Parser** | **已部署，POC 功能回归 Passed** | PatchUTC `2026-11-20T07:24:24.1258875Z`；Patch 后 Raw 2 Envelope / 7 条、Parsed 7 条；QueryExecution 3 列 7/7 非空、DataProcessing 3 列 1/7 非空，与 Raw 比例一致；Key 比较 2/2 命中，六项有值样本匹配，0 mismatch |
| Function / Update Policies | **两项变更均已验证** | Aggregation BodyMatchCandidate=true（SHA256 `626cb510…c7f21`），Policy Enabled / Raw / Transactional；无相关摄入失败 |
| 正式 GitHub KQL Source | **Streaming + Aggregation 均已同步** | fork 的 `src/Gateway/Gateway Monitoring.KQLDatabase/DatabaseSchema.kql` **仅替换两项已验证的 Function**；没有触发 Eventhouse 再次部署 |
| **MashupEvaluationReport Parser** | **尚未部署，部署前核查中** | `TotalBytes` 旧 Parsed 2797/2797 null；有 Raw Key 但尚未单独通过转换验收；五项 Memory/Time Dry-run、Schema 17/17 已通过，需补做六项字段检查 |
| 长时间稳定性与历史数据 | **尚未验证 / 修复** | Streaming、Aggregation 的当前功能验收样本有限；已存 Parsed null 不会自动重算，历史回填另议 |
| PROD Collector 可靠性 | **未验收** | Upload Manifest、重试/去重、文件级审计、Retention、多节点、安全、持续监控尚未完成 |

## 最新验收证据（2026-11-20）

- Streaming：14:48 新数据回归 **Passed**，[实测评审](evidence/2026-11-20/STREAMING_FOLLOWUP_REVIEW.md)。
- Aggregation：15:25 因没有新数据为 **Inconclusive**；**15:47–15:51 Follow-up Passed**。六个 Duration 的 Raw 非空数与 Parsed 一致；组合 Key 2/2 命中且有值字段匹配，0 mismatch。[Follow-up 证据](evidence/2026-11-20/gateway-aggregation-followup-20261120-1547-SGT.sanitized.md) · [独立评审](evidence/2026-11-20/AGGREGATION_FOLLOWUP_REVIEW.md)。
- 7 条新 Raw / Parsed 只是当前采样窗口行数一致；Query 07 使用组合 Key `arg_max`，将 7 条折叠成 2 个 Key。**不能宣称 7/7 已逐行匹配，更不能证明 Exactly-once**。
- 原报告提出的 Query 07 字段引用错误已被复核并在本次 Follow-up 中更正；GitHub 查询 07 无需修改。

## 当前唯一下一步

**让已连接本地 Eventhouse Remote MCP 的 AI 先完成 [Mashup 部署前只读核查（完整地址与报告要求）](kql/poc-parser-fix-20261117/value-checks/MASHUP_PRECHECK_AI_PROMPT.md)**。主要运行 [六项额外数值检查](kql/poc-parser-fix-20261117/value-checks/3-mashup-additional-metrics.kql)，特别确认 `TotalBytes` 经 `tolong(record.TotalBytes)` 的转换有无丢值，并核实现场 Function / Policy 未漂移。把生成的脱敏中文 Markdown 报告交给 ChatGPT，**通过后才评审 Mashup 单函数人工 POC 部署**。

**不要**重复部署 Streaming 或 Aggregation、不要自动修改 Mashup、不要运行 Jumpstart 重装、更不要触碰 PROD。

[GW-001](tracking/ISSUES.md) · [完整进展](tracking/CHANGELOG.md) · [已验证源码差异](docs/UPSTREAM_CHANGES.md)。
