# Gateway Monitoring｜当前状态

> **更新：** 2026-11-20（Asia/Singapore）｜**阶段：** P1 Parser 受控 POC 修复准备｜**环境：** 单节点 POC（执行写操作前仍需在 UI 人工核实）｜**PROD：** 未批准

| 工作项 | 状态 | 实测依据与边界 |
| --- | --- | --- |
| Gateway → Fabric → Report 功能 | **已通过 POC** | 8 种 Report logType 均见 Raw、Heartbeat 单节点；不证明无丢失/无重复 |
| Uploader Move/合并可靠性 | **持续观察中** | 未完成文件级/批次级对账，PROD 的 Manifest、Rotation、去重问题仍开放 |
| Parser 字段修复（GW-001） | **只读验证通过；未部署** | 3/3 Dry-run、39/39 Schema、13 项数值/单位转换、235/235 DataSource 内容等价已通过 |
| 2026-11-20 MCP Preflight | **4/4 PASS（仅进入人工 Review）** | 3/3 Function SHA256 精确匹配、3/3 Update Policies 合格、Raw/Parsed 6/6 有数据、最近 2h ingest failures 0 observed |
| 当前 Parser 新记录 | **仍有缺陷** | Aggregation 6 个 Duration、Streaming 2 个 Duration 仍 100% null；Mashup TotalBytes 2797/2797 null |
| POC Controlled Deployment | **候选及回滚就绪，未执行 DDL** | 建议先改 Streaming 一个 Function，检查新摄入后再继续 Aggregation / Mashup |
| Mashup 和 Streaming 额外字段 | **已登记未关闭** | Mashup TotalBytes 需独立候选回归；Streaming 3 个 Raw-only 字段目前未进入 Parsed |
| 长期生产治理 | **尚未验收** | Raw 保留 1 天；无原始文件/批次对账、幂等、Multi-Node、隐私与 PROD Go/No-Go |

## 最近现场证据：2026-11-20 10:36–10:47（UTC+8）

- 本地 AI 通过 `eventhouse-remote.executeQuery` **只读执行 4/4**；未发生 Function/Policy/Uploader 写操作。
- 函数：`parse_QueryExecutionAggregationReport`、`parse_QueryStreamingReport`、`parse_MashupEvaluationReport` 的**当前 Body** 与 2026-11-17 基线 **3/3 SHA256 完全一致**。
- Update Policies：3/3 均 `Enabled=true`、`Source=GatewayReports-Raw`、`IsTransactional=true`；最近 2 小时失败 **0 observed**（只能说明观察窗口内未检测到失败，静默 null 不计）。
- Raw 和 Parsed 各有活跃数据；报告额外观察到 Mashup TotalBytes 全 null、Streaming 3 个未投影 Raw 字段。
- **报告内一处不一致：** `parse_QueryStartReport_csv` Folder 的现场 vs baseline 存在差异，但报告另一节写“15/15 Folder 一致”；需单独确认，不影响这次 3/3 目标函数 Body 一致性。
- [2026-11-20 脱敏报告与独立评审](evidence/2026-11-20/README.md)。

## 当前唯一下一步

**由用户亲自在 Eventhouse UI 核实 POC 工作区/数据库身份后，依照 [Streaming 简明操作步骤（人手动执行写操作）](kql/poc-parser-fix-20261117/controlled-rollout/README.md) 开始第一轮修复；部署后的三份只读查询交给 [本地 MCP 检查 Prompt](kql/poc-parser-fix-20261117/controlled-rollout/POSTCHECK_AI_PROMPT.md)。** 只操作 `parse_QueryStreamingReport`，记录准确 UTC 变更时间；运行预先准备的只读新数据验证与 ingestion failure 检查。出现问题使用对应原 Function Body 回滚。**不可由本地 MCP 自动执行 DDL，也不可同批修改三个 Function。**

后续：Aggregation → Mashup；**Mashup 发布前先运行** [TotalBytes 等额外指标只读检查](kql/poc-parser-fix-20261117/value-checks/3-mashup-additional-metrics.kql)。Streaming Raw-only 3 字段和 Mashup DataSourceKinds/Paths 属于独立 Schema 设计问题，本次不改表。

**证据解释：** Function Patch 只影响新摄入记录，既有 Parsed null 不自动修复；历史回填与 Uploader 端到端完整性仍未完成。

问题索引：[GW-001/GW-005](tracking/ISSUES.md) · [2026-11-20 Preflight 评审](evidence/2026-11-20/PRE_DEPLOYMENT_REVIEW.md)。
