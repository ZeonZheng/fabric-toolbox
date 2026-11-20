> **证据性质：** 用户上传的本地 Eventhouse MCP 只读 Follow-up 报告。GitHub 备份已检查，不含数据库 GUID、Secret、DataSource 明文或完整 Function Body。本文不是 ChatGPT 直接访问 Eventhouse 得到的原始查询结果。详见 [独立评审](STREAMING_FOLLOWUP_REVIEW.md)。

# Gateway Streaming Parser 后续只读验证报告（Followup）

> 依据 `FOLLOWUP_AI_PROMPT.md`（分支 `docs/gateway-monitoring-poc-20261112`）执行，全程只读，未修改任何 Function、Update Policy、Uploader 或 PROD 对象，无回滚操作。

## 1. 目标 POC 与执行环境

| 项目 | 值 |
| --- | --- |
| 目标数据库 | **Gateway Monitoring**（POC，经 `.show database` 核实；按指令不回传 GUID） |
| MCP 端点 | `eventhouse-remote`（item-scoped，绑定默认 Eventhouse） |
| 仓库 / 分支 | `ZeonZheng/fabric-toolbox` / `docs/gateway-monitoring-poc-20261112`（未切换 main） |
| Patch UTC | `2026-11-20T06:19:18.0313089Z`（与首轮 Postcheck 同一基准） |
| 实际执行时间（UTC+8） | 2026-11-20 14:22 – 14:48 |
| 上轮结论（14:22） | Inconclusive（Patch 后 Raw/Parsed 均 0 行） |

## 2. Candidate Body 对照

| 项目 | 值 |
| --- | --- |
| `BodyMatchesCandidate` | **true** |
| 比较方式 | ① 现场经 `base64_encode_tostring(Body)` 取回精确字节，本地 UTF-8 解码；② GitHub Candidate 文件剥离 `.create-or-alter` 头部后提取函数体；③ 先做逐字节精确比对，再做规范化（CRLF→LF、去行尾空白、去空行）后 SHA256 比对 |
| 精确匹配 | true（逐字节一致，长度均 734 字符） |
| 规范化 SHA256（Candidate） | `587bba1a7ad9dac82c873b8a2ed4ef5ef6cc15a5b8cd3ef3e62df13b810703ac` |
| 规范化 SHA256（现场） | `587bba1a7ad9dac82c873b8a2ed4ef5ef6cc15a5b8cd3ef3e62df13b810703ac` |
| Update Policy | `IsEnabled: true`，Source `GatewayReports-Raw`，Query `parse_QueryStreamingReport()`，`IsTransactional: true`（未变更） |

> 结论：现场函数体与 GitHub Candidate **完全一致**（非仅"与旧版 Hash 不同"）。

## 3. Raw 全部 logType 活跃度（查询 04）

| logType | Envelope 总数 | Patch 后 Envelope | 内嵌记录总数 | 最新 Envelope (UTC) |
| --- | --- | --- | --- | --- |
| ConcurrentOperationAggregationReport | 212 | 6 | 267 | 06:45:45 |
| MashupEvaluationReport | 53 | 6 | 361 | 06:44:45 |
| OpenConnectionReport | 41 | 5 | 362 | 06:44:45 |
| QueryExecutionAggregationReport | 37 | 3 | 101 | 06:45:45 |
| QueryExecutionReport | 54 | 6 | 405 | 06:44:45 |
| QueryStartReport | 44 | 6 | 405 | 06:44:46 |
| **QueryStreamingReport** | **41** | **5** | **161** | **06:44:46** |
| SystemCounterAggregationReport | 430 | 6 | 3662 | 06:45:46 |

**解读：** 全部 8 类 logType 在 Patch 后均有新 Envelope（每类 3–6 条），Uploader 与源文件链路健康；Streaming 类型也已恢复上报，首轮 0 行确系事件稀疏，非摄入中断。

## 4. Parsed 字段恢复（查询 05，全表）

| 指标 | 值 |
| --- | --- |
| AllRows | 2707 |
| StreamingDurationNonNull | 32 |
| SpoolingDurationNonNull | 32 |
| BothNonNull | 32 |
| LastBusinessTimeUTC | 2026-11-20T06:44:21Z |
| NewestValidStreamingBusinessTimeUTC | 2026-11-20T06:44:21Z |
| NewestValidSpoolingBusinessTimeUTC | 2026-11-20T06:44:21Z |

**按 `ingestion_time()` 区分新旧数据（脱敏）：**

