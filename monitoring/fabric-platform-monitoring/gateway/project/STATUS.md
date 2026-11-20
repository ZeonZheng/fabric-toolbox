# Gateway Monitoring｜当前状态

> **更新：** 2026-11-20 16:44（Asia/Singapore；Mashup Precheck 报告内结束时间存在 8 分钟不一致）｜**阶段：** P1 Parser 数据正确性｜**POC：** 单节点｜**PROD：** 未批准

| 工作项 | 最新状态 | 验证事实及边界 |
| --- | --- | --- |
| Gateway → Fabric → Report | **功能 POC 已验证** | 八类 Report 有过持续摄入；不证明 Uploader 的文件级无丢失/无重复 |
| **Streaming Parser** | **已部署，POC 功能回归 Passed** | 新 Parsed 32/32 两项 Duration 非空且 Raw/Parsed 数值 32/32 匹配、0 mismatch |
| **Aggregation Parser** | **已部署，POC 功能回归 Passed** | Patch 后 Raw 2 Envelope/7 条、Parsed 7 条；2/2 唯一组合 Key 对照，六项 Raw 有值字段 0 mismatch；不是 7/7 逐行对账 |
| **Mashup Parser** | **未部署；Precheck Blocked** | 当前 379 条 Raw：TotalBytes **243/243**、TotalRows **34/34**、MaxIODataBytesPerSecond **127/127** 非空输入经原 Candidate 转换成 null |
| Mashup Raw 形状/语义 | **待进一步验证和设计决定** | TotalBytes JSON 数组文本 212 单元素+31 多元素；TotalRows 14+20；MaxIO 为小数字符串且目标表 long。多值如何压成单个标量、MaxIO 是否允许舍入尚无结论 |
| Mashup 其它字段 | **部分只读验证通过** | 当前 Raw 的 5 个 Memory/Time 指标各 379/379 转换成功；AvgIO/AvgPercent/MaxPercent 样本转换成功；DataSourceKinds/Paths 原 Raw 无独立 Key |
| 运行中 Mashup Function/Policy | **保持旧版/未变更** | Legacy `bag_unpack` Body，Policy Enabled/Source Raw/Transactional；**没有执行 Mashup DDL** |
| Fork 正式 Schema | **仅 Streaming + Aggregation 已同步** | `src/Gateway/Gateway Monitoring.KQLDatabase/DatabaseSchema.kql` 未应用 Mashup Candidate；禁止用 Jumpstart 整体覆盖 |
| 24h 稳定性/历史回填/PROD 可靠性 | **未验收** | 历史 Parsed null、Uploader Manifest、重试/去重、保留、Multi-Node 等均待验证 |

## Mashup 阻断证据（2026-11-20）

- 用户本地 Eventhouse Remote MCP 在 379 条 Raw 记录中观察到 TotalBytes 数组文本 243/243 `tolong` 失败、TotalRows 数组文本 34/34 `toreal` 失败、MaxIO 小数字符串 127/127 `tolong` 失败；**这些在旧 Parsed 中同样为 null，不是已执行新 Candidate 造成的回归**。
- 旧 Parsed Mashup 2862 行，TotalBytes / TotalRows / MaxIO 三列均 0 非空；当前 Candidate 不能解决这三项。旧 Candidate 的 **17/17 Schema 检查仅证明静态兼容，不证明这些列数值正确**。
- [用户上传的脱敏 Precheck](evidence/2026-11-20/gateway-mashup-precheck-20261120-1626-SGT.sanitized.md) · [独立技术评审](evidence/2026-11-20/MASHUP_PRECHECK_REVIEW.md)。

## 当前唯一下一步

**让本地 AI + Eventhouse Remote MCP 执行 [Mashup 数组/小数语义的下一轮只读诊断](kql/poc-parser-fix-20261117/value-checks/MASHUP_SHAPE_FOLLOWUP_AI_PROMPT.md)**：分析 TotalBytes/TotalRows 的元素数、数组长度关联、各元素是否可转整数、MaxIO 是否有小数/异常，并仅在可验证的官方或项目数据来源中寻找字段业务含义；以**一份完整脱敏中文 Markdown 报告**返回。**诊断期间禁止修改现有 Mashup Candidate、目标表 Schema、Update Policy 或任何现场 Function**。

后续选项要待数据语义确定后再选：保留标量但明确有损聚合/取整策略，或另建保留多值/高精度的结构。所有新方案均需重新 Dry-run、Schema、数据准确性和部署前检查。**不能把保留原有 null 描述成全字段修复。**

[GW-001/GW-005](tracking/ISSUES.md) · [全部历史进展](tracking/CHANGELOG.md)。
