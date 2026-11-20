# 技术决策记录（轻量 ADR）

> 更新：2026-11-20（Asia/Singapore）。**决策 ≠ 已实现**；保留上下文和约束，不为单人项目增加复杂审批流程。

| ADR | 日期 | 决策 | 状态与理由 |
| --- | --- | --- | --- |
| ADR-001 | 2026-11-17 | **GitHub 作为工程 Source of Truth** | 已采纳：Eventhouse 运行修改必须对应可审查、可回滚的仓库资产 |
| ADR-002 | 2026-11-17 | 我们的资产集中在 `gateway/project/`，上游文件保持原路径 | 已实施：便于识别项目边界与 Upstream Diff |
| ADR-003 | 2026-11-17 | 先做只读 Parser 验证，再决定 POC Function DDL | 已采纳：Dry-run 通过，但 Schema/数值仍需验证；未部署 |
| ADR-004 | 2026-11-17 | PROD 方案考虑 Upload Manifest | 已进入设计 Backlog：不是已实现功能 |
| ADR-005 | 2026-11-13 | 原始 Move Uploader 仅在受控 POC 观察；PROD 首轮使用独立 Staging 与事件时间 Cutoff | POC 决策，尚未完成可靠性测试 |
| ADR-006 | 2026-11-17 | 首轮修复尽量保持现有 Parsed Table 结构；全字段长期留存另议 | 暂定：降低 Report / Model 兼容风险 |
| ADR-007 | 2026-11-17 | 单人项目精简管理，不设 Owner / Assignee；项目说明统一中文 | 已采纳：入口清晰、追踪可审计，保持维护成本低 |

详细理由参见 [架构与运维](../docs/ARCHITECTURE_OPERATIONS.md)、[Parser 审计](../docs/PARSER_SCHEMA_AUDIT.md)、[Manifest 设计](../docs/PROD_UPLOAD_MANIFEST_DESIGN.md)。
| ADR-008 | 2026-11-20 | **建议**先在 POC 单独部署 Streaming Parser，再逐一处理 Aggregation / Mashup | **待用户人工确认；尚未实施。** Preflight 4/4 通过，但三个函数都通过事务性 Update Policy 被实时使用；一轮只变一个函数易定位/回滚；Mashup 新增 TotalBytes 风险须先检查 |
