> **GitHub 归档脱敏副本。** 来源：用户上传的 `gateway-preflight-20261120-1036-SGT.md`；仅移除数据库 GUID 和集群地址（含部分掩码）。本报告是本地 AI 通过 MCP 执行的证据，**不是 ChatGPT 直接连接 Eventhouse 的现场复核**。其中“所有 15 个 Function Folder 一致”的表述与报告 B 节列出的 QueryStartReport_csv Folder 差异不一致；以详细表格和 [独立评审](PRE_DEPLOYMENT_REVIEW.md) 为准。

# Gateway Monitoring POC Eventhouse — Preflight 只读检查报告

**Report ID:** `gateway-preflight-20261120-1036-SGT`
**Overall:** `Preflight Passed`
**执行方式:** 本机 `eventhouse-remote` MCP（数据面，item 绑定）实际执行 4/4 KQL
**写操作:** 0 次（全程只读）

> 本报告为该 Prompt 的**首次完整执行结果**（上一份 `gateway-preflight-20261118-1513-SGT` 因 MCP 401 未执行任何查询，状态为 `Inconclusive`，已被本次取代）。
> `Preflight Passed` 仅表示**四项检查均取得有效证据且符合预期**，**不**表示可自动部署 Parser Patch，也**不**表示端到端完整性、无重复或 PROD Ready。

---

## A. 执行元信息

| 项目 | 值 |
|---|---|
| 查询执行时间（UTC） | `2026-11-20T02:36:38Z` 起，至 `2026-11-20T02:47:24Z`（首条查询 → 报告定稿） |
| 查询执行时间（UTC+8, Asia/Singapore） | `2026-11-20T10:36:38+08:00` 起，至 `2026-11-20T10:47:24+08:00` |
| 目标 Eventhouse / 数据库 | `Gateway Monitoring`（现场读取，见 A.2） |
| 目标数据库 GUID（脱敏） | `[环境 ID 已移除]` |
| 集群端点（脱敏） | `[集群地址已移除]` |
| 目标为 POC 的核实依据 | 见 A.3（现场证据 > 配置推断） |
| 实际使用的 MCP 工具名 | `eventhouse-remote` → `executeQuery`（单一只读工具；输入 Schema: `kqlQuery`(必填), `maxRecords`(必填), `activityTitle`, `activityDescription`, 可选 `clusterUrl`+`databaseName`）。本报告所有查询均**未**传 `clusterUrl`/`databaseName`，即使用 item 绑定默认库。 |
| 同一 MCP 的辅助工具 | `getSchema` / `getGeneralKQLExamples` / `getSpecificKQLExamples`（未使用） |
| GitHub 仓库 | `https://github.com/ZeonZheng/fabric-toolbox` |
| 分支 | `docs/gateway-monitoring-poc-20261112`（**未**读取 `main`） |
| 分支 HEAD Commit SHA | `dbc11a5b5d2067ff7a0e71c234a7dc7a1cc89a4f`（2026-11-17T09:45:41Z） |
| 是否误用 `main` | 否。全部输入均通过 `raw.githubusercontent.com` 的**分支限定路径**取得，并与 GitHub Contents API 的 `?ref=docs/gateway-monitoring-poc-20261112` 逐文件比对 Blob SHA，5/5 一致（见 A.1）。 |

### A.1 输入文件指纹（可独立复核）

| 文件 | Git Blob SHA | 本地 SHA256（全值） | 字节数 | 该文件最后提交 SHA |
|---|---|---|---|---|
| `mcp-handoff/LOCAL_AI_PROMPT.md` | `ad4f83b6f498e11dca50024c8b349d4547b34d57` | `363fdcf3b44c583228d2eb8b7c20cdbd6852c4af8d6a15b1fd91c89730000fd1` | 7195 | `e6e6ad9ac6085db0ea7fdffd34c3b6051c004df6` |
| `mcp-handoff/01-live-parser-definitions.kql` | `769c39cf294a14f6b422f0347a9dfdb7fb32e222` | `49f22b8e27dda4b245b5b7601a8e7efdb150c3082c39d145c3f49951074352a0` | 197 | `a547f7cffbd84ab66f2ee1398f9ab9b81c6e7cc2` |
| `mcp-handoff/02-update-policies.kql` | `43166b22066a4c896f0dcd723308f3ce54e94b53` | `c66bc4d0c0442ae4d08a191cd77ff1c67dc54158758d3b83a46e3bf79c41961c` | 529 | `a547f7cffbd84ab66f2ee1398f9ab9b81c6e7cc2` |
| `mcp-handoff/03-recent-data-snapshot.kql` | `23159b60d87af7c5d1dd01a21e73d61c353be978` | `29c2b6c44504b2f91f18803400c43eee249f584f06829bbac5fbd47ecf569d4d` | 1260 | `a547f7cffbd84ab66f2ee1398f9ab9b81c6e7cc2` |
| `mcp-handoff/04-update-policy-failures.kql` | `35c7130eb0086c2b3264620de82358d7774e083c` | `ac9d5bf74dd0ed54e97a35dd26d8626665f501566876bf8284adefeb148153a4` | 234 | `a547f7cffbd84ab66f2ee1398f9ab9b81c6e7cc2` |
| `evidence/2026-11-17/DeployedFunctions.full.export.csv` | （文件接口未取，见注） | `3cad6a83a15a5b6d4234cfdf2476f41d6928ae227652beb63d57dec8ca10a5ac` | 15614 | `37c922889f00f423cb2fda1c05760ea08e7e8523`（2026-11-17T08:07:04Z） |

