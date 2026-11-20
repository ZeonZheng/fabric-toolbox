> **证据性质：** GitHub 保存的用户本地 Eventhouse Remote MCP Follow-up 报告（检查后未发现需脱敏的环境 GUID / Secrets）。ChatGPT 评审结论见 [独立评审](AGGREGATION_FOLLOWUP_REVIEW.md)，未直接访问用户 Eventhouse。

# Gateway Aggregation Parser 后续只读回归报告（Followup）

> 依据 `AGGREGATION_FOLLOWUP_AI_PROMPT.md`（分支 `docs/gateway-monitoring-poc-20261112`）执行，全程只读；未重新部署 Function、未回滚、未修改 Mashup / Uploader / Update Policy，无任何 PROD 操作。

## A. 环境、时间与 GitHub 溯源

| 项目 | 值 |
| --- | --- |
| 目标数据库 | **Gateway Monitoring**（POC，`eventhouse-remote` item-scoped 端点，仅作用于用户确认的 POC 数据库） |
| PatchUtc（不可更改） | `2026-11-20T07:24:24.1258875Z` |
| 实际执行时间（UTC+8） | 2026-11-20 15:47 – 15:51（对应服务器 UTC 07:47–07:51） |
| 仓库 / 分支 | `ZeonZheng/fabric-toolbox` / `docs/gateway-monitoring-poc-20261112`（非 main） |
| 分支 HEAD Commit | `c500c587b287944b25ae0584caf06e93f632d544`（2026-11-20T07:38:52Z，经 `gh api` 核实） |
| 本轮执行查询 | 09 Activity Gate → Gate 通过 → 06 / 07 / 08（全部成功，无语法或运行错误） |
| 上轮结论（15:25） | Inconclusive（Patch 后 Raw=0 / Parsed=0，聚合节奏未到） |

## B. Function 与 Update Policy

| 项目 | 值 |
| --- | --- |
| `BodyMatchesCandidate` | **true**（规范化 SHA256 `626cb51072d65bf416480da28ddc8646bf96dc9756554db3436c0f845e2c7f21`，现场与 GitHub Candidate 相同；本轮经 `hash_sha256(Body)` 二次复核一致） |
| Update Policy Enabled | **true** |
| Policy Source | **`GatewayReports-Raw`** |
| Policy Query | `parse_QueryExecutionAggregationReport()` |
| IsTransactional | **true** |

## C. 全部 logType 活跃度（查询 09 Activity Gate）

| logType | Envelope 总数 | Patch 后 Envelope | Patch 后内嵌记录 | 最新 Envelope (UTC) |
| --- | --- | --- | --- | --- |
| ConcurrentOperationAggregationReport | 224 | 5 | 9 | 07:45:45 |
| MashupEvaluationReport | 56 | 3 | 10 | 07:39:45 |
| OpenConnectionReport | 44 | 3 | 10 | 07:34:45 |
| **QueryExecutionAggregationReport** | **39** | **2** | **7** | **07:40:46** |
| QueryExecutionReport | 57 | 3 | 14 | 07:39:46 |
| QueryStartReport | 47 | 3 | 14 | 07:34:46 |
| QueryStreamingReport | 46 | 5 | 15 | 07:41:45 |
| SystemCounterAggregationReport | 442 | 5 | 45 | 07:45:46 |

**目标 Parsed Table（`QueryExecutionAggregationReport`）：** 全表 3162 行；**Patch 后入库 7 行**；最新摄入 `07:40:46Z`。

**Gate 判定：** Aggregation Raw（2 Envelope / 7 内嵌记录）与 Parsed（7 行）**双侧均有新数据** → 按 Prompt 第 3 步继续 06/07/08。所有 8 类 logType 在 Patch 后持续产出，Uploader 链路健康。

## D. Aggregation Raw/Parsed 后续计数与六字段 NonNull（查询 06）

| Layer | Rows | AvgQueryExec | MaxQueryExec | MinQueryExec | AvgDataProc | MaxDataProc | MinDataProc | LatestUTC | 时间字段 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Raw | 7 | 7 (100%) | 7 (100%) | 7 (100%) | 1 (14%) | 1 (14%) | 1 (14%) | 07:40:46 | `logDate`（envelope） |
| Parsed | 7 | 7 (100%) | 7 (100%) | 7 (100%) | 1 (14%) | 1 (14%) | 1 (14%) | 07:40:46 | `ingestion_time()` |

**解读：** 两层行数一致（7=7）；六字段非空计数两层**完全一致**。Query Execution 三字段全部有值；Data Processing 三字段仅 1/7 有值——与 Prompt 说明一致（**Data Processing Duration 允许真实空值**），非解析缺陷。

## E. 六字段 Key 对照及 RawPresent−Matches−Mismatches 差额（查询 07）

查询 07 按**分支 HEAD 实际文件内容**逐字执行（仅替换 PatchUtc），并重复执行两次，结果完全一致：

| 指标 | 值 |
| --- | --- |
| UniqueRawKeys | 2 |
| MatchedParsed | 2（2/2 Key 全部命中 Parsed） |

