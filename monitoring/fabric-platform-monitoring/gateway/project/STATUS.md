# Gateway Monitoring｜当前状态

> **更新时间：** 2026-11-23（Asia/Singapore）｜**阶段：** P1 必需指标验收（Mashup 三字段已明确延期）｜**POC：** 单节点｜**PROD：** 未批准

| 工作项 | 当前结论 | 关键证据 / 限制 |
| --- | --- | --- |
| Gateway → Fabric → Report | **功能 POC 通过** | 曾观察到八类 Report Raw 连续出现，但未验证文件级不丢失、不重复 |
| QueryStreamingReport | **已部署，POC 功能 Passed** | 32/32 新 Parsed 两个 Duration 非空且与 Raw 数值一致，0 mismatch |
| QueryExecutionAggregationReport | **已部署，POC 功能 Passed** | Raw/Parsed 新样本各 7 行，组合 Key 2/2 对照、六项有值指标 0 mismatch；非 7/7 逐行对账 |
| **MashupEvaluationReport** | **POC 部署前 Gate PASS；等待用户手工部署/新数据回归** | 五项目标指标转换 4×33/33 + ProcessorTime 36/36，Schema 17/17；非目标 12 列 Legacy/Candidate 36/36、0 差异。Live Function 尚未修改，P1 还不能标记功能 Passed |
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

## 2026-11-23 Mashup 最小 Precheck 新证据

- 用户提交本地 MCP 只读报告：五个目标 Memory/Time 指标（4 × 33/33、1 × 36/36）ConversionLoss 0、ChangedByCast 0，目标 Schema 17/17；当前 Live Function 仍旧版，Update Policy 仍 Transactional。参见 [独立评审](evidence/2026-11-23/MASHUP_MINIMAL_PRECHECK_REVIEW.md)。
- v2 Candidate 的函数体**与旧 Blocked Candidate 完全相同**，区别在三个非必需指标明确延期（GW-009）；该 Precheck 阶段尚未验证另外 12 列兼容性，已在下述单独 Gate 中补足。

## 2026-11-23 Mashup 12 列兼容性 Gate 结果

- 用户报告在 POC 原样只读执行 [compatibility.kql](kql/poc-parser-fix-20261117/value-checks/7-mashup-legacy-candidate-compatibility.kql)：**Legacy 36 / Candidate 36、DifferingSignatureGroups 0、UnmatchedRows 0、PASS**；五项目标字段的独立 Dry-run 也已通过。见 [验收证据](evidence/2026-11-23/MASHUP_LEGACY_CANDIDATE_COMPATIBILITY_PASS.md)。
- 经过独立评审，当前 POC **部署前功能 Gate 已通过**，已准备专门的 [P1 POC Candidate](kql/poc-parser-fix-20261117/candidates/4-parse_MashupEvaluationReport_P1_minimal_POC.kql) 与 [受控部署/回归 Runbook](kql/poc-parser-fix-20261117/controlled-rollout/MASHUP_RUNBOOK.md)；现有现场旧版 Rollback 文件已确认存在。
- **注意：这不是已部署，也不是 Mashup Postcheck Passed。** 事务性 Update Policy 风险、历史 Parsed null、GW-009 延期及 PROD 可靠性风险仍存在。

## 当前唯一下一步

**用户按 [Mashup POC Runbook](kql/poc-parser-fix-20261117/controlled-rollout/MASHUP_RUNBOOK.md) 在已人工确认的 POC Fabric UI 手工部署**单个 Function（仅在再次核对现场 Function 与 Policy 无漂移后，先准备好回滚）。部署后立即记录真实 UTC；本地 MCP **仅只读**执行已备妥的 [10/11/12 Postcheck](kql/poc-parser-fix-20261117/controlled-rollout/MASHUP_RUNBOOK.md)，将新 Raw/Parsed 五字段对账与 Policy failures 报告交回独立评审。**任何非 POC 环境或漂移情况禁止部署**。

- **范围决定：** `TotalBytes`、`TotalRows`、`MaxIODataBytesPerSecond` 暂不修复、暂不额外重复统计、不联系微软；[ADR-009](tracking/DECISIONS.md) 已确认，[GW-009](tracking/ISSUES.md) 独立追踪。已知 null 不代表值为零或已修复。
- **之后优先：** 进入 P2 Collector Reliability，验证源文件到 Parsed 的遗漏、重复、重试和轮转；PROD 仍未批准，GW-002/003/004/006/007 等风险未解除。
- **现场边界：** 本次只有 GitHub 文档变更；Eventhouse Live Function、Parsed Schema、Update Policy、Uploader、PROD 均未修改。

[问题台账](tracking/ISSUES.md) · [进展记录](tracking/CHANGELOG.md) · [设计决定](docs/MASHUP_DESIGN_OPTIONS.md)。