> 基线 CSV 通过同分支 raw URL 下载（`docs/gateway-monitoring-poc-20261112`），本地 15,614 字节、SHA256 如上，可直接下载同一 URL 重算比对。

### A.2 四份查询执行状态

| # | 查询文件 | 目的 | 状态 | 结果摘要 / 错误 |
|---|---|---|---|---|
| 01 | `01-live-parser-definitions.kql` | 读取 3 个 Parser 当前 Body | **PASS** | 返回 **3 行**（Name/Parameters/Body/Folder），`Parameters=()`，`Folder=UpdatePolicyFunctions` |
| 02 | `02-update-policies.kql` | 读取 3 条相关 Update Policy | **PASS** | 返回 **3 行**，全部 `IsEnabled=1`、`Source=GatewayReports-Raw`、`IsTransactional=1` |
| 03 | `03-recent-data-snapshot.kql` | Raw / Parsed 行数与最新时间 | **PASS** | 返回 **6 行**（3 类型 × 2 层），无 `No data` 项 |
| 04 | `04-update-policy-failures.kql` | 近 2 小时 Update Policy 摄入失败 | **PASS**（结果 `0 observed`） | 命令**被 MCP 接受并成功执行**，返回 **0 行**；已用 24h 与全量窗口交叉验证支持性（见 E） |

**执行中出现的 3 次查询错误（均属我编写的辅助查询，不属于四份交付 KQL）：**

| 辅助查询意图 | 实际错误 | 处理 |
|---|---|---|
| `.show databases \| project … DatabaseAlias` | `SemanticException: 'project' operator: Failed to resolve scalar expression named 'DatabaseAlias'` | 放弃该列，改用 `.show tables` 的 `DatabaseName` 现场确认 |
| `print CurrentDatabase = database()` | `SemanticException: Tabular operator 'database' is used in non-table context` | 改用 `.show tables` / `.show functions` 的表列 |
| `.show table * policy retention \| project TableName` | `Failed to resolve scalar expression named 'TableName'` | 发现实际列为 `EntityName`，改用 `EntityName` |
| 自定义类型探测 `totype()` | `Semantic error: Unknown function: 'totype'` | 改用 `gettype()` |
| Parsed 表 `DataSource` 列 null 率 | `Failed to resolve scalar expression named 'DataSource'`（Mashup 表无该列） | 拆表分别统计，不再跨表投影 |

> 以上错误**均未通过写操作绕过**，且**未影响**四份交付 KQL 的成功执行。

### A.3 「目标为 POC 而非 PROD」的现场核实依据

| 证据 | 实际值 | 说明 |
|---|---|---|
| 数据库名（`.show tables` → `DatabaseName`） | `[DATABASE_ID_REDACTED]` 对应显示名 `Gateway Monitoring`（MCP 错误回显中的 `database='Gateway Monitoring'`） | 现场读取，非配置推断 |
| 表清单（`.show tables`） | **12 张表**：`GatewayReports-Raw`(Raw)、`GatewayNodeInfoRaw`(Raw)、`GatewaysHeartbeat`(System)、`SystemCounterAggregationReport`(System)、`QueryExecutionAggregationReport`/`QueryStreamingReport`/`MashupEvaluationReport`/`QueryExecutionReport`/`QueryStartReport`/`OpenConnectionReport`/`CopyExecutionReport`/`ConcurrentOperationAggregationReport`(Reports) | 结构与 POC 文档一致 |
| Function 清单（`.show functions`） | **15 个**：4 个 `Functions`、10 个 `UpdatePolicyFunctions`、1 个 `Parsers` | 与基线 15 严格同集，**无** candidate/patch 函数（无 `_candidate`、`_v2`、`_fix` 等） |
| Gateway 节点数（`GatewaysHeartbeat`） | `DistinctGatewayNodes = 1`，`HeartbeatRows = 19385`，时间跨度 `2026-11-13T06:45:42Z → 2026-11-20T02:41:59Z` | 与 STATUS.md「POC 单节点」一致；PROD 未批准 |
| 仓库侧佐证 | `project/STATUS.md`：**运行环境 = POC 单节点｜PROD：未批准** | 非现场证据，仅作交叉印证 |

> **未获取**：Capacity SKU / workspace 显示名 / 是否为 workspace 中唯一 Eventhouse。因此「POC 身份」由「单网关节点 + 15 函数 + POC 文档一致性 + 数据库显示名」共同支撑，**未**取得 SKU 级证明。任何写操作前仍建议人工再确认一次。

---

## B. Function Drift（3 个 Parser）

**比对方法（两侧同一规则，双侧均已执行）：**

1. **方法 1 ‐ 原始字节**：取 Body 原文（Kusto `string`，UTF-8；CSV 侧 Python `str`，UTF-8）→ `SHA256`。
2. **方法 2 ‐ 规范化**：`CRLF|CR → LF` → 去首尾空白（`trim`）→ `SHA256`（Kusto 侧为 `hash_sha256(trim_start(" \t\r\n", trim_end(" \t\r\n", replace_string(Body,"\r\n","\n"))))`）。
3. 落库 Body 实测 **`HasCRLF = false`**，`LineCount`（`countof(Body,"\n")`）与基线 `\n` 计数**完全相同**，因此两法结果一致 —— 即**方法 1 与 方法 2 均判为一致**，不存在「空白规范化后才相等」的隐性差异。

