# 项目阶段与验收关口

> 更新：2026-11-20（Asia/Singapore）。此表记录阶段，不虚构预计完成日期；每次阶段结束必须有可复现证据。

| 阶段 | 关键交付 / 通过条件 | 状态 |
| --- | --- | --- |
| **P0 — Functional POC** | Gateway → Fabric → Report 链路、8 种 Report、Heartbeat / NodeInfo 功能可用 | **功能性验收完成**；不含可靠性保证 |
| **P1 — Data Correctness** | 所需 Raw 字段正确映射；Schema、数值与新数据回归合格；Patch 与 Rollback 可复现 | **2/3 Parser POC 功能 Passed**：Streaming 32/32、Aggregation Raw 7/Parsed 7（2/2 Key）；Mashup **Blocked**（TotalBytes 243/243、TotalRows 34/34、MaxIO 127/127 原 Candidate 转换为 null），需先调查 JSON 数组多元素语义和 MaxIO 小数/long Schema 冲突。历史 null/长时间稳定性未验收 |
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
