# 2026-11-20｜Aggregation Parser 首次 Postcheck 独立复核

> **报告执行：** 2026-11-20 15:25–15:29，Asia/Singapore。**真实 PatchUTC：** `2026-11-20T07:24:24.1258875Z`。  
> **证据来源：** [用户本地 MCP 报告](gateway-aggregation-postcheck-20261120-1525-SGT.sanitized.md)，非 ChatGPT 直接访问 Eventhouse。  
> **最终判定：Inconclusive（缺少 Patch 后新样本）**；Function Body 已与 GitHub Candidate 逐字节匹配；不能判定六个 Duration 修复成功，也没有证据要求立即回滚。

## 实测

| 检查 | 本地 AI 实测 | 独立解读 |
| --- | --- | --- |
| `parse_QueryExecutionAggregationReport` Body | `BodyMatchesCandidate=true`，相同规范化 SHA256 `626cb51072d65bf416480da28ddc8646bf96dc9756554db3436c0f845e2c7f21` | 已证实 Candidate 在 POC 现场生效 |
| Policy | Enabled=true、Source=`GatewayReports-Raw`、IsTransactional=true | 未见配置漂移 |
| Patch 后 Raw / Parsed | **0 / 0** | 六指标都无法验收；`0 mismatch` 在此处没有统计意义 |
| 最近 Raw Aggregation Envelope | `2026-11-20T06:45:45Z`，早于 PatchUTC 约 39 分钟 | Aggregation 事件可能稀疏，不得声称摄入停止或修复成功 |
| Query 07 样本 Key | 0 Raw Keys，0 Parsed Match | 没有可比数据 |
| Post-Patch Policy failures | **0 observed** | 暂无已观察到的失败；不证明链路无丢失 |

## 对本地 AI 所称“Query 07 脚本 Bug”的独立检查

用户报告第 8 节声称 `MaxDataProcessingDuration_ms_` 和 `MinDataProcessingDuration_ms_` 的 Matches / Mismatches 错误引用 QueryExecution 列。**GitHub 当前工作分支的真实 `07-aggregation-postdeploy-keycheck.kql` 并不存在该错误。**

- 逐条检查 **6 个指标 × Matches/Mismatches = 12 个表达式**，每个都引用同名 `R_<字段>` / `P_<字段>`；未检测到跨指标误引用。
- MaxDataProcessing 比较的是真正的 `R_MaxDataProcessingDuration_ms_` 与 `P_MaxDataProcessingDuration_ms_`，MinDataProcessing 也正确引用自身列。
- 当前 GitHub 文件 Blob SHA：`f5eeb5b70acb87bbf0b6e00f8534dc5cf37bd23a`。本地 AI 可能查看了旧缓存或误读脚本；**不据此修改正常的查询 07**。若本地有同名错误版本，应以指定分支的远程文件为准。
- 现有 Query 07 的另一个局限是：`Mismatches` 只统计两个值均非空但不等的行，**不会将 Raw 有值而 Parsed 为 null 算入 Mismatches**。需要复核 `RawPresent == Matches + Mismatches`，若不等则属于可能未配对或值丢失，不能判 Pass。

## 其它需要谨慎的推断

原报告引用 14:48 Streaming Follow-up 的“八类 Raw 活跃”来说明 15:24 Aggregation Patch 后上游正常；**时间窗口不一致**。过去有活动不等于本次 Patch 后继续活动，已准备按当前 Patch UTC 检查所有 Raw logType 的只读 [Activity Gate](../../kql/poc-parser-fix-20261117/controlled-rollout/09-aggregation-activity-gate.kql)。

## 当前动作与边界

- [MCP 后续检查 Prompt](../../kql/poc-parser-fix-20261117/controlled-rollout/AGGREGATION_FOLLOWUP_AI_PROMPT.md)：先运行 Activity Gate，确认有新 Aggregation Raw / Parsed 后，沿用相同 Patch UTC 再执行 06/07/08；逐项对照六字段，Raw DataProcessing 字段原本空值允许 Parsed 为空。
- 若没有新记录，仍 `Inconclusive`；若出现新增 Policy failures、Function Body 漂移或 Raw 非空却 Parsed null 等明确故障，输出 `Blocked` 并人工评估回滚。
- **不得重复执行 Candidate，不得自动回滚；Mashup 和 PROD 不改动。**
- 仍需验证长时间稳定性、历史回填和文件级无丢失/无重复，这些不属于本次 0 行 Postcheck。

**当前 fork 的正式 `src/Gateway/Gateway Monitoring.KQLDatabase/DatabaseSchema.kql` 尚未同步 Aggregation Candidate**：这是刻意控制，待 POC 新记录验证通过后再同步。