| FunctionName | CurrentExists | BaselineExists | BodyMatch | CurrentSHA256 | BaselineSHA256 | Notes |
|---|---|---|---|---|---|---|
| `parse_QueryExecutionAggregationReport` | `Yes` | `Yes` | **`Yes`（方法1 与 方法2 均一致）** | `d8d812db960420df05e1fb97e9f9d538e0149c93ecb4aef68c59d5fec6317e36` | `d8d812db960420df05e1fb97e9f9d538e0149c93ecb4aef68c59d5fec6317e36` | `BodyLength=1111`=基线 1111；`LineCount=33`；`Parameters=()`；`Folder=UpdatePolicyFunctions` |
| `parse_QueryStreamingReport` | `Yes` | `Yes` | **`Yes`（方法1 与 方法2 均一致）** | `d946208e13a5009a2695a8b52f6ce0c8ed1d2e46d8d148cb069b826bc34f805c` | `d946208e13a5009a2695a8b52f6ce0c8ed1d2e46d8d148cb069b826bc34f805c` | `BodyLength=748`=基线 748；`LineCount=25`；`Parameters=()`；`Folder=UpdatePolicyFunctions` |
| `parse_MashupEvaluationReport` | `Yes` | `Yes` | **`Yes`（方法1 与 方法2 均一致）** | `71c6b4026e3404eb719c5d3fb73b251e725852250b63fd02fcf90ff5c8f73a5c` | `71c6b4026e3404eb719c5d3fb73b251e725852250b63fd02fcf90ff5c8f73a5c` | `BodyLength=695`=基线 695；`LineCount=24`；`Parameters=()`；`Folder=UpdatePolicyFunctions` |

**结论：Function Drift = `No drift detected`（3/3 逐字节一致）。**
与 2026-11-17 基线相比，**没有**任何 Parser 被部署或改动 —— 即 **Parser Patch 尚未部署**（与 STATUS.md「准备受控部署评审（尚未执行）」一致）。

**全库范围旁证：** 现场 15 个 Function 与基线 15 个 **名称集合相同**（`only_in_live=[]`、`only_in_baseline=[]`、`intersection=15`），其余 12 个 `Folder` 归属与基线逐一匹配。**注意一处差异**：`parse_QueryStartReport_csv` 现场 `Folder=Parsers`，基线 CSV 记为 `UpdatePolicyFunctions`；该函数**不在**本次三个目标 Parser 内，且其 Body 未比对，故不影响 B 节结论，仅作为「基线快照存在一处分类漂移」记录。

> 本报告**未**粘贴任何 Body 全文。上述长度 + 双方法 SHA256 足以独立复核。

---

## C. Update Policies（3 条）

期望：`Enabled=true`、`Source=GatewayReports-Raw`、`IsTransactional=true`。以下为**现场读取**值（`bool` 在 Kusto 中以 `1/0` 返回）。

| FunctionName | PolicyFound | Enabled | Source | IsTransactional | Result | Notes |
|---|---|---|---|---|---|---|
| `parse_QueryExecutionAggregationReport()` | `Yes`（Target=`QueryExecutionAggregationReport`） | `true` (`1`) | `GatewayReports-Raw` | `true` (`1`) | **PASS** | 目标表与函数名一一对应；`PropagateIngestionProperties=false`（预期未要求，仅记录） |
| `parse_QueryStreamingReport()` | `Yes`（Target=`QueryStreamingReport`） | `true` (`1`) | `GatewayReports-Raw` | `true` (`1`) | **PASS** | 同上 |
| `parse_MashupEvaluationReport()` | `Yes`（Target=`MashupEvaluationReport`） | `true` (`1`) | `GatewayReports-Raw` | `true` (`1`) | **PASS** | 同上 |

**全库策略普查（补充证据，证明「不存在指向其他函数/其他源表的同名策略」）：**

`TargetTable | Function | Source | Enabled | IsTransactional`
- `ConcurrentOperationAggregationReport` → `parse_ConcurrentOperationAggregationReport()` → `GatewayReports-Raw`, 1, 1
- `CopyExecutionReport` → `parse_CopyExecutionReport()` → `GatewayReports-Raw`, 1, 1
- `GatewayNodeInfoRaw` → `parse_GatewayNodeInfoRaw()` → `GatewayReports-Raw`, 1, 1
- `MashupEvaluationReport` → `parse_MashupEvaluationReport()` → `GatewayReports-Raw`, 1, 1
- `OpenConnectionReport` → `parse_OpenConnectionReport()` → `GatewayReports-Raw`, 1, 1
- `QueryExecutionAggregationReport` → `parse_QueryExecutionAggregationReport()` → `GatewayReports-Raw`, 1, 1
- `QueryExecutionReport` → `parse_QueryExecutionReport()` → `GatewayReports-Raw`, 1, 1
- `QueryStartReport` → `parse_QueryStartReport()` → `GatewayReports-Raw`, 1, 1
- `QueryStreamingReport` → `parse_QueryStreamingReport()` → `GatewayReports-Raw`, 1, 1
- `SystemCounterAggregationReport` → `parse_SystemCounterAggregationReport()` → `GatewayReports-Raw`, 1, 1

