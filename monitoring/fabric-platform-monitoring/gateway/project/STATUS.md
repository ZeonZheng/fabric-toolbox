# Gateway Monitoring｜当前状态

> **更新时间：** 2026-11-23 09:55（Asia/Singapore）｜**阶段：** P1 Parser Data Correctness 与 Mashup 设计评审｜**POC：** 单节点｜**PROD：** 未批准

| 工作项 | 当前结论 | 关键证据 / 限制 |
| --- | --- | --- |
| Gateway → Fabric → Report | **功能 POC 通过** | 曾观察到八类 Report Raw 连续出现，但未验证文件级不丢失、不重复 |
| QueryStreamingReport | **已部署，POC 功能 Passed** | 32/32 新 Parsed 两个 Duration 非空且与 Raw 数值一致，0 mismatch |
| QueryExecutionAggregationReport | **已部署，POC 功能 Passed** | Raw/Parsed 新样本各 7 行，组合 Key 2/2 对照、六项有值指标 0 mismatch；非 7/7 逐行对账 |
| **MashupEvaluationReport** | **未部署；当前 Candidate 仍 Blocked** | 原候选三个字段全转 null：TotalBytes 243/243、TotalRows 34/34、MaxIO 127/127（2026-11-20 样本）。旧 Parsed 亦有同样 null |
| Mashup 数组形状/类型 | **2026-11-23 只读诊断完成** | 新 Raw 36 行；TotalBytes 44/44、TotalRows 35/35 个数组元素均可转 long；两数组共同存在 6/6 行长度相等；MaxIO 12/14 有小数 |
| Mashup 字段业务定义 | **未验证，仍是设计阻碍** | 不清楚多元素对应什么、能否 SUM/FIRST/展开；MaxIO 当前目标 long 不能准确保留小数 |
| **当前版本化 Report / Semantic Model 消费** | **未见对三个问题字段的直接引用** | 检查 GitHub Fabric Report JSON、可读 Semantic Model TMDL、KQL Dashboard JSON；**不证明已部署资产/二进制 PBIT/其他下游完全无依赖** |
| GitHub 正式 KQL Schema | **仅 Streaming + Aggregation 已同步** | Mashup 仍使用旧 Function；未改 Parsed Schema、Policy、Uploader 或 PROD |
| 历史回填与 PROD 可靠性 | **未验收** | 历史 null、Uploader Manifest、重试幂等、Retention、Multi-Node、长期稳定性仍开放 |

## 2026-11-23 最新证据

- [Mashup 本地 MCP Shape Follow-up 原始脱敏报告](evidence/2026-11-23/gateway-mashup-shape-followup-20261123-0949-SGT.sanitized.md) · [独立技术评审](evidence/2026-11-23/MASHUP_SHAPE_FOLLOWUP_REVIEW.md)。3 份统计查询成功（查询 4/6 经 `floor(x,1.0)` 最小修正）；当前 GitHub 只读脚本已同步修正。
- 36 行中 TotalBytes 23 行（17 单、6 多）、TotalRows 12 行（4 单、8 多）；同现 6 行 6/6 等长。**结构不等于业务定义**；元素整数转换成功仅说明拆分在技术上可行。
- MaxIO 14 个值中 12 个包含小数；Kusto `real` 可承载浮点数，但**不是保证任意十进制值精确无损**。
- 本地 AI 的 Github Code Search 返回零不能证明无相关项目源码；检查仓库下的 Fabric Report/SM/Dashboard 可读文件未发现问题字段的**直接**消费。

## 当前唯一下一步

**先完成 [Mashup 字段业务必要性与实现范围评审](docs/MASHUP_DESIGN_OPTIONS.md)**：确认实际部署的 Gateway Monitor Report / Semantic Model（以及预期 PROD 监控需求）是否需要 TotalBytes、TotalRows、MaxIO 三个字段。当前 fork 版本化报表未见直接引用，但不是线上无依赖的证明。

- **若当前监控不需要：** 可以评审“限定 P1 验收范围、保留旧 Mashup Function 和已知 null，并将全字段修复列为单独 Backlog”，但**不能静默关闭 GW-001**。
- **若需要：** 先向 Microsoft Gateway 产品/支持渠道确认数组元素定义、求和/聚合规则和 MaxIO 精度，再决定是否新增明细结构或变更类型。未经定义不能擅自取首元素/求和/取整。

**暂时不运行任何新 DDL、Schema 变更或重复数值采样，不部署原 Mashup Candidate。**

[问题台账](tracking/ISSUES.md) · [进展记录](tracking/CHANGELOG.md) · [设计选项](docs/MASHUP_DESIGN_OPTIONS.md)。
