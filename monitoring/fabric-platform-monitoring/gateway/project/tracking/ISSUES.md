# 问题与风险台账

> 更新：2026-11-17（Asia/Singapore）｜当前由我们维护，**不设责任人字段**。一个问题使用固定 ID；详细证据留在 `docs/`、`evidence/`，此处只记录状态与下一动作。

| ID | 优先级 | 状态 | 首次记录 | 最后复核 | 问题 / 风险 | 下一动作 | 证据 |
| --- | --- | --- | --- | --- | --- | --- | --- |
| **GW-001** | P1 | **验证中** | 2026-11-17 | 2026-11-17 | KQL Parser 的 Raw Key 错配导致 Parsed 指标全 null | Schema 39/39、数值转换 13 项零丢失/零超阈值差异；DataSource 235/235 内容一致；用本地 MCP 做只读现场 Preflight，人工审核后逐个 POC 修复 | [字段审计](../docs/SCHEMA_FIELD_MAPPING_AUDIT.md) · [Dry-run](../evidence/2026-11-17/DRYRUN_RESULTS.md) |
| **GW-002** | P1 | 待验证 | 2026-11-12 | 2026-11-17 | 活动日志 Move、部分 Batch 失败/重试、重复/遗漏风险 | 文件/行数/批次对账与故障测试 | [文件生命周期](../docs/ARCHITECTURE_OPERATIONS.md) |
| **GW-003** | P1 | 待决策 | 2026-11-17 | 2026-11-17 | Raw 仅保留 1 天，回放及长期审计受限 | 评估 Retention / Archive 成本与恢复策略 | [Eventhouse Overview](../evidence/2026-11-17/EventhouseOverview.MCP.sanitized.md) |
| **GW-004** | P2 | 设计中 | 2026-11-17 | 2026-11-17 | PROD 需 Upload Manifest 追踪 File、Hash、Rows、Batch Status | 确定快照、幂等与持久化设计 | [设计草案](../docs/PROD_UPLOAD_MANIFEST_DESIGN.md) |
| **GW-005** | P2 | 待决策 | 2026-11-17 | 2026-11-17 | Parsed Tables 没有保留全部 Raw 字段；Mashup 103 条 Raw 中 `DataSourceKinds/Paths` 两个原始 Key 均 0/103 | 明确长期分析需全量字段还是标准化指标，先核对 Mashup 的 DataSource 结构，不能凭空映射 | [字段矩阵](../docs/SCHEMA_FIELD_MAPPING_AUDIT.md) |
| **GW-006** | P2 | 待验证 | 2026-11-13 | 2026-11-17 | 多节点隔离与 PROD 首轮 Backfill 边界 | 按事件时间和独立 Staging 规划 | [架构运维](../docs/ARCHITECTURE_OPERATIONS.md) |
| **GW-007** | P2 | 待验证 | 2026-11-17 | 2026-11-17 | QueryText、DataSource、Secrets 与安全审计 | PROD 之前开展权限与隐私评估 | [架构运维](../docs/ARCHITECTURE_OPERATIONS.md) |
| **GW-008** | P3 | 待验证 | 2026-11-17 | 2026-11-17 | CopyExecutionReport 暂无源样本；部分 Function DocString 错误 | 产生对应日志后验证，文字错误后续清理 | [Eventhouse Overview](../evidence/2026-11-17/EventhouseOverview.MCP.sanitized.md) |

**状态定义：** 验证中 = 已有候选方案但未最终通过；待验证 = 尚无合格的实测结果；待决策 = 需要权衡方案；设计中 = 尚未实施；已关闭 = 附日期、验证证据及结果。不要根据“脚本执行成功”提前关闭问题。

如将来真正使用 GitHub Issues，只保留一个权威状态来源，本文件改为索引，避免双重维护。
