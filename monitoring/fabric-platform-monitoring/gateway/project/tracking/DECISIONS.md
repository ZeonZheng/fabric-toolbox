# 技术决策记录（轻量 ADR）

> 更新：2026-11-23（Asia/Singapore）。**决策 ≠ 已实现**；保留上下文和约束，不为单人项目增加复杂审批流程。

| ADR | 日期 | 决策 | 状态与理由 |
| --- | --- | --- | --- |
| ADR-001 | 2026-11-17 | **GitHub 作为工程 Source of Truth** | 已采纳：Eventhouse 运行修改必须对应可审查、可回滚的仓库资产 |
| ADR-002 | 2026-11-17 | 我们的资产集中在 `gateway/project/`，上游文件保持原路径 | 已实施：便于识别项目边界与 Upstream Diff |
| ADR-003 | 2026-11-17 | 先做只读 Parser 验证，再决定 POC Function DDL | 已实施：Streaming 与 Aggregation 通过完整只读检查后在 POC 手动部署并验证；Mashup 检查 Blocked，不予部署 |
| ADR-004 | 2026-11-17 | PROD 方案考虑 Upload Manifest | 已进入设计 Backlog：不是已实现功能 |
| ADR-005 | 2026-11-13 | 原始 Move Uploader 仅在受控 POC 观察；PROD 首轮使用独立 Staging 与事件时间 Cutoff | POC 决策，尚未完成可靠性测试 |
| ADR-006 | 2026-11-17 | 首轮修复尽量保持现有 Parsed Table 结构；全字段长期留存另议 | 暂定：降低 Report / Model 兼容风险 |
| ADR-007 | 2026-11-17 | 单人项目精简管理，不设 Owner / Assignee；项目说明统一中文 | 已采纳：入口清晰、追踪可审计，保持维护成本低 |
| ADR-008 | 2026-11-20 | 按 Streaming → Aggregation → Mashup 顺序，单个 Parser POC 受控部署 | **已完成本轮单函数 POC 验收（2026-11-23）**：Streaming、Aggregation 新数据 Passed；Mashup 在首轮 0 新 Raw Inconclusive 后，新 Raw/Parsed 各 4 条、五项指标零 mismatch、Policy failures 0 observed，功能 Passed。三项已同步 fork 正式 KQL Function；不涵盖 Collector 可靠性 |
| ADR-009 | 2026-11-23（范围决定） | **采纳方案 A：** `TotalBytes`、`TotalRows`、`MaxIODataBytesPerSecond` 对当前 Gateway Monitoring POC/计划中的 PROD 监控不是必需指标，转为独立延期 Backlog（GW-009）；不阻塞当前 P1 必需指标验收 | **已采纳，仅为范围决定，非技术修复：** 三字段已知 null/数组语义及精度问题保留；不擅自 SUM/FIRST/截断。**本决定当时没有改动现场**；后来按该范围定义手工部署了仅 P1 必需字段的 Mashup Candidate（2026-11-23），随后新 Raw/Parsed 4/4 的 Postcheck 已通过；表 Schema、Policy 与 PROD 仍未改。历史原 Candidate 仍保留作审计；未来有实际需求时再评估微软定义与 Schema。见 [选项](../docs/MASHUP_DESIGN_OPTIONS.md) |

详细理由参见 [架构与运维](../docs/ARCHITECTURE_OPERATIONS.md)、[Parser 审计](../docs/PARSER_SCHEMA_AUDIT.md)、[Manifest 设计](../docs/PROD_UPLOAD_MANIFEST_DESIGN.md)。