| 数据段 | 行数 | Streaming 非空 | Spooling 非空 | 双非空 | 摄入时间范围 (UTC) |
| --- | --- | --- | --- | --- | --- |
| Patch 前摄入（历史） | 2675 | 0 | 0 | 0 | 11-13 06:46 ~ 11-20 06:11 |
| **Patch 后摄入（新）** | **32** | **32 (100%)** | **32 (100%)** | **32 (100%)** | 11-20 06:34 ~ 06:44 |

**解读：** 非空值**首次且仅**出现在 Patch 后摄入的行；历史 2675 行保持 null（旧解析器产物，未被重放，符合预期）。两项 Duration 字段恢复确证。

## 5. 部署后三项回归重跑（同一 PatchUtc）

### 5.1 查询 01 — Raw & Parsed 汇总

| Layer | Rows | WithStreaming | WithSpooling | LatestTimeUTC | 时间字段 |
| --- | --- | --- | --- | --- | --- |
| Raw | 32 | 32 | 32 | 06:44:46 | `logDate`（envelope） |
| Parsed | 32 | 32 | 32 | 06:44:21 | `StreamingStartTimeUTC`（业务） |

### 5.2 查询 02 — KeyCheck（RequestId+QueryTrackingId）

| 指标 | 值 |
| --- | --- |
| UniqueRawKeys | 32 |
| MatchedParsed | 32 |
| RawStreamingPresent / ParsedStreamingMatches | 32 / 32 |
| RawSpoolingPresent / ParsedSpoolingMatches | 32 / 32 |
| StreamingMismatches / SpoolingMismatches | **0 / 0** |

**解读：** 32 个 Raw Key 全部在 Parsed 中匹配，且两项 Duration 值逐一相等，零不匹配。

> KeyJoin 限制（仍适用）：ID 可能复用；Raw 按 envelope 时间、Parsed 按业务时间（放宽 1 天）过滤，语义不同；本结果为样本一致性，非 exactly-once 证明。

### 5.3 查询 03 — Update Policy 摄入失败

| Table | FailureKind | FailureCount | First / Last |
| --- | --- | --- | --- |
| （无记录） | — | 0 | — |

Patch 后无任何 `OriginatesFromUpdatePolicy == true` 失败。

## 6. Verdict：`Passed`

**判定依据（对照 POSTCHECK 契约的 Passed 条件，全部满足）：**

| 条件 | 证据 |
| --- | --- |
| 部署后新摄入 Raw 及对应 Parsed 样本 | Raw 32 行 / Parsed 32 行（Patch 后） |
| 两项 Duration 值恢复且值合理 | Parsed 新行 32/32 双非空（100%），与 Raw 值 32/32 相等、0 mismatch |
| 无相关摄入失败 | 查询 03 为空（0 条） |
| 确认现场函数已改变 | `BodyMatchesCandidate: true`（逐字节 + 规范化 SHA256 双重一致） |

**未验证项 / 局限：**

1. 样本量 32 行、时间窗约 25 分钟（06:19–06:45 UTC），属小样本早期验证；建议持续观察 24 小时。
2. KeyJoin 不能证明 exactly-once（无丢失/无重复）；`0 ingestion failures` 亦不证明端到端无丢失。
3. 历史 2675 行 null 未重放（Update Policy 不回填旧行），如需历史修复须另行评估（本任务禁止操作）。
4. Raw `logDate` 与 Parsed `StreamingStartTimeUTC` 语义不同（envelope vs 业务时间），两者 25 秒级差异属正常管道延迟，未做逐条时间对账。

**建议下一人工决策：**

1. 保持现状观察 24 小时，可择机重跑 01/02/03（同一 PatchUtc）确认持续性。
2. 若确认稳定，可由人工决定是否将 Candidate 推广至其它环境（本任务不执行）。
3. 历史 null 数据是否回填，需单独决策（涉及重放，超出本次只读范围）。

---

### 附：真实查询结果计数汇总（供独立复核）

| # | 查询 | 关键计数 |
| --- | --- | --- |
| 1 | `.show functions` Body base64 | 1 行；规范化 SHA256 `587bba1a…703ac` |
| 2 | 查询 04 活跃度 | 8 类 logType；Patch 后 Envelope 3–6/类；Streaming 5 |
| 3 | 查询 05 字段恢复 | 2707 行；非空 32/32/32；最新有效业务时间 06:44:21Z |
| 4 | ingestion_time 分段 | 旧 2675 行 0 非空；新 32 行 100% 非空 |
| 5 | 查询 01 重跑 | Raw 32/32/32；Parsed 32/32/32 |
| 6 | 查询 02 重跑 | 32 Key 全匹配；Matches 32/32；Mismatch 0/0 |
| 7 | 查询 03 重跑 | 0 条 Update Policy 失败 |
| 8 | Update Policy | IsEnabled=true；Source=GatewayReports-Raw；Query=parse_QueryStreamingReport() |