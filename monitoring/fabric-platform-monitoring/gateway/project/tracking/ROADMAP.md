# 项目阶段与验收关口

> 更新：2026-11-17（Asia/Singapore）。此表记录阶段，不虚构预计完成日期；每次阶段结束必须有可复现证据。

| 阶段 | 关键交付 / 通过条件 | 状态 |
| --- | --- | --- |
| **P0 — Functional POC** | Gateway → Fabric → Report 链路、8 种 Report、Heartbeat / NodeInfo 功能可用 | **功能性验收完成**；不含可靠性保证 |
| **P1 — Data Correctness** | 所需 Raw 字段均正确映射；Schema、数值、回归合格；KQL 修复与 Rollback 可复现 | **进行中**：Dry-run 3/3、Schema 39/39、数值/单位样本已通过；DataSource 语义、POC 部署仍待验证 |
| **P2 — Collector Reliability** | 文件/记录/批次可对账，轮转、部分失败、重试、去重、资源开销及长时间运行通过 | 未完成 |
| **P3 — PROD Architecture** | Upload Manifest 方案、保留/回放、Event-time Backfill、多节点、安全与监控达成技术决策 | 规划中 |
| **P4 — Limited PROD Pilot** | 受控部署、验证看板、Rollback、运维交接及实际验收 | 尚未批准 |

### PROD 进入条件（必须逐项验证）

- [ ] 关闭 GW-001：修复 Raw → Parsed 字段映射并验证新记录。
- [ ] 证实源 CSV → Eventstream → Raw → Parsed 无不可解释的遗漏/重复。
- [ ] 活动文件写入、旋转、断网重试及部分 Batch 故障有可靠测试。
- [ ] 确定 Manifest、Retention、Replay、按真实事件时间 Backfill 的方案。
- [ ] 确定 QueryText / DataSource / Secret 的权限和保留策略。
- [ ] 确定多节点隔离、告警、容量成本、运维与回滚程序。
- [ ] 完成 PROD 技术验收后才批准上线。

相关：[问题台账](ISSUES.md) · [决策](DECISIONS.md) · [架构](../docs/ARCHITECTURE_OPERATIONS.md)。
