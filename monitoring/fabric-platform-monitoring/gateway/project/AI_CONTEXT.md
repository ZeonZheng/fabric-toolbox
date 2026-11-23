# AI / 技术交接说明

> 更新时间：2026-11-23（Asia/Singapore）。**先读 [项目总览](README.md) → [当前状态](STATUS.md) → [问题清单](tracking/ISSUES.md)**，然后按任务读取相关证据。

## 固定背景

- 当前 POC 是 **单 Gateway Cluster、单 Node**；项目路径 `monitoring/fabric-platform-monitoring/gateway/project/`；工作分支 `docs/gateway-monitoring-poc-20261112`。
- `project/` 是**我们的工作区**；Vendor PowerShell / Scheduler / Report 以及 `src/Gateway/` 仍留在上游位置，避免影响上游升级差异比对。已确认的 Heartbeat 修改详见 [UPSTREAM_CHANGES](docs/UPSTREAM_CHANGES.md)。
- Uploader 原始实现为 `Move → Temp → Eventstream → History`，8 种 Report CSV 已观察到；`HTTP 201` 不等于端到端 Exactly-once。
- KQL 已获得 15/15 实际部署 Function Body 和 142 行 Schema；部署 Body 与 GitHub 源码一致。3 个 Parser 的只读修复候选成功恢复非空，**Schema 39/39 已通过**，13 个数值转换项在当前 Raw 样本中零丢失、零超阈值差异；Aggregation / Streaming 235/235 条 DataSource 规范化值与 Legacy 一致；Mashup 103 条 Raw 无 Kinds/Paths 独立 Key。**Streaming、Aggregation 与 Mashup 三项已在 POC 部署并通过本轮必需字段的新数据功能回归（2026-11-23），但文件级 Collector Reliability 尚未验证**。
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

**当前交接点（以 STATUS 为准）：** `P0 Functional POC` 已完成；`P1 Data Correctness` **本轮必需 Parser 功能验收已完成（2026-11-23）**。Streaming 新数据 32/32、Aggregation Raw/Parsed 各 7 行且复合 Key 2/2、Mashup 2026-11-23T03:06–03:08Z 复测 Gate 13 新 Raw 2 Envelope/4 Records、Raw/Parsed 各 4 行、五个 Memory/Time 字段各 4/4、KeyCheck 0 unmatched/0 different、Policy failures 0 observed，**POC 功能 PASS**。见 [独立评审](evidence/2026-11-23/MASHUP_POSTCHECK_PASS_1105_REVIEW.md)。Mashup POC Live Function 已由用户手工部署；**仅其通过测试的 Body** 已在 GitHub fork 工作分支同步正式 `src/Gateway/Gateway Monitoring.KQLDatabase/DatabaseSchema.kql`（commit `f4304485`），**本次没有额外 Eventhouse DDL**。GW-001 在范围内 Closed；GW-009 三个不必需字段 Deferred、GW-005 其它覆盖缺口开放、历史 null 未补回。**现在的唯一重点是 P2 Collector Reliability / GW-002 的只读 Baseline**：审计 Uploader 文件 Move/Temp/Batch/History、Task Scheduler、可用 VM 元数据和 Eventhouse Raw/Parsed，对照哪些层可证明文件/行/批次完整性；不要假设本地 AI 能访问 VM，不能因 Eventhouse 4/4 宣称源端 Exactly-once。首轮不实施写操作、故障注入、Scheduler/Config/源码修改，不动 PROD。之后再准备受控可靠性测试。最新状态以 [STATUS](STATUS.md) 为准。
