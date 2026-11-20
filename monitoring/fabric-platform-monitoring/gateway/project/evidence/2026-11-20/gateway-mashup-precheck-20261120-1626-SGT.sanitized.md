> **用户上传的本地 Eventhouse Remote MCP 只读 Precheck 报告（GitHub 备份）。** 本报告仅包含脱敏统计、字段名和 Function SHA256，未显示原始 DataSource、QueryText、数组真实数值、个人 ID。ChatGPT 没有直接查询用户的 Eventhouse。**报告中 UTC 08:27–08:36 与 SGT 16:27–16:44 的结束时刻存在 8 分钟不一致，需保留原记录并在后续查询核对。** 审核结论见 [独立评审](MASHUP_PRECHECK_REVIEW.md)。

# Gateway Mashup Parser 部署前只读核查报告（Precheck）

> 依据 `MASHUP_PRECHECK_AI_PROMPT.md`（分支 `docs/gateway-monitoring-poc-20261112`）执行，全程只读；未执行任何 Function DDL、`.create-or-alter`、`.set`、`.alter`、Rollback、Uploader 修改或 PROD 操作。

## A. 环境 / 执行时间

| 项目 | 值 |
| --- | --- |
| 目标数据库 | **Gateway Monitoring**（POC，`eventhouse-remote` item-scoped 端点，`.show database` 核实 ReadWrite） |
| MCP 工具 | `eventhouse-remote`（ExecuteKQLQueryTool，只读查询 + `.show` 管理命令） |
| 仓库 / 分支 | `ZeonZheng/fabric-toolbox` / `docs/gateway-monitoring-poc-20261112`（非 main） |
| 分支 HEAD Commit | `641408f9d982188fa1b9464ab05e5501bd3d97d3`（2026-11-20T08:03:38Z，经 `gh api` 核实） |
| 实际执行时间 | UTC 2026-11-20 08:27–08:36（UTC+8 16:27–16:44） |
| 执行查询 | 指标检查（分支原文逐字）+ 4 项补充只读诊断 + 1 项 Memory/Time 健全性复核，全部成功（2 次辅助查询语法报错后修正重试，不影响结论） |

## B. Function / Policy 状态

| 项目 | 值 |
| --- | --- |
| `parse_MashupEvaluationReport` 仍存在 | 是 |
| 现场版本 | **仍为旧版**（bag_unpack 实现，SHA256 `71c6b4026e3404eb719c5d3fb73b251e725852250b63fd02fcf90ff5c8f73a5c`），**与 Candidate（逐字段 project 版）不同** → 确认 Mashup 尚未部署，符合"部署前"前提 |
| 本轮是否执行 DDL | **否**（零写操作） |
| Update Policy | `IsEnabled: true`，Source `GatewayReports-Raw`，Query `parse_MashupEvaluationReport()`，`IsTransactional: true`，无漂移 |

## C. 六项新增数值检查（分支 `3-mashup-additional-metrics.kql` 原文执行）

Raw 样本：59 Envelope / **379 条内嵌记录**（2026-11-19 01:24 ~ 2026-11-20 08:13 UTC，近 6h 21 条、近 1h 4 条，链路活跃）。

| FieldName | TargetType | Rows | RawNonEmpty | CandidateNonNull | ConversionLoss | ChangedByCast |
| --- | --- | --- | --- | --- | --- | --- |
| AverageIODataBytesPerSecond | real | 379 | 127 | 127 | 0 | 0 |
| AveragePercentProcessorTime | real | 379 | 135 | 135 | 0 | 0 |
| **MaxIODataBytesPerSecond** | long | 379 | 127 | **0** | **127** | 0 |
| MaxPercentProcessorTime | real | 379 | 135 | 135 | 0 | 0 |
| **TotalBytes** | long | 379 | 243 | **0** | **243** | 0 |
| **TotalRows** | real | 379 | 34 | **0** | **34** | 0 |

