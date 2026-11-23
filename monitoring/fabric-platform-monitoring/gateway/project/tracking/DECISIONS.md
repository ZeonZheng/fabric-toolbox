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
| ADR-008 | 2026-11-20 | 按 Streaming → Aggregation → Mashup 顺序，单个 Parser POC 受控部署 | **部分实施**：Streaming、Aggregation 已手工部署并新数据 Passed；Mashup 因类型/数组转换 Blocked，未部署 |
| ADR-009 | 2026-11-20（2026-11-23 复核） | Mashup JSON 数组字段与 MaxIO 小数必须先确认业务必要性及语义，再选择实现范围 | **未决策：** 44/44 + 35/35 元素可转整数、共同出现数组 6/6 等长，但不证明可 SUM/FIRST；MaxIO 12/14 有小数且目标为 long；Fork Fabric Report/SM 未发现三个字段的直接引用。见 [选项](../docs/MASHUP_DESIGN_OPTIONS.md) |

详细理由参见 [架构与运维](../docs/ARCHITECTURE_OPERATIONS.md)、[Parser 审计](../docs/PARSER_SCHEMA_AUDIT.md)、[Manifest 设计](../docs/PROD_UPLOAD_MANIFEST_DESIGN.md)。
