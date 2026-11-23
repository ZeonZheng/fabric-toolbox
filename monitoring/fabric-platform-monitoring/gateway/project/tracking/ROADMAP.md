# 项目阶段与验收关口

> 更新：2026-11-23（Asia/Singapore）。此表记录阶段，不虚构预计完成日期；每次阶段结束必须有可复现证据。

| 阶段 | 关键交付 / 通过条件 | 状态 |
| --- | --- | --- |
| **P0 — Functional POC** | Gateway → Fabric → Report 链路、8 种 Report、Heartbeat / NodeInfo 功能可用 | **功能性验收完成**；不含可靠性保证 |
| **P1 — Data Correctness** | **当前必需** Raw 字段正确映射；Schema、数值、新数据回归合格；Patch/Rollback 可复现；被明确排除的字段保留独立风险记录 | **本轮 P1 必需 Parser 功能性 POC 验收完成（2026-11-23）**：Streaming 32/32、Aggregation 新 Raw/Parsed 各 7 行（复合 Key 2/2）、Mashup 4/4 新 Raw/Parsed 五项指标零不匹配，三组修复均已部署 POC 并同步 fork 正式 KQL Function。三个非必需 Mashup 指标按 ADR-009 → **GW-009 Deferred**，仍为已知 null；GW-005 的非必需字段覆盖评估不在本轮范围。**不保证端到端文件级完整性、长期稳定性或历史 null 回填** |
| **P2 — Collector Reliability** | 文件/记录/批次可对账，轮转、部分失败、重试、去重、资源开销及长时间运行通过 | **当前阶段：准备只读基线与证据缺口分析，尚未可靠性验收** |
| **P3 — PROD Architecture** | Upload Manifest 方案、保留/回放、Event-time Backfill、多节点、安全与监控达成技术决策 | 规划中 |
| **P4 — Limited PROD Pilot** | 受控部署、验证看板、Rollback、运维交接及实际验收 | 尚未批准 |

### PROD 进入条件（必须逐项验证）

- [x] **P1 功能 POC**：按确认的必需字段范围，GW-001 Streaming / Aggregation / Mashup Raw → Parsed 映射与新记录数值通过；GW-009 三项非必需指标明确延期。**这不等于 PROD 级数据完整性通过**。
- [ ] 证实源 CSV → Eventstream → Raw → Parsed 无不可解释的遗漏/重复。
- [ ] 活动文件写入、旋转、断网重试及部分 Batch 故障有可靠测试。
- [ ] 确定 Manifest、Retention、Replay、按真实事件时间 Backfill 的方案。
- [ ] 确定 QueryText / DataSource / Secret 的权限和保留策略。
- [ ] 确定多节点隔离、告警、容量成本、运维与回滚程序。
- [ ] 完成 PROD 技术验收后才批准上线。

相关：[问题台账](ISSUES.md) · [决策](DECISIONS.md) · [架构](../docs/ARCHITECTURE_OPERATIONS.md)。