**null 来源判别（区分 Raw 本来 null vs 有值转 null）：**
- Raw 本来为 null：AverageIO 252、AvgPercent 244、MaxIO 252、MaxPercent 244、TotalBytes 136、TotalRows 345（379 − RawNonEmpty）——这部分 Parsed null 属正常。
- **Raw 有值却转为 null（ConversionLoss）**：MaxIO 127/127、TotalBytes 243/243、TotalRows 34/34，**全部 100% 丢失**，见 D 节根因。
- 有值且成功转换的字段（AverageIO 127、AvgPercent 135、MaxPercent 135）：ConversionLoss=0、ChangedByCast=0，数值精度无损。

## D. TotalBytes 特别复核（含根因诊断）

**1. 当前 Raw 有样本：** 243/379 条记录 TotalBytes 非空 → 本轮**有实测证据**，非 Not verified。

**2. 转换结果：** `tolong(record.TotalBytes)` 对 243 条**全部失败**（CandidateNonNull=0）。

**3. 根因（脱敏形状诊断）：** Raw 值并非数值标量，而是**未解析的 JSON 数组文本**：
- 形如 `[#####]`（单元素）104 条、`[######]` 39 条、`[#######]` 42 条等；
- 多元素数组 31 条（最多 6 个元素，如 `[######,#,#,#]` 形状）；
- `parse_json` 后：**212 条单元素数组**（首元素 100% 可转 long）、**31 条多元素数组**（语义待定）。
- `TotalRows` 同构：34 条有值记录中 14 条单元素、20 条多元素（最多 6 元素）。
- `MaxIODataBytesPerSecond` 不同：127 条全部为**带小数点的标量文本**（形状 `#####.##` 等），`toreal`/`todouble` 100% 成功、`tolong` 100% 失败——即该字段实际是**小数值**，Candidate 用 `tolong` 类型选择错误。

**4. 旧 Parsed null 的原有证据与解释：** 当前 Parsed `MashupEvaluationReport` 全表 2862 行，`TotalBytes` null **2862/2862（100%）**、`TotalRows` 非空 0、`MaxIODataBytesPerSecond` 非空 0（最新摄入 08:13:46Z）。与 Preflight 记录的 2797/2797 null 一致（行数增长为持续摄入所致）。**根因即上述载荷形状**：旧 bag_unpack 函数同样无法把数组文本/小数文本转为 long/real → null。**因此"旧 Parsed 全 null"不能靠换解析函数自动修复——Candidate 现行写法会原样复现该问题。**

**5. 限制：** 多元素数组（TotalBytes 31 条、TotalRows 20 条）的语义（求和？取首元素？逐元素展开？）无法从数据本身判定，需上游/人工决策；本轮不猜测、不修改 Candidate。

## E. Schema / 既有测试状态（历史证据，本轮未重跑 DDL）

| 项目 | 状态 | 说明 |
| --- | --- | --- |
| Schema Check 17/17 | **历史通过**（2026-11-17，仓库 `schema/3-schema-check` 记录） | 本轮**未重测**，引用历史证据；本轮仅确认现场 Function/Policy 未漂移 |
| 五项 Memory/Time Dry-run | **历史通过**（仓库 `dryrun/3-dryrun` 记录） | 本轮**未重跑 DDL**；另做了一项**当前数据健全性复核**（只读）：`AverageCommit(byte)`、`AverageWorkingSet(byte)`、`MaxCommit(byte)`、`MaxWorkingSet(byte)`、`TotalProcessorTime(ms)` 五字段在当前 379 条记录上 RawNonEmpty 379/379、ConversionLoss 0、ChangedByCast 0 —— 历史结论在当前样本仍成立 |
| DataSourceKinds / DataSourcePaths | **已知数据模型设计限制** | 379/379 条 Raw 记录的 Key 集合中**均不含**这两个 Key（`DataSource` Key 379/379 存在）；Candidate 中两列保持 `tostring(record.…)` 为空属预期，**不视为缺陷、不推断、不改目标 Schema** |

## F. Verdict：`Blocked`

**判定依据（对照 Prompt 判定标准）：** "确有转换丢失" → Blocked。三项字段在当前 Raw 有值样本上 **ConversionLoss = 100%**：

