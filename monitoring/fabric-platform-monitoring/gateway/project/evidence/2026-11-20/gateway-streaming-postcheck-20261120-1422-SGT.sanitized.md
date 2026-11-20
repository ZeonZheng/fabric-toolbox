> **GitHub 脱敏备份：** 2026-11-20 用户上传的本地 MCP Postcheck 报告；数据库 GUID 已移除。本报告为本地 AI 只读查询结果，不代表 ChatGPT 直接访问用户的 Eventhouse。

# Gateway Streaming Parser 部署后只读回归报告（Postcheck）

> 依据 `POSTCHECK_AI_PROMPT.md`（分支 `docs/gateway-monitoring-poc-20261112`）执行，全程只读，无任何写操作。

## 1. Environment & Change

| 项目 | 值 |
| --- | --- |
| 用户已人工确认 POC | 是（用户在 Fabric UI 确认后提供本任务） |
| 实际 `RecordedAfterPatchUTC` | `2026-11-20 06:19:18.0313089`（用户执行 `print RecordedAfterPatchUTC=now()` 提供，非虚构） |
| MCP 工具 | `eventhouse-remote`（item-scoped，绑定默认 Eventhouse） |
| 目标数据库 | `Gateway Monitoring`（DatabaseId `[DATABASE_ID_REDACTED]`，已用 `.show database` 核实） |
| 仓库 / 分支 | `ZeonZheng/fabric-toolbox` / `docs/gateway-monitoring-poc-20261112` |
| 查询 01（postdeploy-summary） | ✅ 成功执行 |
| 查询 02（postdeploy-keycheck） | ✅ 成功执行 |
| 查询 03（ingestion-failures） | ✅ 成功执行 |
| 附加只读检查 | 函数 Hash、Update Policy、Raw 事件节奏、Parsed 最新摄入时间（均成功；一次辅助查询因 `map(parameters,…)` 语法不支持报错，已简化重试成功，不影响结论） |

## 2. Function Check

| 项目 | 值 |
| --- | --- |
| `parse_QueryStreamingReport` 存在 | 是（`.show functions` 返回 1 行） |
| 现场 Body SHA256（`hash_sha256(Body)` 计算） | `587bba1a7ad9dac82c873b8a2ed4ef5ef6cc15a5b8cd3ef3e62df13b810703ac` |
| 2026-11-20 Preflight 原始 SHA256 | `d946208e13a5009a2695a8b52f6ce0c8ed1d2e46d8d148cb069b826bc34f805c` |
| 是否不同 | **不同** → 函数定义已变更 |
| Update Policy（`QueryStreamingReport` 表） | `IsEnabled: true`，Source `GatewayReports-Raw`，Query `parse_QueryStreamingReport()`，`IsTransactional: true`，完好未变 |

> 注意：Hash 不同只能证明函数已变化，**不单独证明修复成功**。本环境 `.show functions` 无 `BodySHA256` 原生列，故用 `hash_sha256(Body)` 现场计算，口径与 Preflight 的记录方式可能不同，仅作"已变更"证据。

## 3. Raw & Parsed（PatchUtc 之后）

| Layer | Rows | WithStreaming | WithSpooling | LatestTimeUTC | 时间字段 |
| --- | --- | --- | --- | --- | --- |
| Raw | 0 | 0 | 0 | null | `logDate`（envelope 到达时间） |
| Parsed | 0 | 0 | 0 | null | `StreamingStartTimeUTC`（业务时间） |

**历史数据 vs Patch 后数据区分：**

| 范围 | Raw 行数 | 说明 |
| --- | --- | --- |
| 全表（logType=QueryStreamingReport） | 36 | 最新事件 `2026-11-20T06:11:46Z`，早于 PatchUtc 约 7.5 分钟 |
| PatchUtc 之后 | 0 | 无新事件 |

