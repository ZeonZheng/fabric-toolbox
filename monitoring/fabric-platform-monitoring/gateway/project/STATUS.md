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
| 数值及时间单位 | **当前 Raw 样本通过** | 15 项检查中数值项 13 项无转换丢失/超过阈值变化；Mashup 毫秒 → timespan 回算正常 |
| DataSource 语义等价性 | **已通过（适用的两种类型）** | Aggregation 96/96、Streaming 139/139，`DifferentSerialized=0`；Mashup Kinds/Paths 独立 Raw Key 均 0/103，保留单独风险 |
| POC Function 修复 | **准备受控部署评审（尚未执行）** | GitHub 有 3 个 Candidate 与 Rollback；需现场 Preflight 与逐一部署/验证，严禁自动 DDL |
| PROD 设计 | **未完成** | Upload Manifest、重试/去重、多节点、保留策略、Backfill、隐私/告警 |

### 最近一次实测（2026-11-17 15:40，UTC+8）

| Dry-run | Raw Rows | 恢复非空的指标 |
| --- | ---: | --- |
| QueryExecutionAggregationReport | 77 | Query Duration 三列各 77；Data Processing Duration 三列各 56 |
| QueryStreamingReport | 113 | 两列各 104 |
| MashupEvaluationReport | 91 | 五列各 91 |

字段读取恢复非空及 Schema 对照均通过；尚未证明数值/单位准确或完整无丢失。

### 当前唯一下一步

用本机已连接的 **Eventhouse Remote MCP**，让本地 AI 执行 [只读 Preflight 套件](kql/poc-parser-fix-20261117/mcp-handoff/README.md)：重新读取 3 个当前 Function Body、Update Policies、Raw/Parsed 最新情况与最近 2 小时失败记录；与本分支部署基线核对并返回**脱敏中文报告**。完成现场确认和人工审查后，再决定是否**逐个**应用 POC Parser Patch；**MCP 不得执行 DDL**。

**DataSource 对照已通过（2026-11-17 17:11 UTC+8）：** Aggregation 96/96、Streaming 139/139，Legacy 与 Candidate 全部内容一致（`DifferentSerialized=0`），类型均观察到 `array/string`。Mashup 103 条 Raw 中 `DataSourceKinds`、`DataSourcePaths` 两个 Key 出现 **0 次**，不属于本轮字段映射修复可凭空填充的字段。完整 [CSV](evidence/2026-11-17/DataSourceCheck-LegacyVsCandidate.csv)。该项不代表所有日志字段已完整保存。

**Value Check 已完成（2026-11-17 16:54 UTC+8）：** 103 Mashup + 96 Aggregation + 139 Streaming = 338 条 Raw 记录的 15 项（13 数值 + 2 DataSource 非空率）统计；`ConversionLoss` 全 0，13 个数值项 `ChangedByCast` 全 0（阈值 0.0001），原值与 Candidate 非空次数一致。详情：[CSV](evidence/2026-11-17/ValueCheck-ParserConversions.csv)。

**Schema Check 已完成（2026-11-17 16:37～16:38 UTC+8）：** 13/13 + 9/9 + 17/17 = **39/39 列 IsMatch=true**，列名/类型/顺序一致，见 [导出证据](evidence/2026-11-17/README.md)。

相关记录：[问题 GW-001](tracking/ISSUES.md) · [本轮 Dry-run 证据](evidence/2026-11-17/DRYRUN_RESULTS.md) · [部署 Schema 备份](evidence/2026-11-17/DeployedTableSchema.sanitized.csv)。

注：Raw 默认仅保留 1 天，不能用当前 Raw 全量与 3650 天 Parsed 全量作直接计数比较。
