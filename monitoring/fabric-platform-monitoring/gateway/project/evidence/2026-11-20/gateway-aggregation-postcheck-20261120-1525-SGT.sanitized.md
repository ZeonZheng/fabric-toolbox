> **证据说明：** 用户上传的本地 Eventhouse Remote MCP 只读 Postcheck 报告的脱敏仓库副本，ChatGPT 未直接通过用户的 MCP 复跑。报告第 8 节所述 KeyCheck 错误与 GitHub 当前文件不符，详见 [独立复核](AGGREGATION_POSTCHECK_REVIEW.md)。

# Gateway Aggregation Parser 部署后只读回归报告（Postcheck）

> 依据 `AGGREGATION_POSTCHECK_AI_PROMPT.md`（分支 `docs/gateway-monitoring-poc-20261112`）执行，全程只读，未执行任何 DDL、回滚、修改 Uploader / Update Policy 或 PROD 操作。

## 1. POC 身份与执行环境

| 项目 | 值 |
| --- | --- |
| 目标数据库 | **Gateway Monitoring**（POC，`eventhouse-remote` 经 `.show database` 核实） |
| MCP 工具 | `eventhouse-remote`（item-scoped，绑定默认 Eventhouse） |
| 仓库 / 分支 | `ZeonZheng/fabric-toolbox` / `docs/gateway-monitoring-poc-20261112`（未切换 main） |
| `RecordedAfterPatchUTC` | `2026-11-20 07:24:24.1258875`（用户实际提供） |
| 实际执行时间（UTC+8） | 2026-11-20 15:25 – 15:29 |
| 用户人工确认 DDL | 是（用户声明已完成 `parse_QueryExecutionAggregationReport` 手动部署后才发起本任务） |
| 查询 06（summary） | ✅ 成功执行 |
| 查询 07（keycheck） | ✅ 成功执行（另见 §8 的文件缺陷说明） |
| 查询 08（failures） | ✅ 成功执行 |

## 2. Function 与 Update Policy

| 项目 | 值 |
| --- | --- |
| `parse_QueryExecutionAggregationReport` 存在 | 是 |
| `BodyMatchesCandidate` | **true** |
| 比较方式 | 现场经 `base64_encode_tostring(Body)` 取回精确字节，与 GitHub Candidate（`candidates/1-parse_QueryExecutionAggregationReport.kql` 剥离 `.create-or-alter` 头部）做逐字节精确比对 + 规范化（CRLF→LF、去行尾空白、去空行）SHA256 比对，非仅新旧 Hash 不同 |
| 精确匹配 | true（逐字节一致，长度均 1099 字符） |
| 规范化 SHA256（Candidate / 现场） | `626cb51072d65bf416480da28ddc8646bf96dc9756554db3436c0f845e2c7f21`（两者相同） |
| Update Policy | `IsEnabled: true`，Source `GatewayReports-Raw`，Query `parse_QueryExecutionAggregationReport()`，`IsTransactional: true`，完好未变 |

## 3. Raw / Parsed 两层新行数与六字段非空率（查询 06）

| Layer | Rows | AvgQED | MaxQED | MinQED | AvgDPD | MaxDPD | MinDPD | LatestUTC | 时间字段 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Raw | 0 | 0 | 0 | 0 | 0 | 0 | 0 | null | `logDate`（envelope） |
| Parsed | 0 | 0 | 0 | 0 | 0 | 0 | 0 | null | `ingestion_time()` |

**PatchUtc 后无任何新行**，六字段非空率无从计算。

**历史数据与事件节奏背景（补充只读检查）：**

| 范围 | Raw (QueryExecutionAggregationReport) |
| --- | --- |
| 全表 Envelope | 37 |
| 近 6 小时 | 10 |
| 近 1 小时 | 3 |
| 最新 Envelope | 2026-11-20T06:45:45Z（PatchUtc 前约 39 分钟） |

Parsed `QueryExecutionAggregationReport` 全表 3155 行，最新摄入 `06:45:45Z`（PatchUtc 前）。该类型为**聚合周期上报**，节奏本身稀疏（约每分钟至数分钟一批，且存在间隔），PatchUtc 后至今（约 5 分钟）无新 Envelope。

## 4. 复合 Key 样本对照（查询 07）