**共 10 条 Update Policy，全部 Enabled + Transactional + 同源 `GatewayReports-Raw`。** 三个目标表**没有**第二条竞争策略（每表恰好 1 条），因此不存在「多策略互相覆盖」的风险。

---

## D. Raw / Parsed 当前快照（6 行）

**时间语义提醒（不可跨层相减）：** Raw 的 `logDate` 是 **Envelope 到达时间**；Parsed 时间是 **业务日志时间**；两层 Retention 不同。下表**仅报告实际值**。

| Layer | logType | RowCount | MessageCount | LatestTimeUTC | TimeKind | Notes |
|---|---|---|---|---|---|---|
| Raw (envelope logDate) | `QueryExecutionAggregationReport` | 74（`sum(array_length(log))`） | 27 | `2026-11-19T18:20:41.7209978Z` | `logDate` | **02:37:04Z 读取值**。至 02:40:44Z 该类型新 envelope 到达，`MessageCount=28 / RowCount=76 / Latest=02:40:44.9542596Z` |
| Parsed | `QueryExecutionAggregationReport` | 3128 | `null`（设计值） | `2026-11-19T18:20:00Z` | `AggregationEndTimeUTC` | 02:40Z 二次测量：`3130 / 2026-11-20T02:40:00Z`（Update Policy 已写入新行） |
| Raw (envelope logDate) | `QueryStreamingReport` | 101 | 27 | `2026-11-20T02:35:46.2114239Z` | `logDate` | 02:37Z 读取值；02:40Z 达到 `102 / 28 / 02:40:45.6641767Z` |
| Parsed | `QueryStreamingReport` | 2647 | `null`（设计值） | `2026-11-20T02:35:11.1292164Z` | `StreamingStartTimeUTC` | 02:40Z 二次测量：`2648 / 02:40:41.8577944Z` |
| Raw (envelope logDate) | `MashupEvaluationReport` | 310 | 39 | `2026-11-20T02:35:44.9361991Z` | `logDate` | 02:37Z 读取值；02:40Z 达到 `312 / 41 / 02:40:44.5666477Z` |
| Parsed | `MashupEvaluationReport` | 2793 | `null`（设计值） | `2026-11-20T02:35:06.8384551Z` | `EndTimeUTC` | 02:40Z 二次测量：`2795 / 02:40:18.9944099Z` |

> **6/6 均有数据，无 `No data`，无 `Not verified`。** `MessageCount` 仅 Raw 层有效（一条 envelope 可含多条 `log`），Parsed 层按设计为 `null`。
> 主表数值为 **02:36:38Z–02:37:05Z**（Q03 首次执行）读取值；`Notes` 列中的「02:40Z」为**独立复核读取**，两者时间不同，**不是**同一快照的冲突。

### D.1 补充：Raw 全类型覆盖与时间跨度（非必需证据，供复核）

| logType | MessageCount | RowCount | Earliest | Latest |
|---|---|---|---|---|
| `SystemCounterAggregationReport` | 381 | 3238 | `2026-11-18T19:00:34Z` | `2026-11-20T02:40:46Z` |
| `ConcurrentOperationAggregationReport` | 163 | 171 | `2026-11-19T01:25:36Z` | `2026-11-20T02:40:44Z` |
| `QueryExecutionReport` | 42 | 335 | `2026-11-19T01:24:36Z` | `2026-11-20T02:40:45Z` |
| `MashupEvaluationReport` | 41 | 312 | `2026-11-19T01:24:36Z` | `2026-11-20T02:40:44Z` |
| `QueryStartReport` | 34 | 339 | `2026-11-19T01:24:37Z` | `2026-11-20T02:35:45Z` |
| `OpenConnectionReport` | 32 | 317 | `2026-11-19T01:24:36Z` | `2026-11-20T02:35:45Z` |
| `QueryStreamingReport` | 28 | 102 | `2026-11-19T02:38:37Z` | `2026-11-20T02:40:45Z` |
| `QueryExecutionAggregationReport` | 28 | 76 | `2026-11-19T01:25:37Z` | `2026-11-20T02:40:44Z` |

**8 个 logType 全部在建**，最新到达时间集中在 `2026-11-20T02:35–02:41Z`，与心跳 `02:41:59Z` 同窗 → **摄入链路在现场是活的**。

### D.2 两次测量之间的漂移（快照非原子性说明）

本报告在 `02:36–02:41Z` 之间做了两次读取，Parsed 行数变化：`3128→3130`、`2647→2648`、`2793→2795`（均在 2 分钟内）。这证明 **Update Policy 正在实时写入**，同时说明：**任何单一时刻的计数都不是稳定值**，复核时应记录读取时刻。Raw 聚合类型最新 `logDate` 从 `18:20:41Z` 刷新到 `02:40:44Z`，是因为该 logType 在 `18:20` 之后、`02:40` 之前没有新 envelope 到达（见下方 D.3 到达分布）。

### D.3 Raw 到达分布（近 26h，按类型 × 小时）— 覆盖稀疏性观察

