# 本地 AI / Eventhouse Remote MCP｜Aggregation 修复后有新样本再验收

**目标：** 第一轮 2026-11-20 15:25–15:29 Postcheck `Inconclusive`，原因是变更后 **Raw=0 / Parsed=0**。本轮只读检查新数据是否到达，并在有样本后重跑六字段对照。**不得再次部署 Candidate 或自动回滚。**

## 正确的 GitHub 地址

- 仓库：https://github.com/ZeonZheng/fabric-toolbox
- **必须使用分支**：`docs/gateway-monitoring-poc-20261112`，不是 main。
- 完整目录：https://github.com/ZeonZheng/fabric-toolbox/tree/docs/gateway-monitoring-poc-20261112/monitoring/fabric-platform-monitoring/gateway/project/kql/poc-parser-fix-20261117/controlled-rollout
- [原始 Postcheck 报告及 ChatGPT 独立复核](https://github.com/ZeonZheng/fabric-toolbox/blob/docs/gateway-monitoring-poc-20261112/monitoring/fabric-platform-monitoring/gateway/project/evidence/2026-11-20/AGGREGATION_POSTCHECK_REVIEW.md)

**不可更改的实际 PatchUtc：** `2026-11-20T07:24:24.1258875Z`（用户提供；后续所有检查仍使用同一时间）。

## 请按以下逻辑执行

1. 使用已连接的 `eventhouse-remote` MCP，只在用户已确认的 POC Gateway Monitoring KQL Database 上运行只读查询。
2. 从以上工作分支读取 [09-aggregation-activity-gate.kql](09-aggregation-activity-gate.kql) 并执行。输出 **每类 Raw logType 的 Patch 后 Envelope 数量及内嵌记录数、最新时间**，以及目标 Parsed Table 的 Patch 后入库记录数。
3. **如果 Aggregation Raw 和 Parsed 均已有新记录：** 继续读取并分别执行 [06-aggregation-postdeploy-summary.kql](06-aggregation-postdeploy-summary.kql)、[07-aggregation-postdeploy-keycheck.kql](07-aggregation-postdeploy-keycheck.kql)、[08-aggregation-ingestion-failures.kql](08-aggregation-ingestion-failures.kql)，把 **所有 `2099-01-01T00:00:00Z` 替换为真实 PatchUtc**，保留其它逻辑。
4. **若只有 Raw 或 Parsed 一侧有值：** 同样检查 Policy failures，并报告层间时间字段不一致、业务 Key 复用、摄入延迟等可能原因；除非存在明确错误，不要直接断定丢失。
5. **如果仍没有新 Aggregation 数据：** 不要反复执行 06/07/08；说明其它 Raw 类型是否在 Patch 后继续产出。结论 `Inconclusive`，需有真实聚合周期数据才能回归。
6. 复核现场 `parse_QueryExecutionAggregationReport` Body 与 GitHub [Candidate](https://github.com/ZeonZheng/fabric-toolbox/blob/docs/gateway-monitoring-poc-20261112/monitoring/fabric-platform-monitoring/gateway/project/kql/poc-parser-fix-20261117/candidates/1-parse_QueryExecutionAggregationReport.kql) 一致，检查 Update Policy Enabled、Source Raw、Transactional；只输出布尔和 hash，不粘贴函数体。
7. **严谨检查六个 Duration**：对每个字段分别比较 `RawPresent`、`Matches`、`Mismatches`，确保 **`RawPresent = Matches + Mismatches`**；若不等，计算潜在遗漏并报告。Raw 本身为 null 的 DataProcessing 记录允许 Parsed 继续 null；如果某字段 RawPresent=0，则该字段本轮没有证据，不能标记 Verified。
8. **纠正以前本地 AI 的脚本误报**：远程 GitHub 工作分支查询 07 的全部 6 组 `Matches/Mismatches` 引用都正确。请基于当前**实际文件内容**核对，而不是引用旧报告第 8 节的说法。只有明确指出文件准确路径、具体表达式并与实际内容比对，才能报告新的 Bug；不要未经核实自行改写 GitHub KQL。

## 输出契约

生成中文 Markdown：`gateway-aggregation-followup-YYYYMMDD-HHmm-SGT.md`，提供真实文件路径或全文让用户上传 ChatGPT。

必须包含：**A** 环境、时间和 GitHub Commit/分支；**B** BodyMatchesCandidate、Policy；**C** 全部 logType 活跃度；**D** Aggregation Raw/Parsed 后续计数与六字段 NonNull；**E** 六字段 Key 对照及 RawPresent-Matches-Mismatches 的每项差额；**F** Patch 后 ingestion failures；**G** `Passed / Blocked / Inconclusive`、已验证和未验证事项。附真实脱敏查询输出，不得只写主观概括。

`Passed` 需要看到部署后新 Raw / Parsed、Raw 有值的字段对应 Parsed 有值且数值相符、无相关 failures、Body 仍匹配。无足够样本则 `Inconclusive`；确认失败、值错或异常丢值则 `Blocked`。不要把 Key Join 比对声称为 Exactly-once。

**全部只读。不得触发 DDL、回滚、Task/Uploader 修改、PROD 操作，不得泄露 Source IDs、DataSource 路径、QueryText 或 Token。**