| 指标 | 值 |
| --- | --- |
| UniqueRawKeys | 0 |
| MatchedParsed | 0 |
| 六字段 RawPresent / Matches / Mismatches | 全部 0 |

无样本，逐字段 Matches/Mismatches 无统计意义。

> KeyJoin 限制（仍适用）：组合 Key 可能重复（取最新记录）；Raw 按 `logDate`（envelope）、Parsed 按 `ingestion_time()` 筛选，两者语义不同；本结果非 Exactly-once 证明。

## 5. Post-Patch Ingestion Failures（查询 08）

| Table | FailureKind | FailureCount | FirstUtc | LastUtc |
| --- | --- | --- | --- | --- |
| （无记录） | — | 0 | — | — |

PatchUtc 后无任何 `OriginatesFromUpdatePolicy == true` 失败。

## 6. Verdict：`Inconclusive`

**判定依据：** 按 Prompt 契约，PatchUtc 后**无新事件**（Raw 0 行、Parsed 0 行）→ `Inconclusive`。函数部署本身已确认成功（Body 与 Candidate 完全一致、Policy 启用、无摄入失败），但六字段非空率与值匹配两项核心验证因无样本而**无法执行**。

**已确认项（有证据）：**
- 现场函数体与 GitHub Candidate 逐字节一致（`BodyMatchesCandidate: true`）。
- Update Policy 完好启用（Enabled / Source Raw / Transactional 三项均符合）。
- Patch 后无相关摄入失败。

**未验证项：**
- Parsed 新行六字段非空率（无新行）。
- 六字段 Raw↔Parsed 值匹配 / 错配（无样本）。
- Data Processing Duration 真实空值场景下"仅对 Raw 有值记录要求匹配"的行为（无样本）。

**非问题排除：** 上游 Uploader 与其它 logType 在 Patch 后持续活跃（参考 Followup 报告 §3：8 类 logType 均有新 Envelope），本类型 0 行属聚合节奏稀疏，非链路中断；PatchUtc 仅间隔约 5 分钟，聚合类型大概率在下一周期产生新数据。

## 7. 建议下一人工决策

1. **等待下一个聚合周期（建议 30–60 分钟）后重跑 06/07/08**（同一 PatchUtc `2026-11-20T07:24:24.1258875`），届时应有新 Envelope 完成完整验证。
2. 重跑前建议先修正查询 07 的文件缺陷（见 §8），否则 Data Processing 两字段的 Matches/Mismatches 结果会误报为 Query Execution 字段的比较值。
3. 若重跑后六字段非空率与匹配全部通过且无失败 → 可判 `Passed`；若出现非空率异常、值错配或新失败 → 判 `Blocked`。

## 8. 附注：查询 07 文件缺陷（不影响本次结论）

分支上 `07-aggregation-postdeploy-keycheck.kql` 的 **Data Processing** 两组比较存在复制粘贴缺陷：`MaxDataProcessingDuration_ms__Matches/Mismatches` 与 `MinDataProcessingDuration_ms__Matches/Mismatches` 的比较表达式实际引用了 **QueryExecution** 字段（`R_MaxQueryExecutionDuration_ms_ == P_MaxQueryExecutionDuration_ms_` 等），而非对应 DataProcessing 字段。本次因 0 行无实际影响；本报告另跑了一版**修正版** keycheck（已验证可执行，结果同为全 0），下次重跑有数据时应采用修正版或由用户先在 GitHub 修复该文件。

---

### 附：脱敏查询结果计数汇总（供独立复核）

| # | 查询 | 关键结果 |
| --- | --- | --- |
| 1 | `.show functions` Body base64 | 1 行；规范化 SHA256 `626cb510…c7f21`，与 Candidate 一致 |
| 2 | 查询 06 | Raw 0 / Parsed 0（六字段全 0，LatestUTC null） |
| 3 | 查询 07（原版 + 修正版各一次） | 均 20 列全 0 |
| 4 | 查询 08 | 0 条 Update Policy 失败 |
| 5 | `.show table QueryExecutionAggregationReport policy update` | IsEnabled=true；Source=GatewayReports-Raw；Query=parse_QueryExecutionAggregationReport()；IsTransactional=true |
| 6 | Raw 节奏（补充） | 全表 37；近 6h=10；近 1h=3；最新 06:45:45Z（Patch 前 39 分钟） |
| 7 | Parsed 全表（补充） | 3155 行；最新摄入 06:45:45Z |