| 范围 | Parsed 行数 | 说明 |
| --- | --- | --- |
| 全表 | 2675 | 最新业务时间 `2026-11-20T06:10:47Z`，最新摄入 `2026-11-20T06:11:47Z` |
| PatchUtc 之后 | 0 | 无新摄入 |

**事件节奏（近 6 小时，Raw，UTC）：** 02 时 6 行、03 时 3 行、04–05 时 0 行、06 时 1 行（06:11，Patch 前）。该事件流本身稀疏且不连续，Patch 后 0 行与"低频事件"一致，但**无法排除摄入中断**。

## 4. Sample Key Check（RequestId+QueryTrackingId）

| 指标 | 值 |
| --- | --- |
| UniqueRawKeys | 0 |
| MatchedParsed | 0 |
| RawStreamingPresent / ParsedStreamingMatches | 0 / 0 |
| RawSpoolingPresent / ParsedSpoolingMatches | 0 / 0 |
| StreamingMismatches / SpoolingMismatches | 0 / 0 |

> **KeyJoin 限制**：不同记录可能复用 RequestId/QueryTrackingId；Raw 按 `logDate`（envelope 时间）过滤、Parsed 按 `StreamingStartTimeUTC`（业务时间，且放宽 1 天）过滤，两者语义不同；本结果仅为样本一致性，不是 exactly-once 证明。本次样本为 0，无统计意义。

## 5. Ingestion Failures（PatchUtc 之后，Update Policy 来源）

| Table | FailureKind | FailureCount | FirstFailureUtc | LastFailureUtc |
| --- | --- | --- | --- | --- |
| （无记录） | — | 0 | — | — |

Patch 后无任何 `OriginatesFromUpdatePolicy == true` 的摄入失败。

## 6. Verdict

**结论：`Inconclusive`**

**有证据的结论：**
- 现场函数 `parse_QueryStreamingReport` 已变更（Body Hash 与 Preflight 原始值不同），Update Policy 完好且启用。
- Patch 后无新的 Update Policy 摄入失败（查询 03 为空）。
- Patch 后 Raw 与 Parsed 均无新数据（0 行），因此**无法验证**两项 Duration 值（StreamingDuration / SpoolingDiskReadingDuration）是否恢复且合理。

**未验证项：**
- 部署后新摄入 Raw 及对应 Parsed 样本（不存在）。
- 两项 Duration 值的恢复与合理性（无样本）。
- Keycheck 匹配率（样本为 0）。

**判定依据：** 按 Prompt 契约，"没有新 Raw 事件" → `Inconclusive`。0 行结果与该事件流稀疏节奏一致，但样本不足，不能判 `Passed`；亦无失败或字段异常证据，不能判 `Blocked`。

**建议下一人工决策：**
1. 等待或触发一条新的 `QueryStreamingReport` 事件（PatchUtc 之后），确认 Raw `logDate` 出现新行。
2. 新事件到达后重跑查询 01/02/03（同一 PatchUtc），重点核对 Parsed 的 `StreamingDuration_ms_` 与 `SpoolingDiskReadingDuration_ms_` 非 null 且与 Raw 一致。
3. 若 24 小时后仍无新事件，检查上游上报链路是否本身停发（与本次函数变更无关）。

---

### 脱敏复核附表（供独立复核）

| 检查 | 结果 |
| --- | --- |
| `.show database` | Gateway Monitoring，ReadWrite，v20824.0 |
| 查询 01 PrimaryResult | 2 行（Raw=0/0/0/null，Parsed=0/0/0/null） |
| 查询 02 PrimaryResult | 1 行（全 0） |
| 查询 03 PrimaryResult | 0 行 |
| `.show functions`（目标函数） | 1 行，hash_sha256(Body)=`587bba1a…703ac` |
| `.show table QueryStreamingReport policy update` | 1 条策略，IsEnabled=true，Source=GatewayReports-Raw |
| Raw 全表节奏 | 36 行；近 6h：02h=6、03h=3、06h=1（均 Patch 前） |
| Parsed 全表 | 2675 行；最新摄入 06:11:47Z（Patch 前） |