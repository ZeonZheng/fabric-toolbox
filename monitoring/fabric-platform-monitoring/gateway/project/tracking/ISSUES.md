# 问题与风险台账

> 更新：2026-11-23（Asia/Singapore）｜当前由我们维护，**不设责任人字段**。一个问题使用固定 ID；详细证据留在 `docs/`、`evidence/`，此处只记录状态与下一动作。

| ID | 优先级 | 状态 | 首次记录 | 最后复核 | 问题 / 风险 | 下一动作 | 证据 |
| --- | --- | --- | --- | --- | --- | --- | --- |
| **GW-001** | P1 | **Streaming/Aggregation Passed；Mashup 必需字段待验收（Candidate 未部署）** | 2026-11-17 | 2026-11-23 | 三个非必需 Mashup 指标已明确排除当前 P1 验收并转 GW-009，**不是数据已修好**；其他需用的 Mashup Memory/Time/CPU/IO 字段是否正常仍须按范围验证 | 核对 P1 必需指标与现有 Parsed 新数据；仅在确有缺陷时准备最小范围 Candidate、Dry-run、回归和受控变更；不部署旧 Candidate | [决策](DECISIONS.md) · [Nov23 Review](../evidence/2026-11-23/MASHUP_SHAPE_FOLLOWUP_REVIEW.md) |
| **GW-002** | P1 | 待验证 | 2026-11-12 | 2026-11-17 | 活动日志 Move、部分 Batch 失败/重试、重复/遗漏风险 | 文件/行数/批次对账与故障测试 | [文件生命周期](../docs/ARCHITECTURE_OPERATIONS.md) |
| **GW-003** | P1 | 待决策 | 2026-11-17 | 2026-11-17 | Raw 仅保留 1 天，回放及长期审计受限 | 评估 Retention / Archive 成本与恢复策略 | [Eventhouse Overview](../evidence/2026-11-17/EventhouseOverview.MCP.sanitized.md) |
| **GW-004** | P2 | 设计中 | 2026-11-17 | 2026-11-17 | PROD 需 Upload Manifest 追踪 File、Hash、Rows、Batch Status | 确定快照、幂等与持久化设计 | [设计草案](../docs/PROD_UPLOAD_MANIFEST_DESIGN.md) |
| **GW-005** | P2 | **Schema 覆盖缺口待评估** | 2026-11-17 | 2026-11-23 | Streaming Raw 三字段未投影；Mashup Kinds/Paths 原 Raw 无对应 Key。Mashup 三个非必需指标的类型/语义冲突独立转 GW-009 | 结合实际业务消费决定其余 Schema 覆盖要求；不为无源 Key 或非必需指标强制变更 Schema | [Nov23 Review](../evidence/2026-11-23/MASHUP_SHAPE_FOLLOWUP_REVIEW.md) · [决策](DECISIONS.md) |
| **GW-009** | P3 | **Deferred / 当前非阻塞** | 2026-11-23 | 2026-11-23 | `TotalBytes`、`TotalRows` 为 JSON 多元素数组，当前标量转换全 null；`MaxIODataBytesPerSecond` 含小数，与 long 类型不兼容。三个指标当前 POC/计划中 PROD 监控非必需，保留数据完整性缺口，不视为修复 | 暂不解析/修改/联系微软；仅当明确有下游需求时复核实际消费、数组元素语义、精度、Schema 和迁移方案 | [决策 ADR-009](DECISIONS.md) · [设计选项](../docs/MASHUP_DESIGN_OPTIONS.md) · [证据](../evidence/2026-11-23/MASHUP_SHAPE_FOLLOWUP_REVIEW.md) |
| **GW-006** | P2 | 待验证 | 2026-11-13 | 2026-11-17 | 多节点隔离与 PROD 首轮 Backfill 边界 | 按事件时间和独立 Staging 规划 | [架构运维](../docs/ARCHITECTURE_OPERATIONS.md) |
| **GW-007** | P2 | 待验证 | 2026-11-17 | 2026-11-17 | QueryText、DataSource、Secrets 与安全审计 | PROD 之前开展权限与隐私评估 | [架构运维](../docs/ARCHITECTURE_OPERATIONS.md) |
| **GW-008** | P3 | **待复核** | 2026-11-17 | 2026-11-20 | CopyExecutionReport 无源样本；部分 Parser DocString；2026-11-20 Preflight 报告称 `parse_QueryStartReport_csv` Folder 有差异，但另有全库 Folder 一致的相反表述 | 单独用实际 `.show functions` 复核该元数据差异，不影响本次目标三个 Parser 的 Body | [Preflight](../evidence/2026-11-20/PRE_DEPLOYMENT_REVIEW.md) |

**状态定义：** 验证中 = 已有候选方案但未最终通过；待验证 = 尚无合格的实测结果；待决策 = 需要权衡方案；设计中 = 尚未实施；已关闭 = 附日期、验证证据及结果。不要根据“脚本执行成功”提前关闭问题。

如将来真正使用 GitHub Issues，只保留一个权威状态来源，本文件改为索引，避免双重维护。
