# 项目阶段与验收关口

> 更新：2026-11-23（Asia/Singapore）。此表记录阶段，不虚构预计完成日期；每次阶段结束必须有可复现证据。

| 阶段 | 关键交付 / 通过条件 | 状态 |
| --- | --- | --- |
| **P0 — Functional POC** | Gateway → Fabric → Report 链路、8 种 Report、Heartbeat / NodeInfo 功能可用 | **功能性验收完成**；不含可靠性保证 |
| **P1 — Data Correctness** | **当前必需** Raw 字段正确映射；Schema、数值、新数据回归合格；Patch/Rollback 可复现；被明确排除的字段保留独立风险记录 | **本轮 P1 必需 Parser 功能性 POC 验收完成（2026-11-23）**：Streaming 32/32、Aggregation 新 Raw/Parsed 各 7 行（复合 Key 2/2）、Mashup 4/4 新 Raw/Parsed 五项指标零不匹配，三组修复均已部署 POC 并同步 fork 正式 KQL Function。三个非必需 Mashup 指标按 ADR-009 → **GW-009 Deferred**，仍为已知 null；GW-005 的非必需字段覆盖评估不在本轮范围。**不保证端到端文件级完整性、长期稳定性或历史 null 回填** |
| **P2 — Collector Reliability / Pilot Readiness** | 核心字段覆盖确认、连续数据新鲜度/源→Raw 合理对账、活动文件处理与 Gateway 资源影响；区分阻塞 PROD 的丢失风险与可延期的异常重复 | **正在进行，重点调整：** 已观察 8 类 Raw、任务持续运行、P1 必需 Parser Passed。P2-T2 Mock 重发风险单独拆 GW-010 Deferred，不将全套 Batch 幂等/Manifest 改造作为**有限、受监控 PROD Pilot**的默认前置。仍需验证持续采集、异常告警及可接受数据质量；GW-002 Open |
| **P3 — PROD Architecture / Readiness** | 定义实际需要的 Report 字段、Raw Retention/回放、节点隔离、初次 Backfill、QueryText/权限、安全与成本、监测/回滚；完整 Manifest 为可延期增强 | **下一并行重点：** 先形成并执行 [有限 PROD Pilot 前最小检查](../docs/PROD_READINESS_GAP_20261123.md)；GW-010/GW-004 暂缓，P4 未获批准 |
| **P4 — Limited PROD Pilot** | 受控部署、验证看板、Rollback、运维交接及实际验收 | 尚未批准 |

### PROD 进入条件（必须逐项验证）

- [x] **P1 功能 POC**：按确认的必需字段范围，GW-001 Streaming / Aggregation / Mashup Raw → Parsed 映射与新记录数值通过；GW-009 三项非必需指标明确延期。**这不等于 PROD 级数据完整性通过**。
- [ ] 在受控代表性窗口确认源 CSV → Eventstream → Raw → Parsed 的**关键数据无不可解释的静默遗漏**及新鲜度；异常重发去重（GW-010）记录为当前非精确审计场景的已知延期风险，不声称 Exactly-once。
- [ ] 验证活动文件处理/轮转不会明显影响 Gateway 正常业务、资源占用可接受，并有任务故障/摄入失败告警与退出方案；部分 Batch 重发异常已在 Mock 复现并按 ADR-010 延期强化。
- [ ] 明确 Raw Retention/Replay 的 Pilot 可接受边界，以及按真实事件时间初次 Backfill 策略；**完整 Upload Manifest（GW-004）作为可延期增强，不是有限 Pilot 默认硬前提**。
- [ ] 确定 QueryText / DataSource / Secret 的权限和保留策略。
- [ ] 确定多节点隔离、告警、容量成本、运维与回滚程序。
- [ ] 完成 PROD 技术验收后才批准上线。

相关：[问题台账](ISSUES.md) · [决策](DECISIONS.md) · [架构](../docs/ARCHITECTURE_OPERATIONS.md)。
