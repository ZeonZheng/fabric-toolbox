# Gateway Monitoring｜当前状态

> **更新：** 2026-11-20 14:22（Asia/Singapore）｜**阶段：** P1 Streaming Parser 已在 POC 手工修改，等待真实新数据验收｜**PROD：** 未批准

| 工作项 | 当前状态 | 证据与限制 |
| --- | --- | --- |
| Gateway → Eventstream → Eventhouse → Report | **POC 功能链路验证过** | 多类 Gateway Report 及 Heartbeat 曾入库；Uploader 可靠性 / 无丢失无重复未验收 |
| QueryStreaming Parser 修改 | **用户已在 POC 手工执行 Candidate** | 记录的验证起点：`2026-11-20T06:19:18.0313089Z`；本地 Postcheck 显示 Function 仍存在且 Body hash 与旧值不同，但尚未做 Live Body vs GitHub Candidate 同口径比对 |
| Streaming Postcheck | **Inconclusive（不是失败，也不是通过）** | 14:22（UTC+8）只读查询：Patch 后 Raw 0、Parsed 0、KeyJoin 0；此前 Raw 最新 06:11:46Z、Parsed 最新摄入 06:11:47Z |
| Streaming Update Policy / ingestion failures | **无当前失败证据** | Enabled=true、Source Raw、IsTransactional=true；Patch 后 `0 observed` Update Policy ingestion failures。无新数据时不能以此证明成功 |
| Aggregation / Mashup Parser | **未修改** | 继续保留 Candidate / Rollback；先验收 Streaming 再行动 |
| Mashup 额外字段 | **仍开放** | 先前 Preflight 发现 `TotalBytes` 全 null，后续修复前需补充测试 |
| PROD 可靠性 | **未验收** | 文件级上传 Manifest、重试/去重、Retention、Multi-Node 及安全性仍开放 |

## 最新证据

- [2026-11-20 14:22 Streaming Postcheck 脱敏报告](evidence/2026-11-20/gateway-streaming-postcheck-20261120-1422-SGT.sanitized.md)。
- [独立复核：为什么目前 Inconclusive](evidence/2026-11-20/STREAMING_POSTCHECK_REVIEW.md)。
- 完整之前测试：3/3 Dry-run、39/39 Schema、13 项数值/时间转换、235/235 DataSource 内容对照，以及 10:36 Preflight 4/4 均已通过；但它们是**部署前**验证。

## 当前唯一下一步

**将 [本地 MCP 后续只读检查 Prompt](kql/poc-parser-fix-20261117/controlled-rollout/FOLLOWUP_AI_PROMPT.md) 发给本地 AI**，检查现有 Function 是否真正与 Candidate 相符、全部 Raw logType 的最新活跃度，以及 `QueryStreamingReport` 全表两个 Duration 是否首次出现非空值。如果仍无新 Streaming，先找产生该类型日志的真实 Gateway 活动来源，而不是再改 Parser。新事件产生后重复原 01/02/03 只读回归，使用相同 UTC 起点。

**禁止**对 Streaming 再次执行 Candidate、自动回滚、继续修改 Aggregation/Mashup 或触碰 PROD。若新出现 Policy ingestion failure、实际配置漂移或明确的解析异常，先暂停后续改动并评审是否人工回滚。

[问题 GW-001](tracking/ISSUES.md) · [2026-11-20 证据](evidence/2026-11-20/README.md)。
