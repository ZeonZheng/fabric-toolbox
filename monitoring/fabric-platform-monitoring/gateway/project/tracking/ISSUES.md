# 问题与风险台账

> 更新：2026-11-20（Asia/Singapore）｜当前由我们维护，**不设责任人字段**。一个问题使用固定 ID；详细证据留在 `docs/`、`evidence/`，此处只记录状态与下一动作。

| ID | 优先级 | 状态 | 首次记录 | 最后复核 | 问题 / 风险 | 下一动作 | 证据 |
| --- | --- | --- | --- | --- | --- | --- | --- |
| **GW-001** | P1 | **Streaming/Aggregation Passed；Mashup 需业务范围决策（未部署）** | 2026-11-17 | 2026-11-23 | Mashup 2026-11-23 Shape Follow-up：Raw 36 行，TotalBytes 44/44、TotalRows 35/35 元素可转整数，数组同现 6/6 等长；MaxIO 12/14 含小数；现 Candidate 三字段依旧丢值、不能部署 | 先核实当前部署 Report/SM 是否消费三个字段及 POC 范围；如必需，再向微软确认数组语义与精度要求；无需重复相同形状统计 | [Nov23 Review](../evidence/2026-11-23/MASHUP_SHAPE_FOLLOWUP_REVIEW.md) · [Options](../docs/MASHUP_DESIGN_OPTIONS.md) |
| **GW-002** | P1 | 待验证 | 2026-11-12 | 2026-11-17 | 活动日志 Move、部分 Batch 失败/重试、重复/遗漏风险 | 文件/行数/批次对账与故障测试 | [文件生命周期](../docs/ARCHITECTURE_OPERATIONS.md) |
| **GW-003** | P1 | 待决策 | 2026-11-17 | 2026-11-17 | Raw 仅保留 1 天，回放及长期审计受限 | 评估 Retention / Archive 成本与恢复策略 | [Eventhouse Overview](../evidence/2026-11-17/EventhouseOverview.MCP.sanitized.md) |
| **GW-004** | P2 | 设计中 | 2026-11-17 | 2026-11-17 | PROD 需 Upload Manifest 追踪 File、Hash、Rows、Batch Status | 确定快照、幂等与持久化设计 | [设计草案](../docs/PROD_UPLOAD_MANIFEST_DESIGN.md) |
| **GW-005** | P2 | **Schema / 下游消费待确认** | 2026-11-17 | 2026-11-23 | Streaming Raw 三字段未投影；Mashup Kinds/Paths 原 Raw 无对应 Key，TotalBytes/Rows 多值数组 ↔ 标量、MaxIO 小数 ↔ long 冲突。版本化 Fabric SM/Report/Dashboard 未发现对这三个问题字段的直接引用（部署现场/二进制 PBIT 未核实） | 核实实际下游依赖与业务需求，随后决策保持范围/语义正确的标量/无损明细；未经定义不变更 Schema | [Nov23 Review](../evidence/2026-11-23/MASHUP_SHAPE_FOLLOWUP_REVIEW.md) · [Options](../docs/MASHUP_DESIGN_OPTIONS.md) |
| **GW-006** | P2 | 待验证 | 2026-11-13 | 2026-11-17 | 多节点隔离与 PROD 首轮 Backfill 边界 | 按事件时间和独立 Staging 规划 | [架构运维](../docs/ARCHITECTURE_OPERATIONS.md) |
| **GW-007** | P2 | 待验证 | 2026-11-17 | 2026-11-17 | QueryText、DataSource、Secrets 与安全审计 | PROD 之前开展权限与隐私评估 | [架构运维](../docs/ARCHITECTURE_OPERATIONS.md) |
| **GW-008** | P3 | **待复核** | 2026-11-17 | 2026-11-20 | CopyExecutionReport 无源样本；部分 Parser DocString；2026-11-20 Preflight 报告称 `parse_QueryStartReport_csv` Folder 有差异，但另有全库 Folder 一致的相反表述 | 单独用实际 `.show functions` 复核该元数据差异，不影响本次目标三个 Parser 的 Body | [Preflight](../evidence/2026-11-20/PRE_DEPLOYMENT_REVIEW.md) |

**状态定义：** 验证中 = 已有候选方案但未最终通过；待验证 = 尚无合格的实测结果；待决策 = 需要权衡方案；设计中 = 尚未实施；已关闭 = 附日期、验证证据及结果。不要根据“脚本执行成功”提前关闭问题。

如将来真正使用 GitHub Issues，只保留一个权威状态来源，本文件改为索引，避免双重维护。
