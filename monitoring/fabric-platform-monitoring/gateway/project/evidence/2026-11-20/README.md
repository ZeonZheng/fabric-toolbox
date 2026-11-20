# 2026-11-20｜POC Eventhouse Preflight、Streaming 与 Aggregation Postcheck

| 证据 | 来源 | 判定 |
| --- | --- | --- |
| [Preflight 原始脱敏报告](gateway-preflight-20261120-1036-SGT.sanitized.md) | 用户上传的本地 AI 只读 Preflight | 4/4 PASS，允许人工部署评审 |
| [Preflight 独立评审](PRE_DEPLOYMENT_REVIEW.md) | GitHub 基线 + 报告的字段、Policy 和 Function 核对 | 准备 Streaming 单函数修复；未授权自动修改 |
| [Streaming Postcheck 脱敏报告](gateway-streaming-postcheck-20261120-1422-SGT.sanitized.md) | 用户手工部署后，本地 MCP 只读查询 | **Inconclusive**：Patch 后 Raw/Parsed 都无 Streaming 新记录；0 相关摄入失败 |
| [Streaming Postcheck 独立评审](STREAMING_POSTCHECK_REVIEW.md) | 2026-11-20 14:22 独立分析 | 当时没有新数据，因此 Inconclusive；历史步骤保留 |
| [Streaming Follow-up 原始脱敏报告](gateway-streaming-followup-20261120-1445-SGT.sanitized.md) | 本地 MCP 14:22–14:48 的新数据只读回归 | **Passed：新 Parsed 32/32 双 Duration 非空、Raw 32/32 值一致、0 mismatch** |
| [Streaming Follow-up 独立评审](STREAMING_FOLLOWUP_REVIEW.md) | GitHub Candidate 与报告证据复核 | **POC 功能性修复通过**，24h 长期观察/历史回填另议 |
| [Aggregation 首次 Postcheck 脱敏报告](gateway-aggregation-postcheck-20261120-1525-SGT.sanitized.md) | 本地 MCP 15:25–15:29 Postcheck | **Inconclusive**：现场 Candidate Body 匹配，但 Patch 后 Raw/Parsed 均 0，六指标无新样本 |
| [Aggregation 独立评审](AGGREGATION_POSTCHECK_REVIEW.md) | GitHub Query 07 实际代码核对 + 报告审计 | **12/12 字段比较表达式正确**；报告错误声明未采纳；准备按真实 PatchUtc Follow-up |
| [Aggregation Follow-up 脱敏报告](gateway-aggregation-followup-20261120-1547-SGT.sanitized.md) | 本地 MCP 15:47–15:51 的只读回归 | **Passed**：新 Raw/Parsed 均 7 行、六指标非空计数一致；2/2 唯一 Key 对照，0 mismatch |
| [Aggregation Follow-up 独立评审](AGGREGATION_FOLLOWUP_REVIEW.md) | ChatGPT 根据用户报告和 fork Candidate 复核 | **功能性修复通过**，仅限当前样本；已同步 fork 正式 KQL Schema |

| [Mashup Precheck 脱敏报告](gateway-mashup-precheck-20261120-1626-SGT.sanitized.md) | 本地 MCP 16:27–16:44 只读核查（报告结束时刻有 8 分钟差异） | **Blocked**：TotalBytes 243/243、TotalRows 34/34、MaxIO 127/127 Raw 有值却转换为 null；Mashup 未部署 |
| [Mashup Precheck 独立评审](MASHUP_PRECHECK_REVIEW.md) | 源代码与实测统计独立分析 | 数组多元素含义与 long/real 的 Schema 设计风险，禁止部署原 Candidate |

**Streaming 与 Aggregation 已 POC 功能性通过；Mashup 未修改且 Precheck Blocked。**

**当前已实施变更：** 用户报告已在 POC 手动应用 `parse_QueryStreamingReport` Candidate；已获得部署后 32 条新 Parsed 的字段与 Raw 匹配证据，POC 首轮功能回归通过。Aggregation 已手动部署并通过 15:47–15:51 新数据回归，Mashup 不变。**PROD 未改动。**

当前状态：[STATUS](../../STATUS.md)。
