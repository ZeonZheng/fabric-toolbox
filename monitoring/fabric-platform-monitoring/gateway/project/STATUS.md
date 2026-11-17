# Gateway Monitoring｜当前状态

> **更新：** 2026-11-17（Asia/Singapore）｜**阶段：** P1 数据解析准确性｜**运行环境：** POC 单节点｜**PROD：** 未批准

| 工作项 | 现状 | 证据与边界 |
| --- | --- | --- |
| Fabric 端到端功能 | **已通过 POC** | Gateway VM → Eventstream → Eventhouse → Semantic Model → Report |
| NodeInfo / Heartbeat | **功能已验证** | 曾确认定时任务与 KQL 有数据；不是长期 SLA |
| 原始 Move Uploader | **运行观察中** | VM 快照 42 个 Report 文件（History 40、Source 2）；无遗漏/无重复未验收 |
| Raw 日志覆盖 | **部分验证** | 8 类 Performance CSV + 独立的 GatewayNodeInfo；尚无源文件级完整对账 |
| KQL Parser 缺陷（GW-001） | **根因已确认，Dry-run 通过** | Raw 有数值，Parsed 三项指标全 null；修复候选 3/3 可从 Raw 读取非空值 |
| 部署前备份 | **已完成现有材料的基线** | 15 Function Bodies、142 行 Schema、Policy 快照、MCP Overview；非整库物理备份 |
| Schema / 数值准确性 | **未验证（当前关口）** | 目标表列名、类型、顺序及 DataSource / timespan 等仍需逐项对比 |
| POC Function 修复 | **未执行** | GitHub 中仅有候选 Patch 与 Rollback，运行中的 Eventhouse 未修改 |
| PROD 设计 | **未完成** | Upload Manifest、重试/去重、多节点、保留策略、Backfill、隐私/告警 |

### 最近一次实测（2026-11-17 15:40，UTC+8）

| Dry-run | Raw Rows | 恢复非空的指标 |
| --- | ---: | --- |
| QueryExecutionAggregationReport | 77 | Query Duration 三列各 77；Data Processing Duration 三列各 56 |
| QueryStreamingReport | 113 | 两列各 104 |
| MashupEvaluationReport | 91 | 五列各 91 |

以上只证明**字段读取成功**，未证明 Schema 相容或数据入库链路不丢失。

### 当前唯一下一步

执行 [3 个只读 Schema Compatibility Check](kql/poc-parser-fix-20261117/schema/README.md)：对比候选输出与现有目标表的列名、列类型、顺序，每个 `IsMatch` 都应为 `true`。随后检查 Raw → Candidate 的样本值、`DataSource` 和 `TotalProcessorTime(ms)` → timespan 的单位。**这些检查完成前禁止运行 `candidates/` 中的 DDL。**

相关记录：[问题 GW-001](tracking/ISSUES.md) · [本轮 Dry-run 证据](evidence/2026-11-17/DRYRUN_RESULTS.md) · [部署 Schema 备份](evidence/2026-11-17/DeployedTableSchema.sanitized.csv)。

注：Raw 默认仅保留 1 天，不能用当前 Raw 全量与 3650 天 Parsed 全量作直接计数比较。
