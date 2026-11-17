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
| Schema 兼容性 | **已通过 39/39** | Aggregation 13/13、Streaming 9/9、Mashup 17/17；CSV 已归档 |
| 数值、单位及 DataSource | **待验证（当前关口）** | 原值转换、long 精度、TotalProcessorTime 单位及 DataSource 语义 |
| POC Function 修复 | **未执行** | GitHub 中仅有候选 Patch 与 Rollback，运行中的 Eventhouse 未修改 |
| PROD 设计 | **未完成** | Upload Manifest、重试/去重、多节点、保留策略、Backfill、隐私/告警 |

### 最近一次实测（2026-11-17 15:40，UTC+8）

| Dry-run | Raw Rows | 恢复非空的指标 |
| --- | ---: | --- |
| QueryExecutionAggregationReport | 77 | Query Duration 三列各 77；Data Processing Duration 三列各 56 |
| QueryStreamingReport | 113 | 两列各 104 |
| MashupEvaluationReport | 91 | 五列各 91 |

字段读取恢复非空及 Schema 对照均通过；尚未证明数值/单位准确或完整无丢失。

### 当前唯一下一步

运行 [只读数值/单位验证 KQL](kql/poc-parser-fix-20261117/value-checks/README.md)：检查 Raw 非空到类型转换的损失、long 精度、TotalProcessorTime(ms) → timespan 及 DataSource 非空率。后续仍要确认 DataSource 语义；**勿运行 candidates/ 的 DDL**。

**Schema Check 已完成（2026-11-17 16:37～16:38 UTC+8）：** 13/13 + 9/9 + 17/17 = **39/39 列 IsMatch=true**，列名/类型/顺序一致，见 [导出证据](evidence/2026-11-17/README.md)。

相关记录：[问题 GW-001](tracking/ISSUES.md) · [本轮 Dry-run 证据](evidence/2026-11-17/DRYRUN_RESULTS.md) · [部署 Schema 备份](evidence/2026-11-17/DeployedTableSchema.sanitized.csv)。

注：Raw 默认仅保留 1 天，不能用当前 Raw 全量与 3650 天 Parsed 全量作直接计数比较。
