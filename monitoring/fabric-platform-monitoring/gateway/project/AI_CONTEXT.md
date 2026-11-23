# AI / 技术交接说明

> 更新时间：2026-11-23（Asia/Singapore）。**先读 [项目总览](README.md) → [当前状态](STATUS.md) → [问题清单](tracking/ISSUES.md)**，然后按任务读取相关证据。

## 固定背景

- 当前 POC 是 **单 Gateway Cluster、单 Node**；项目路径 `monitoring/fabric-platform-monitoring/gateway/project/`；工作分支 `docs/gateway-monitoring-poc-20261112`。
- `project/` 是**我们的工作区**；Vendor PowerShell / Scheduler / Report 以及 `src/Gateway/` 仍留在上游位置，避免影响上游升级差异比对。已确认的 Heartbeat 修改详见 [UPSTREAM_CHANGES](docs/UPSTREAM_CHANGES.md)。
- Uploader 原始实现为 `Move → Temp → Eventstream → History`，8 种 Report CSV 已观察到；`HTTP 201` 不等于端到端 Exactly-once。
- KQL 已获得 15/15 实际部署 Function Body 和 142 行 Schema；部署 Body 与 GitHub 源码一致。3 个 Parser 的只读修复候选成功恢复非空，**Schema 39/39 已通过**，13 个数值转换项在当前 Raw 样本中零丢失、零超阈值差异；Aggregation / Streaming 235/235 条 DataSource 规范化值与 Legacy 一致；Mashup 103 条 Raw 无 Kinds/Paths 独立 Key。**Streaming 与 Aggregation 已在 POC 部署并通过新数据回归；Mashup 已于 2026-11-23 在 POC 手工部署，但首轮 Postcheck 无新 Raw、结果 INCONCLUSIVE**。
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

**当前交接点（以 STATUS 为准）：** Streaming 和 Aggregation POC 功能 Passed；Mashup P1 最小 Candidate 经 Precheck+12列兼容性 Gate PASS 后于 2026-11-23 **用户手工部署**。真实 `RecordedAfterPatchUTC=2026-11-23T02:39:15.4152749Z`。本地 Eventhouse MCP 首轮 02:47–02:50 UTC Postcheck：Live Body 与 Candidate SHA256 `7f333c39…2813` 匹配，Policy Enabled/Transactional；新 Raw=0、新 Parsed=0、Keycheck `INCONCLUSIVE_NO_NEW_RAW`、Policy failures 0 observed（零样本无成功摄入证明）。见 [独立评审](evidence/2026-11-23/MASHUP_POSTCHECK_1045_REVIEW.md)。**不要重跑 DDL、不要因零样本回滚、不要判 Mashup Passed 或修改正式 KQL Schema**。下一步用只读 [13 Activity Gate](kql/poc-parser-fix-20261117/controlled-rollout/13-mashup-postdeploy-activity-gate.kql) 检查是否有新 Mashup Raw；有新记录才跑 [10/11/12 Postcheck](kql/poc-parser-fix-20261117/controlled-rollout/MASHUP_RUNBOOK.md)，PatchUtc 保持不变。可并行筹备 P2 GW-002。GW-009 三字段延期，PROD 未批准。
