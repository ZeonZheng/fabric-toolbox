# Gateway Monitoring｜当前状态

> **更新时间：** 2026-11-23（Asia/Singapore）｜**阶段：** P1 必需指标验收（Mashup 三字段已明确延期）｜**POC：** 单节点｜**PROD：** 未批准

| 工作项 | 当前结论 | 关键证据 / 限制 |
| --- | --- | --- |
| Gateway → Fabric → Report | **功能 POC 通过** | 曾观察到八类 Report Raw 连续出现，但未验证文件级不丢失、不重复 |
| QueryStreamingReport | **已部署，POC 功能 Passed** | 32/32 新 Parsed 两个 Duration 非空且与 Raw 数值一致，0 mismatch |
| QueryExecutionAggregationReport | **已部署，POC 功能 Passed** | Raw/Parsed 新样本各 7 行，组合 Key 2/2 对照、六项有值指标 0 mismatch；非 7/7 逐行对账 |
| **MashupEvaluationReport** | **原 Candidate 未部署、仍 Blocked；必需字段尚待验收** | 三个问题字段（TotalBytes/TotalRows/MaxIO）按 ADR-009 转 GW-009 Deferred，不作为当前 P1 阻塞；其当前 null 仍是已知缺陷 |
| Mashup 数组形状/类型 | **2026-11-23 只读诊断完成** | 新 Raw 36 行；TotalBytes 44/44、TotalRows 35/35 个数组元素均可转 long；两数组共同存在 6/6 行长度相等；MaxIO 12/14 有小数 |
| Mashup 三个非必需字段 | **范围已决定：Deferred（GW-009）** | 元素语义/聚合规则未验证、MaxIO 的 long 与小数冲突；未来有实际需求时才重新评估，不影响其余必需字段验收 |
| **当前版本化 Report / Semantic Model 消费** | **未见对三个问题字段的直接引用** | 检查 GitHub Fabric Report JSON、可读 Semantic Model TMDL、KQL Dashboard JSON；**不证明已部署资产/二进制 PBIT/其他下游完全无依赖** |
| GitHub 正式 KQL Schema | **仅 Streaming + Aggregation 已同步** | Mashup 仍使用旧 Function；未改 Parsed Schema、Policy、Uploader 或 PROD |
| 历史回填与 PROD 可靠性 | **未验收** | 历史 null、Uploader Manifest、重试幂等、Retention、Multi-Node、长期稳定性仍开放 |

## 2026-11-23 最新证据

- [Mashup 本地 MCP Shape Follow-up 原始脱敏报告](evidence/2026-11-23/gateway-mashup-shape-followup-20261123-0949-SGT.sanitized.md) · [独立技术评审](evidence/2026-11-23/MASHUP_SHAPE_FOLLOWUP_REVIEW.md)。3 份统计查询成功（查询 4/6 经 `floor(x,1.0)` 最小修正）；当前 GitHub 只读脚本已同步修正。
- 36 行中 TotalBytes 23 行（17 单、6 多）、TotalRows 12 行（4 单、8 多）；同现 6 行 6/6 等长。**结构不等于业务定义**；元素整数转换成功仅说明拆分在技术上可行。
- MaxIO 14 个值中 12 个包含小数；Kusto `real` 可承载浮点数，但**不是保证任意十进制值精确无损**。
- 本地 AI 的 Github Code Search 返回零不能证明无相关项目源码；检查仓库下的 Fabric Report/SM/Dashboard 可读文件未发现问题字段的**直接**消费。

## 当前唯一下一步

**核查剩余必需的 Mashup 指标在 POC 新 Raw/Parsed 中是否正确**（如 Memory/Time/CPU/IO）；如果现有 Function 已满足需求，不为三个已延期字段继续修改它；如果必需字段仍有缺陷，再准备最小 Candidate 并完整 Dry-run / Schema / 回归。**原 Mashup Candidate 仍禁止部署。**

- **范围决定：** `TotalBytes`、`TotalRows`、`MaxIODataBytesPerSecond` 暂不修复、暂不额外重复统计、不联系微软；[ADR-009](tracking/DECISIONS.md) 已确认，[GW-009](tracking/ISSUES.md) 独立追踪。已知 null 不代表值为零或已修复。
- **之后优先：** 进入 P2 Collector Reliability，验证源文件到 Parsed 的遗漏、重复、重试和轮转；PROD 仍未批准，GW-002/003/004/006/007 等风险未解除。
- **现场边界：** 本次只有 GitHub 文档变更；Eventhouse Live Function、Parsed Schema、Update Policy、Uploader、PROD 均未修改。

[问题台账](tracking/ISSUES.md) · [进展记录](tracking/CHANGELOG.md) · [设计决定](docs/MASHUP_DESIGN_OPTIONS.md)。