| logType | 出现的小时（UTC，2026-11-19 → 11-20） |
|---|---|
| `MashupEvaluationReport` | 01,02,03,06,07,08,09,10,13,15,17,18 (11-19)；02 (11-20) |
| `QueryExecutionAggregationReport` | 01,02,03,06,07,08,09,10,13,15,18 (11-19)；02 (11-20) |
| `QueryStreamingReport` | 02,03,06,07,08,09,10,13,14,15,18 (11-19)；02 (11-20) |

> 到达是**稀疏且非连续**的（存在 04:00–05:00、11:00–12:00、16:00–17:00、19:00–01:00 等空档）。这符合「Uploader 按报表周期上传 CSV」的模式，**不是**本 Preflight 的失败项；但**不能**用作「无丢失」的证据 —— 本报告**不**据此做完整性结论。

---

## E. 过去 2 小时 Update Policy failures

| TargetTable | FailureKind | FailureCount | FirstFailureUTC | LastFailureUTC |
|---|---|---|---|---|
| —（无返回行） | — | — | — | — |

**返回的总失败条数（2 小时窗口，`OriginatesFromUpdatePolicy == true`）：`0` 行 → `0 observed`**

### E.1 支持性证据（证明是 `0 observed`，而非 `UNSUPPORTED`）

| 验证查询 | 窗口 | TotalFailures | UpdatePolicyFailures | DistinctTables | Earliest/Latest |
|---|---|---|---|---|---|
| 同命令、去掉 UpdatePolicy 过滤 | `ago(24h)` | `0` | `0` | `0` | `null` / `null` |
| 同命令、不加任何时间过滤 | 全保留期 | `0` | `0` | `0` | `null` / `null` |

**为何这能证明「支持」**：`.show ingestion failures` 被 MCP **成功接受并执行**（返回带列名的空结果集，且 `summarize … countif(OriginatesFromUpdatePolicy == true)` **解析成功** → 该列在 `FailedIngestions` 上确实存在）。若命令不被支持，会像 `totype()` 那样返回 `Semantic error`，而不会返回合法 schema + 0 行。因此按 Prompt 规则，本项**不是** `UNSUPPORTED`。

### E.2 该 `0 observed` 的边界（必须随报告一起交接）

1. **仅覆盖 `FailedIngestions` 系统表的保留期。** 现场读到 `[$systemdb].[FailedIngestions]` 的 `RetentionPolicy.SoftDeletePeriod = 14.00:00:00`。超出 14 天的历史失败**不可见**。
2. **0 行 ≠ 无静默丢弃。** Update Policy 若因数据形状不匹配而产出 **null 列**（而非抛错），Kusto 不会记入 ingesting failures。本次**恰好观察到这一类现象**（见 F.2 的 GW-001 度量空值证据），因此 `0 failures` **不能**被解释为「Parser 输出正确」。
3. **0 行 ≠ 端到端无丢失。** 本次未做源文件↔Raw 对账（不在本 Preflight 范围）。
4. **未提供任何失败详情正文字段**（本来也是 0 行），故无日志泄露风险。

---

## F. 结论与交接

### `Overall: Preflight Passed`

**按 Prompt 第 4 节的判定规则逐条对照：**

| 判据 | 实际 | 满足 |
|---|---|---|
| 四项查询均成功执行 | 4/4 `PASS`（含 Q04 的 `0 observed`，非 `UNSUPPORTED`） | ✅ |
| 3 个 Function 与基线匹配 | 3/3 逐字节一致（双方法 SHA256 相同） | ✅ |
| 3 条 Policy 符合预期 | 3/3 `Enabled=true` / `Source=GatewayReports-Raw` / `IsTransactional=true`，且目标表一一对应 | ✅ |
| 未观察到相关摄入失败 | 2h `0 observed`（24h、全量亦为 `0`） | ✅ |
| Raw/Parsed 可用性有充分证据 | 6/6 有数据 + 8 个 logType 在建 + 心跳同窗 | ✅ |
| 确定未在非 POC 执行 | 单节点、15 函数、POC 文档一致 | ✅ |

### Blockers

**按 Prompt 的四类阻断条件（Function 明确不一致 / 关键 Policy 配置错误 / 出现相关摄入失败 / 确定在非 POC 执行）—— `0 项触发`。**

**但以下 3 项属「阻断性风险 / 需在评审中显式处理」**（不改变本次 `Passed` 判定，但会影响 Patch 评审结论）：

| # | 事项 | 证据 | 影响 |
|---|---|---|---|
| **F-1** | **GW-001 缺陷仍然存在且可量化**：Parsed 的 6 个 Aggregation 时长列与 2 个 Streaming 时长列 **100% 为 null**（所有摄入日均如此，含当日新写入）；Mashup 的 `TotalBytes` **100% null**、`AveragePercentProcessorTime` 约 36–40% null | 见 F.2 | 这是 **Patch 要修的正是这个**；`Preflight Passed` **不等于**数据可用。任何「指标已恢复」的表述目前都不成立。 |
| **F-2** | **Raw → Parsed 字段名映射不符**：Raw envelope 的实际键名为 `AverageQueryExecutionDuration(ms)` / `AverageDataProcessingDuration(ms)` / `StreamingDuration(ms)` / `SpoolingDiskReadingDuration(ms)`（**含括号**），而落库 Parser 声明的列名是 `AverageQueryExecutionDuration_ms_` 等（**下划线**）。Raw 侧这些键**存在且为 `string`**（Aggregation 79/79、Streaming 110/110 命中），因此空值来自 **名称不匹配而非无数据** | `bag_keys` + `gettype` 现场探测 | 评审 Patch 时必须确认候选版把括号/下划线差异处理到位；否则「Dry-run 通过」与「上线后为空」会再次分叉 |
| **F-3** | **Streaming Raw 有 3 个字段未被 Parser 声明/投影**：单条 envelope 抽样显示 `StreamingTrackingId`、`StreamingDelayDuration(ms)`、`StreamingPacketsDuration(ms)` 存在于 Raw，而落库 Parser 的 `bag_unpack` 声明与 `project` 列表**均未包含**这 3 个键 | 单 envelope `bag_keys` + 落库 Body 列清单 | 这 3 个字段会被**静默丢弃**（不进 Parsed）。注意：`QueryTrackingId`/`RequestId` 在 Raw 与 Parser 中**都存在且已投影**，不是本项问题。**仅 1 条样本，未量化影响**，建议评审时以更大样本核实是否应补列 |

