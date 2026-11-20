# Gateway Monitoring｜当前状态

> **更新：** 2026-11-20 14:48（Asia/Singapore）｜**阶段：** P1 Parser 数据准确性（Streaming 功能性修复已通过）｜**环境：** POC 单节点｜**PROD：** 尚未批准

| 工作项 | 当前状态 | 实测依据 / 验收边界 |
| --- | --- | --- |
| Gateway → Fabric → Report | **POC 功能链路已验证** | 8 种 Report Raw 类型在 2026-11-20 Patch 后均有新 Envelope；不能证明 Uploader 无丢失、无重复 |
| **Streaming Parser** | **已部署；首轮功能回归 Passed** | 现场 Body 与 GitHub Candidate **逐字节一致**；Patch 后 32/32 Parsed 行两个 Duration 均非空，Raw / Parsed 32/32 双指标匹配、0 mismatch |
| Streaming Update Policy | **保持正常** | Enabled、Transactional、Source Raw；Post-Patch **0 observed** ingestion failures |
| Streaming 历史与长时间稳定性 | **未完成** | 历史 2675 行两个 Duration 仍全 null；仅约 25 分钟、32 行样本；后续持续观察不等于此时功能回归失败 |
| Aggregation Parser | **待 POC 受控部署** | 6 Duration 的 Dry-run / Schema / 数值只读测试先前通过，旧 Function 未修改 |
| Mashup Parser | **待额外测试与受控部署** | 旧 Parsed TotalBytes 2797/2797 null；部署前单独测试该字段和其他数值转换，旧 Function 未修改 |
| GitHub Source of Truth | **Streaming 已同步** | fork `src/Gateway/Gateway Monitoring.KQLDatabase/DatabaseSchema.kql` 仅替换已验证的 Streaming Function；原始定义在 dated baseline，可回滚；**未对 Eventhouse 再部署** |
| PROD Collector 可靠性 | **未验收** | GW-002～GW-007：File/Batch Manifest、幂等、重试、Retention、Multi-node、隐私、生产回填 |

## 2026-11-20 最新验收证据

**Patch 验证起点：** `2026-11-20T06:19:18.0313089Z`（新加坡 14:19:18）。

- 14:22 首轮 Postcheck 因没有新 Streaming 记录为 **Inconclusive**；[第一次报告](evidence/2026-11-20/STREAMING_POSTCHECK_REVIEW.md)。
- 14:22–14:48 Follow-up **Passed**：Streaming Raw 5 个新 Envelope，展开 32 行；Parsed 32 行，两项 Duration **32/32 非空并与 Raw 逐值一致**，0 mismatch；历史 2675 行仍 null。现场 Function 的 Body 与 Candidate 严格一致（SHA256 `587bba1a…703ac`）；Policy 未变，0 observed failures。
- [Follow-up 原始脱敏报告](evidence/2026-11-20/gateway-streaming-followup-20261120-1445-SGT.sanitized.md) · [独立复核](evidence/2026-11-20/STREAMING_FOLLOWUP_REVIEW.md)。
- **结论适用范围：** 仅 Streaming 修复后的新记录在这批样本中恢复正确；不是原始日志的完整采集/Exactly-once 证明，也未授权 PROD。

## 当前唯一下一步

**对第二个对象 `parse_QueryExecutionAggregationReport` 做一次 POC 受控变更。** 参照 [Aggregation 操作指南](kql/poc-parser-fix-20261117/controlled-rollout/AGGREGATION_RUNBOOK.md)：在 Fabric UI 人工确认 POC、打开回滚文件、**仅运行 Aggregation Candidate 一次**、记录实际 UTC；部署后的检查交由本地 Eventhouse Remote MCP，返回完整 Markdown 报告再评审。**不要同时修改 Mashup，不要重跑 Streaming Candidate。**

**并行观察（非本轮单一操作）：** Streaming 的 24h 稳定性及历史 null 是否回填为后续独立事项；GW-001 作为总体问题保持进行中。

[问题台账](tracking/ISSUES.md) · [进展](tracking/CHANGELOG.md) · [源码差异](docs/UPSTREAM_CHANGES.md)。