| 阻塞项 | 证据 | 根因 |
| --- | --- | --- |
| `TotalBytes = tolong(record.TotalBytes)` | 243/243 丢失 | Raw 值为 JSON 数组文本（212 单元素 + 31 多元素） |
| `TotalRows = toreal(record.TotalRows)` | 34/34 丢失 | Raw 值为 JSON 数组文本（14 单元素 + 20 多元素） |
| `MaxIODataBytesPerSecond = tolong(record.MaxIODataBytesPerSecond)` | 127/127 丢失 | Raw 值为小数标量文本，`tolong` 不接受小数（`toreal` 127/127 成功） |

**结论：** Candidate 现行写法**不能**修复 Preflight 发现的 TotalBytes 全 null 问题（会原样复现），且额外引入 MaxIODataBytesPerSecond、TotalRows 两项 100% 丢值。**不满足进入手工 POC Function DDL 评审的条件**，需先修订 Candidate。

**已验证无问题项：** AverageIODataBytesPerSecond、AveragePercentProcessorTime、MaxPercentProcessorTime 三字段转换 0 丢失 0 变化；五项 Memory/Time 字段当前数据健全性复核通过；Function 仍为旧版未漂移；Policy Enabled/Source Raw/Transactional 正常；Raw 链路活跃。

**未验证项 / 待人工决策：**
1. 多元素数组的取值语义（TotalBytes 31 条、TotalRows 20 条，最多 6 元素）——需上游确认是求和、首元素还是逐元素展开，本轮无法从数据判定。
2. `MaxIODataBytesPerSecond` 目标类型：改 `toreal`（需同步表 Schema 该列 long→real）或保留 long 并明确取整规则——涉及 Schema 决策，超出本轮只读范围。
3. 修订后的 Candidate 需重做本轮六项检查 + Dry-run 后再评审。

**建议下一人工决策（供 ChatGPT 复核后定）：**
- 修订 Candidate：TotalBytes/TotalRows 先 `parse_json` 再取值（单元素场景 212/212、14/14 可恢复）；多元素语义定案后处理 31+20 条；MaxIO 改 real 或定取整规则。
- 修订完成前**不要**执行 Mashup 手工 POC DDL。
- 本轮报告不构成对任何修订方案的授权。

---

### 附：真实脱敏查询输出汇总（供独立复核）

| # | 查询 | 关键输出 |
| --- | --- | --- |
| 1 | 六项指标检查（分支原文） | 6 行：AvgIO 127/127/0/0；AvgPct 135/135/0/0；MaxIO 127/**0**/**127**/0；MaxPct 135/135/0/0；TotalBytes 243/**0**/**243**/0；TotalRows 34/**0**/**34**/0 |
| 2 | 形状诊断 | TotalBytes/TotalRows 值形状均为 `[...]` 数组文本；MaxIO 形状 `#####.##` 等小数标量；ToDouble/ToReal 成功数=有值数，ToLong 成功数=0 |
| 3 | 数组结构量化 | TotalBytes：单元素 212、多元素 31（max 6）；TotalRows：单元素 14、多元素 20（max 6）；MaxIO：标量 127 |
| 4 | Raw Key 检查 | 379 条记录：DataSourceKinds 0、DataSourcePaths 0、DataSource 379 |
| 5 | 旧 Parsed 证据 | 2862 行；TotalBytes null 2862/2862；TotalRows 非空 0；MaxIO 非空 0；最新摄入 2026-11-20T08:13:46Z |
| 6 | 五项 Memory/Time 复核 | 5 字段均 379/379 非空、0 丢失、0 变化 |
| 7 | `.show functions` | 旧版 SHA256 `71c6b402…3a5c`（非 Candidate） |
| 8 | Update Policy | IsEnabled=true；Source=GatewayReports-Raw；Query=parse_MashupEvaluationReport()；IsTransactional=true |
| 9 | Raw 节奏 | 59 Envelope / 379 记录；2026-11-19 01:24 ~ 11-20 08:13 UTC；近 6h 21、近 1h 4 |