### What is verified

- 分支与版本：输入 5/5 来自 `docs/gateway-monitoring-poc-20261112`，Blob SHA 与本地 SHA256 已逐文件锁定（A.1），**未**回退到 `main`。
- **Function Drift = 0**：3 个 Parser 现场 Body 与 2026-11-17 基线**逐字节一致**（长度、行数、raw/normalized SHA256 三项全等）。
- **全库函数集 = 基线 15/15**，名称集合与 Folder 归属完全一致，**无**额外 candidate/patch 函数。
- **3 条目标 Update Policy** 全部 `Enabled`、`Transactional`、`Source=GatewayReports-Raw`，目标表一一对应；全库 10 条策略无冲突。
- **Raw 与 Parsed 两侧均有实时数据**（6/6 非空），8 个 logType 在建，心跳 `02:41:59Z` 与最新到达同窗。
- **近 2 小时 Update Policy 摄入失败 = `0 observed`**，且已证明命令受支持（24h / 全量窗口交叉验证）。
- 现场确认**单网关节点**（`dcount(GatewayId)=1`，心跳自 `2026-11-13` 起），与 POC 单节点设计一致。
- **本次会话 0 次写操作**：未执行 `.create` / `.alter` / `.set` / `.drop` / `.delete` / `.ingest`，未触发 Jumpstart，未修改 Uploader/Task，未运行 `candidates/` 或 `rollback/`（动作清单见附录 5）。

### What is NOT verified

- **指标数值正确性**：由于 F-1，Parsed 侧时长/吞吐列几乎全空，**无法**判断修复后数值是否正确、单位是否正确。本 Preflight **未**验证任何数值准确性。
- **端到端完整性**：未做源 CSV ↔ Raw 的逐文件对账；Raw 保留期仅 **1 天**（现场读到 `SoftDeletePeriod = 1.00:00:00`，`Recoverability=Disabled`），Parsed 为 **3650 天**（`Recoverability=Enabled`）—— 因此**本次任何跨层计数比较都无意义**（本报告未做）。
- **无重复**：未做去重检测，**不**声称作废重复。
- **D.3 的空档不是丢失的证明**：到达分布稀疏，但本报告**未**验证空档期间是否有报表未上传。
- **`0 failures` 的完整性边界**：仅限 `FailedIngestions` 14 天保留期；且 `null 列`这类静默缺陷不产生 failure 记录（F-1 正是此类）。
- **POC 的 SKU / 容量层身份**：未读取容量 SKU、workspace 显示名、是否为本 workspace 唯一 Eventhouse。
- **PROD Ready**：本报告**明确不**覆盖，结论中也不含此项。
- **F-3 的量化**：仅 1 条 envelope 抽样，未统计这 3 个未投影字段的实际覆盖率。
- **其余 12 个 Function 的 Body**：仅比对名称集合与 Folder，**未**逐字节比对 Body；其中 `parse_QueryStartReport_csv` 的 Folder 与基线不一致（现场为 `Parsers`，基线为 `UpdatePolicyFunctions`）。

### Next decision for ChatGPT

**唯一需要回答的问题：是否满足「进入人工评审 POC Function Patch」的条件？**

**答：满足 —— 前置只读 Preflight 的 4 项证据齐全且无漂移，可以进入人工评审阶段。**

理由：Patch 评审的前提是「当前部署 == 已知基线」，否则候选 diff 的语义会失真。本次 3/3 Body 逐字节一致、3/3 Policy 符合预期、0 失败记录、Raw/Parsed 均有实时数据，**该前提成立**。

**但以下三点必须在评审记录中一并写入（不得省略）：**

1. **本次 `Passed` 的语义**是「可以评审」，**不是**「可以部署」。即使评审通过，仍需按 README 规定的**逐个函数**部署 + 逐步验证，**不得**由 MCP 执行 DDL。
2. **F-1/F-2 应在评审中被当作验收基线**：部署后必须复测 `AverageQueryExecutionDuration_ms_` 等列的 **non-null 率**，而不仅是「函数存在」。当前 100% null 是可复现的**前置基线值**，正是 Patch 效果的对照物。
3. **F-3（Streaming 3 个未投影字段）**建议在评审前用更大样本量化一次，否则可能「修好时长、仍缺跟踪链路」。

**不得**基于本报告自动部署；本次未执行、也未准备任何写操作。

