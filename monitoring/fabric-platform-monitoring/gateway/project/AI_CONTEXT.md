# AI / 技术交接说明

> 更新时间：2026-11-23（Asia/Singapore）。**先读 [项目总览](README.md) → [当前状态](STATUS.md) → [问题清单](tracking/ISSUES.md)**，然后按任务读取相关证据。

## 固定背景

- 当前 POC 是 **单 Gateway Cluster、单 Node**；项目路径 `monitoring/fabric-platform-monitoring/gateway/project/`；工作分支 `docs/gateway-monitoring-poc-20261112`。
- `project/` 是**我们的工作区**；Vendor PowerShell / Scheduler / Report 以及 `src/Gateway/` 仍留在上游位置，避免影响上游升级差异比对。已确认的 Heartbeat 修改详见 [UPSTREAM_CHANGES](docs/UPSTREAM_CHANGES.md)。
- Uploader 原始实现为 `Move → Temp → Eventstream → History`，8 种 Report CSV 已观察到；`HTTP 201` 不等于端到端 Exactly-once。
- KQL 已获得 15/15 实际部署 Function Body 和 142 行 Schema；部署 Body 与 GitHub 源码一致。3 个 Parser 的只读修复候选成功恢复非空，**Schema 39/39 已通过**，13 个数值转换项在当前 Raw 样本中零丢失、零超阈值差异；Aggregation / Streaming 235/235 条 DataSource 规范化值与 Legacy 一致；Mashup 103 条 Raw 无 Kinds/Paths 独立 Key。**2026-11-20 用户已手工执行 Streaming 与 Aggregation Function DDL，二者新数据 POC 功能回归均已通过；Mashup 未更改**。
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

**当前交接点（以 STATUS 为准）：** Streaming 与 Aggregation 已在 POC 部署并通过新数据功能回归。Mashup 三个非必需字段已按 ADR-009 延期至 GW-009；2026-11-23 本地 MCP 的五字段最小 Precheck（Memory 4×33/33、ProcessorTime 36/36、Schema 17/17）与**非目标 12 列兼容性 Gate（Legacy 36/Candidate 36、0 差异、PASS）**均通过。**仅 Mashup 的 POC 部署前验证 PASS，Live Function 未更改、未做部署后 Postcheck。** GitHub 已准备 [P1-only Mashup Candidate](kql/poc-parser-fix-20261117/candidates/4-parse_MashupEvaluationReport_P1_minimal_POC.kql)、已核对的 [旧版 Rollback](kql/poc-parser-fix-20261117/rollback/3-rollback-parse_MashupEvaluationReport.kql)、[完整 Runbook 及只读 Postcheck 10/11/12](kql/poc-parser-fix-20261117/controlled-rollout/MASHUP_RUNBOOK.md)；原历史 Blocked Candidate 不要使用。**下一步：由用户核对 POC 环境/现场 Body/Policy 无漂移后，在 Fabric UI 人工执行单个 Mashup DDL，记录真实 Patch UTC；本地 MCP 只读验证新数据与 Policy failures**。未经用户执行，不得称 Mashup 已部署/P1 Completed，不改 PROD。之后重点是 P2 Collector Reliability。
