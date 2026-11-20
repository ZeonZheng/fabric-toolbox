# 2026-11-20｜Streaming Parser 修复后 Follow-up 独立评审

> **实测时间：** 2026-11-20 14:22–14:48 Asia/Singapore（来源报告时间窗口）。**变更后验证起点：** 2026-11-20T06:19:18.0313089Z。  
> **证据：** [本地 AI / Eventhouse Remote MCP 只读报告](gateway-streaming-followup-20261120-1445-SGT.sanitized.md)。ChatGPT 未直接连接 POC 数据库复跑，结论基于用户提供的报告与之前 GitHub Candidate / Baseline。  
> **验收决定：** **Streaming Parser POC 首次新数据功能回归 Passed**；不是端到端 Uploader 可靠性通过，也不代表 PROD 可部署。

## 已证实的事实

| 证据项 | 结果 | 判定 |
| --- | --- | --- |
| Live Body 与 Candidate | 逐字节一致，均 734 字符；同一规范化 SHA256: `587bba1a7ad9dac82c873b8a2ed4ef5ef6cc15a5b8cd3ef3e62df13b810703ac` | 已验证 **确实部署了 GitHub Candidate** |
| Update Policy | Enabled=true、Source=`GatewayReports-Raw`、IsTransactional=true | 配置没有观察到漂移 |
| 其他类型 Raw 活跃度 | 8/8 类在 Patch 后均有 3–6 条新 Envelope；Streaming 5 个 Envelope | Uploader/流转当时有活动，**不能证明完整传输** |
| Parsed 历史行 | 2675 行，两个 Duration 非空均 0 | 旧数据未重算，符合预期 |
| Parsed 新摄入行 | 32 行，`StreamingDuration_ms_` 32/32、`SpoolingDiskReadingDuration_ms_` 32/32，均非空 | **修复生效** |
| Raw vs Parsed | 新样本 Raw=32、Parsed=32；KeyCheck 32/32，两个 Duration 每项 32/32 数值相同，Mismatches=0 | **当前样本的字段映射和数值一致**；不证明文件级 exactly-once |
| Post-Patch ingestion failures | `0 observed` Update Policy 来源失败 | 没有观察到相关失败；无失败 ≠ 无其它数据问题 |

## 仍然开放的事项

1. **观察窗口约 25 分钟，样本为 32 行。** 建议继续观察至少 24h 的真实负载（后续新报告），确认中途没有空值回归、Policy failures 或记录中断；本次短时修复可判通过，不必等 24h 才确认功能修复已生效。
2. Raw (`logDate` 到达/envelope 时间) 和 Parsed (`StreamingStartTimeUTC` 业务时间) 非同一时钟；样本 KeyJoin 不确保无重传、无重复或无丢失。
3. **历史 2675 条 Parsed null 未修复。** Update Policy Function 更新仅对后续摄入生效；历史数据重放/回填单独评审。
4. Streaming Raw 有三个未投影字段：`StreamingTrackingId`、`StreamingDelayDuration(ms)`、`StreamingPacketsDuration(ms)`；本轮未更改目标 Schema（GW-005）。
5. Aggregation 6 Duration 和 Mashup 的 Memory/TotalBytes 等问题仍待实际修复；**GW-001 总问题不得关闭**。

## GitHub 与 POC 同步

- 已验证的 Streaming Candidate 与现场 Body 完全相同；可将**该单个 Function** 的修改同步到 fork 的上游 Schema 位置 `src/Gateway/Gateway Monitoring.KQLDatabase/DatabaseSchema.kql`，保留原始 [2026-11-17 KQL 基线](../2026-11-17/DatabaseSchema.source.kql) 作为回滚参考，并在 [UPSTREAM_CHANGES](../../docs/UPSTREAM_CHANGES.md) 中标识 fork delta。
- 本次对 GitHub 源文件的更新**不自动执行 Eventhouse 部署**，不会修改实际 PROD、Uploader、Policy、其他 Parser。
- 不要用整个 Schema 的 Jumpstart re-install / update_existing=True 来部署单个 Function，避免覆盖其它项。

**阶段结论：** `GW-001 / Streaming subtask = POC functional fix accepted`；总体 GW-001 仍进行中。下一对象为 Aggregation；建议继续对 Streaming 开展独立的 24h 监测，而非将该功能验收与长期稳定性混为一谈。