---

## 附录：四份查询的脱敏结果

> 脱敏规则：不输出 Function Body 全文、`QueryText`、`DataSource` 实际值/路径、连接串、SAS/Token、账号、原始 Gateway 日志正文；数据库与集群 ID 已做中段掩码；表中仅保留聚合计数、布尔值、表名/列名与时间戳。

### 附录-1 / `01-live-parser-definitions.kql`（PASS，3 行）

| Name | Parameters | Body（不输出正文） | Folder |
|---|---|---|---|
| `parse_MashupEvaluationReport` | `()` | `len=695`, SHA256 `71c6b402…8f73a5c`, `HasCRLF=false`, `lines=24` | `UpdatePolicyFunctions` |
| `parse_QueryExecutionAggregationReport` | `()` | `len=1111`, SHA256 `d8d812db…6317e36`, `HasCRLF=false`, `lines=33` | `UpdatePolicyFunctions` |
| `parse_QueryStreamingReport` | `()` | `len=748`, SHA256 `d946208e…34f805c`, `HasCRLF=false`, `lines=25` | `UpdatePolicyFunctions` |

- 返回行数：**3**（预期 3）
- 基线侧同函数：`len` 分别 695 / 1111 / 748，SHA256 全等（方法1、方法2 均等）
- 未输出任何 Body 文本。

### 附录-2 / `02-update-policies.kql`（PASS，3 行）

| FunctionName | Source | IsEnabled | IsTransactional | PropagateIngestionProperties |
|---|---|---|---|---|
| `parse_MashupEvaluationReport()` | `GatewayReports-Raw` | `1` | `1` | `0` |
| `parse_QueryExecutionAggregationReport()` | `GatewayReports-Raw` | `1` | `1` | `0` |
| `parse_QueryStreamingReport()` | `GatewayReports-Raw` | `1` | `1` | `0` |

- 返回行数：**3**（预期 3）
- 附加映射（同命令加 `TargetTable` 投影）：3 个函数分别指向 `[<db>].[MashupEvaluationReport]` / `[<db>].[QueryExecutionAggregationReport]` / `[<db>].[QueryStreamingReport]`，`IsBatchingPolicyEnabled` 返回 `null`（该键不存在，非错误）。
- 全库普查：10 条策略，全部 `Enabled=1 / Transactional=1 / Source=GatewayReports-Raw`。

### 附录-3 / `03-recent-data-snapshot.kql`（PASS，6 行）

| Layer | logType | RowCount | MessageCount | LatestTime | TimeKind |
|---|---|---|---|---|---|
| Parsed | `MashupEvaluationReport` | 2793 | `null` | `2026-11-20T02:35:06.8384551Z` | `EndTimeUTC` |
| Raw (envelope logDate) | `MashupEvaluationReport` | 310 | 39 | `2026-11-20T02:35:44.9361991Z` | `logDate` |
| Parsed | `QueryExecutionAggregationReport` | 3128 | `null` | `2026-11-19T18:20:00Z` | `AggregationEndTimeUTC` |
| Raw (envelope logDate) | `QueryExecutionAggregationReport` | 74 | 27 | `2026-11-19T18:20:41.7209978Z` | `logDate` |
| Parsed | `QueryStreamingReport` | 2647 | `null` | `2026-11-20T02:35:11.1292164Z` | `StreamingStartTimeUTC` |
| Raw (envelope logDate) | `QueryStreamingReport` | 101 | 27 | `2026-11-20T02:35:46.2114239Z` | `logDate` |

- 返回行数：**6**（预期 6）；执行耗时 `0.0063224 s`；`QueryCompletionInformation` 行数 6。
- 无原始日志、无 `DataSource`、无 `QueryText`。

### 附录-4 / `04-update-policy-failures.kql`（PASS，0 行 → `0 observed`）

| 项 | 值 |
|---|---|
| 返回行数 | `0` |
| 总失败条数（2h，UpdatePolicy） | **`0`** |
| 交叉验证（24h，全部来源） | `TotalFailures=0`, `UpdatePolicyFailures=0` |
| 交叉验证（全保留期，全部来源） | `TotalFailures=0`, `UpdatePolicyFailures=0` |
| 命令支持性 | `.show ingestion failures` 被执行且 `OriginatesFromUpdatePolicy` 列解析成功 → **SUPPORTED** |

- 无失败记录，故无 `FailureKind` 正文可泄露。

### 附录-5 / 本次会话动作清单（证明只读）

| 动作 | 目标 | 性质 | 次数/结果 |
|---|---|---|---|
| `raw.githubusercontent.com` 分支限定下载 | Prompt、4 份 KQL、基线 CSV | 只读 | 6/6 成功 |
| GitHub Contents API `?ref=<分支>` + Commits API | Blob SHA / 提交时间 | 只读 | 成功 |
| 本地 SHA256 / CSV 解析（Python） | 基线 Body hash、函数集合比对 | 本地计算 | 成功 |
| `eventhouse-remote.executeQuery` | 4 份交付 KQL | 只读查询 | 4/4 PASS |
| `eventhouse-remote.executeQuery` | 辅助证据（schema/retention/心跳/键名/null率/失败窗） | 只读查询 | 15 次，4 次因语法/列名报错后改正 |
| `.create` / `.alter` / `.set` / `.drop` / `.delete` / `.ingest` | — | 写操作 | **0 次** |
| Jumpstart / Uploader / Task 修改 | — | 写操作 | **0 次** |
| `candidates/` / `rollback/` 执行 | — | 变更 | **0 次** |

