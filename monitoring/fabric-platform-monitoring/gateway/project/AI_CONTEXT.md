# AI / 技术交接说明

> 更新时间：2026-11-20 14:22（Asia/Singapore）。**先读 [项目总览](README.md) → [当前状态](STATUS.md) → [问题清单](tracking/ISSUES.md)**，然后按任务读取相关证据。

## 固定背景

- 当前 POC 是 **单 Gateway Cluster、单 Node**；项目路径 `monitoring/fabric-platform-monitoring/gateway/project/`；工作分支 `docs/gateway-monitoring-poc-20261112`。
- `project/` 是**我们的工作区**；Vendor PowerShell / Scheduler / Report 以及 `src/Gateway/` 仍留在上游位置，避免影响上游升级差异比对。已确认的 Heartbeat 修改详见 [UPSTREAM_CHANGES](docs/UPSTREAM_CHANGES.md)。
- Uploader 原始实现为 `Move → Temp → Eventstream → History`，8 种 Report CSV 已观察到；`HTTP 201` 不等于端到端 Exactly-once。
- KQL 已获得 15/15 实际部署 Function Body 和 142 行 Schema；部署 Body 与 GitHub 源码一致。3 个 Parser 的只读修复候选成功恢复非空，**Schema 39/39 已通过**，13 个数值转换项在当前 Raw 样本中零丢失、零超阈值差异；Aggregation / Streaming 235/235 条 DataSource 规范化值与 Legacy 一致；Mashup 103 条 Raw 无 Kinds/Paths 独立 Key。**2026-11-20 用户已手工执行 Streaming Function DDL；Aggregation / Mashup 尚未更改**。
- Eventhouse `GatewayReports-Raw` 默认 1 天保留，Parsed 大多 3650 天。40 个合并 History 文件 + 2 个 Source 文件不能证明全部记录均被上传。
- **PROD 尚未获准**；Upload Manifest 是候选设计，不是已实现能力。

## 入口与工程规则

| 目的 | 路径 |
| --- | --- |
| 唯一最新状态 / 下一步 | [STATUS.md](STATUS.md) |
| Issue / Phase / 决策 / 时间线 | [tracking/](tracking/ISSUES.md) |
| 技术 Runbook 与审计 | [docs/INDEX.md](docs/INDEX.md) |
| 原始来源与验证记录 | [evidence/README.md](evidence/README.md) |
| 只读查询 / 待审核补丁 / 回滚 | [kql/poc-parser-fix-20261117/README.md](kql/poc-parser-fix-20261117/README.md) |

1. **证据优先**：区分已验证、推断、待实施；附真实测试日期，不得把 HTTP 201 当作完整性证明。
2. **安全边界**：未经明确指示，不运行 Candidate DDL、覆盖式 Jumpstart、不对 PROD 操作；不得提交 QueryText、原始日志、Secrets、内部 Connection IDs。
3. **改动前**：核对 Live Function 与 GitHub 源码、目标 Schema、Update Policy（尤其是 Transactional），准备回滚。
4. **改动后**：只更新 [STATUS](STATUS.md)、对应 [ISSUES](tracking/ISSUES.md) 状态，以及 [CHANGELOG](tracking/CHANGELOG.md) 时间记录；需要技术细节才补充 docs/evidence。
5. **文档语言**：**我们的说明、项目管理、交接用中文**，保留 PowerShell/KQL/字段名称等代码与技术标识原文；上游文件和原始导出不翻译。

**当前交接点：** 用户已于 2026-11-20 在 POC 手工执行 `parse_QueryStreamingReport` Candidate，UTC 观察起点 `2026-11-20T06:19:18.0313089Z`。14:22 Postcheck **Inconclusive**：Patch 后 Raw/Parsed Streaming 0 行、Policy errors 0 observed；尚不能证明数值恢复，也不能因此说部署失败。请先运行 [本地 MCP Follow-up 只读 Prompt](kql/poc-parser-fix-20261117/controlled-rollout/FOLLOWUP_AI_PROMPT.md)；严禁继续修改其余 Functions 或自动回滚。详细报告：[2026-11-20 Evidence](evidence/2026-11-20/README.md)。
