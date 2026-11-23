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

**当前交接点（以 STATUS 为准）：** Streaming 与 Aggregation POC 功能回归 Passed。Mashup 三个非必需指标 GW-009 Deferred。2026-11-23 Mashup 五项目标字段 Precheck + 12 列兼容性 Gate Passed；随后**用户已手工执行 POC Candidate**，并给出 `RecordedAfterPatchUTC=2026-11-23T02:39:15.4152749Z`，上传的 Function 导出显示 Live Body 已包含 `mv-expand record = log` 和显式五字段映射，见 [部署摘要](evidence/2026-11-23/MASHUP_MANUAL_DEPLOYMENT_20261123.md)。**Postcheck 仍未完成**，不能称 Mashup 功能 Passed/关闭 GW-001，也不改正式 `DatabaseSchema.kql`。下一步立即让本地 Eventhouse MCP 用真实 Patch UTC **只读**运行 [Runbook](kql/poc-parser-fix-20261117/controlled-rollout/MASHUP_RUNBOOK.md) 的 10/11/12，核查新 Raw / Parsed、五指标多重签名、Policy failures、Live Body/Policy；无新数据为 Inconclusive，失败则评估手工 Rollback，绝不自动执行 DDL。之后进入 P2 Collector Reliability；PROD 尚未批准。