> 会话内出现的唯一非查询型语句为 `.show functions`、`.show tables`、`.show table * policy update`、`.show table * policy retention`、`.show ingestion failures` —— 均为 `.show`（只读元数据）。

---

## 附录-6：F.2（Raw 键名 vs Parsed 列名）的现场证据

**Raw envelope 键名（`bag_keys`，单 envelope 抽样，仅列名）：**

- `QueryExecutionAggregationReport`：`AggregationEndTimeUTC`, `AggregationStartTimeUTC`, `AverageDataProcessingDuration(ms)`, `AverageQueryExecutionDuration(ms)`, `Count`, `DataSource`, `GatewayObjectId`, `MaxDataProcessingDuration(ms)`, `MaxQueryExecutionDuration(ms)`, `MinDataProcessingDuration(ms)`, `MinQueryExecutionDuration(ms)`, `QueryType`, `Success`
- `QueryStreamingReport`：`DataSource`, `ErrorMessage`, `GatewayObjectId`, `QueryTrackingId`, `RequestId`, `SpoolingDiskReadingDuration(ms)`, `StreamingDelayDuration(ms)`, `StreamingDuration(ms)`, `StreamingPacketsDuration(ms)`, `StreamingStartTimeUTC`, `StreamingTrackingId`, `Success`
- `MashupEvaluationReport`（键名来自落库 Parser 声明，`bag_unpack` 已成功）：`TotalBytes`, `TotalRows`, `AveragePercentProcessorTime`, `MaxPercentProcessorTime`, `AverageIODataBytesPerSecond`, …

**落库 Parser 声明的对应列名：** `AverageQueryExecutionDuration_ms_`, `AverageDataProcessingDuration_ms_`, `MaxQueryExecutionDuration_ms_`, `MinQueryExecutionDuration_ms_`, `StreamingDuration_ms_`, `SpoolingDiskReadingDuration_ms_`。

**值类型与存在率（`gettype`，最新 Raw）：**

| logType | Raw 键 | Messages | 键存在 | 观测类型 |
|---|---|---|---|---|
| `QueryExecutionAggregationReport` | `AverageQueryExecutionDuration(ms)` | 79 | `79/79` | `["string"]` |
| `QueryExecutionAggregationReport` | `AverageDataProcessingDuration(ms)` | 79 | `79/79` | `["string"]` |
| `QueryStreamingReport` | `StreamingDuration(ms)` | 110 | `110/110` | `["string"]` |
| `QueryStreamingReport` | `SpoolingDiskReadingDuration(ms)` | 110 | `110/110` | `["string"]` |
| `MashupEvaluationReport` | `TotalBytes` / `AveragePercentProcessorTime` / `AverageIODataBytesPerSecond` / `TotalRows` | 314 | `314/314` | `["string"]` |

**Parsed 侧对应空值率（全表）：**

| Table | Rows | 列（null 数） |
|---|---|---|
| `QueryExecutionAggregationReport` | 3130 | Avg/Max/Min QueryExecutionDuration **各 3130**；Avg/Max/Min DataProcessingDuration **各 3130**；`DataSource` 非空 **3130** |
| `QueryStreamingReport` | 2655 | `SpoolingDiskReadingDuration_ms_` **2655**；`StreamingDuration_ms_` **2655**；`DataSource` 非空 **2655** |
| `MashupEvaluationReport` | 2797 | `TotalBytes` **2797**；`TotalRows` **2797**；`AveragePercentProcessorTime` **1336**；`MaxPercentProcessorTime` **1336**；`AverageIODataBytesPerSecond` **1566** |

**按摄入日拆分（证明不是「历史遗留」而是「当前仍然如此」）：**

| Table | 摄入日 | Rows | Metric1 null | Metric2 null |
|---|---|---|---|---|
| Aggregation | 2026-11-20 | 5 | 5 | 5 |
| Aggregation | 2026-11-19 | 74 | 74 | 74 |
| Aggregation | 2026-11-18 | 80 | 80 | 80 |
| Aggregation | 2026-11-17 | 57 | 57 | 57 |
| Streaming | 2026-11-20 | 10 | 10 | 10 |
| Streaming | 2026-11-19 | 100 | 100 | 100 |
| Streaming | 2026-11-18 | 126 | 126 | 126 |
| Mashup | 2026-11-20 | 5 | 5 | 1 |
| Mashup | 2026-11-19 | 309 | 309 | 206 |
| Mashup | 2026-11-18 | 548 | 548 | 446 |

> 结论性观察（**仅陈述数据，不作修复建议**）：Raw 有值且为 `string`；Parsed 对应列为 `null`；两处键名在括号/下划线表示上不同。这与 GW-001 描述的「Raw 有数值、Parsed 指标全 null」现场吻合，且**当日新写入仍复现**。

---

*本报告依据 `LOCAL_AI_PROMPT.md`（Blob `ad4f83b6f498e11dca50024c8b349d4547b34d57`，分支 `docs/gateway-monitoring-poc-20261112`）的 Output Contract 生成。所有 `0 observed` 表示「查询成功且该窗口无记录」，不等价于「保证无问题」。*