| 字段 | RawPresent | Matches | Mismatches | 差额（RawPresent − Matches − Mismatches） |
| --- | --- | --- | --- | --- |
| AverageQueryExecutionDuration_ms_ | 2 | 2 | 0 | **0** ✓ |
| MaxQueryExecutionDuration_ms_ | 2 | 2 | 0 | **0** ✓ |
| MinQueryExecutionDuration_ms_ | 2 | 2 | 0 | **0** ✓ |
| AverageDataProcessingDuration_ms_ | 1 | 1 | 0 | **0** ✓ |
| MaxDataProcessingDuration_ms_ | 1 | 1 | 0 | **0** ✓ |
| MinDataProcessingDuration_ms_ | 1 | 1 | 0 | **0** ✓ |

**六项 `RawPresent = Matches + Mismatches` 全部成立，无潜在遗漏。**

**说明：**
- 7 条内嵌记录聚合为 2 个唯一组合 Key（同一聚合窗口跨多个 Envelope 重复上报，`arg_max` 取最新），属正常形态。
- Data Processing 字段：2 个 Key 中 1 个 Raw 有值且 Parsed 精确匹配（浮点容差 ≤0.0001 / 整数相等）；另 1 个 Raw 真实 null → Parsed 亦 null（`toreal/tolong(null)` 行为，符合函数定义与"Raw null 允许 Parsed null"的验收规则），不计入 Matches 亦不计入 Mismatches，故 RawPresent=1。
- KeyJoin 仅为样本一致性对照，**不构成 Exactly-once 证明**。

## F. Patch 后 Ingestion Failures（查询 08）

| Table | FailureKind | FailureCount | FirstUtc | LastUtc |
| --- | --- | --- | --- | --- |
| （无记录） | — | 0 | — | — |

Patch 后无任何 `OriginatesFromUpdatePolicy == true` 失败。

## G. Verdict：`Passed`

**判定依据（对照 Prompt 的 Passed 条件，全部满足）：**

| 条件 | 证据 |
| --- | --- |
| 部署后新 Raw / Parsed | Raw 2 Envelope / 7 内嵌记录；Parsed 7 行（ingestion_time ≥ PatchUtc） |
| Raw 有值字段对应 Parsed 有值且数值相符 | Query Execution 3 字段 2/2 匹配；Data Processing 3 字段 1/1 匹配；**0 Mismatch** |
| 无相关 failures | 查询 08 为空 |
| Body 仍匹配 Candidate | SHA256 `626cb510…c7f21` 相同；Policy Enabled / Source Raw / Transactional 正常 |
| 六字段恒等式 | RawPresent = Matches + Mismatches，六项差额均为 0 |

**已验证事项：** 新数据端到端流转（Raw→Update Policy→Parsed）、六字段非空行为两层一致、Raw 有值字段的 Parsed 值逐一相等、Data Processing 真实空值场景按预期保持 null、无摄入失败、函数与 Policy 状态正确。

**未验证事项 / 局限：**
1. 样本量小（2 唯一 Key / 7 记录，时间窗约 17 分钟），仅覆盖极少数聚合窗口与 QueryType/Success 组合；建议后续持续观察累积样本。
2. KeyJoin 不是 Exactly-once 证明；`0 failures` 也不证明端到端无丢失/无重复。
3. Raw `logDate`（envelope）与 Parsed `ingestion_time()`（摄入）语义不同，本报告未做逐条时间对账（Prompt 已知限制）。
4. Data Processing 无值记录（1/2 Key）本轮无匹配证据（RawPresent=0，按规则不标记 Verified），属设计内行为而非缺陷。

## 附：关于此前"查询 07 字段引用错误"报告的更正

经本轮重新读取分支 HEAD（`c500c587`）的 `07-aggregation-postdeploy-keycheck.kql` **实际内容**逐组核对：全部 6 组 `Matches/Mismatches` 表达式均正确引用对应字段（含 `R_MaxDataProcessingDuration_ms_ == P_MaxDataProcessingDuration_ms_`、`R_MinDataProcessingDuration_ms_ == P_MinDataProcessingDuration_ms_`）。**此前 Postcheck 报告（15:25）第 8 节所称"Data Processing 两组比较误引用 QueryExecution 字段"不成立**，系当时抓取到旧版本文件所致，特此更正。本次执行即为分支 HEAD 原文逐字运行，未对 GitHub KQL 做任何改写。

---

### 附：真实脱敏查询输出汇总（供独立复核）

| # | 查询 | 关键输出 |
| --- | --- | --- |
| 1 | 09 Activity Gate | 9 行（8 类 Raw logType + 1 行 Parsed）；Aggregation Raw AfterPatch=2/7，Parsed AfterPatch=7 |
| 2 | 06 summary | Raw：7/7/7/7/1/1/1，Latest 07:40:46；Parsed：7/7/7/7/1/1/1，Latest 07:40:46 |
| 3 | 07 keycheck（两次执行一致） | [2, 2, 2,2,0, 2,2,0, 2,2,0, 1,1,0, 1,1,0, 1,1,0]（UniqueRawKeys=2，MatchedParsed=2，六字段 RawPresent/Matches/Mismatches 依序） |
| 4 | 08 failures | 0 行 |
| 5 | `.show functions` hash | `626cb51072d65bf416480da28ddc8646bf96dc9756554db3436c0f845e2c7f21` |
| 6 | Update Policy | IsEnabled=true；Source=GatewayReports-Raw；Query=parse_QueryExecutionAggregationReport()；IsTransactional=true |