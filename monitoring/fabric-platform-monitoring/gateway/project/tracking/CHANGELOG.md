# 项目进展时间线

> 时间统一采用 **Asia/Singapore（UTC+8）**；较早的数据时间若标注 UTC 则保留原始时区。这里记录历史事件；**最新状态只看 [STATUS](../STATUS.md)**。

## 2026-11-23 — Mashup 最小 Precheck：五个字段 Dry-run 通过，12 列兼容性待验证

- 用户上传本地 Eventhouse MCP 报告 `gateway-mashup-minimal-precheck-20261123.md`：现场旧 Mashup Function / Transactional Update Policy 无漂移；最近 Raw 15 Envelope / 36 行；Memory 四字段 33/33、ProcessorTime 36/36 全部无转换损失；Candidate Schema 17/17 匹配。**现场未执行任何 DDL**。
- 本次独立复核指出：报告称“3 行五项均为空”与 ProcessorTime 36/36 不一致；“回滚后无数据损坏”不能保证。报告所谓 v2 与旧 Blocked Candidate **函数体完全相同**，仅验收范围因 ADR-009 改变。
- 为避免整函数 `bag_unpack → mv-expand + project` 引发非目标字段回归，新增 [12 列只读兼容性 Gate](../kql/poc-parser-fix-20261117/value-checks/7-mashup-legacy-candidate-compatibility.kql)；保留 [独立评审](../evidence/2026-11-23/MASHUP_MINIMAL_PRECHECK_REVIEW.md)；必须有 Raw 数据、行数及多重签名计数完全匹配才可进入部署评审。
- **没有改 POC/PROD Eventhouse、Table、Policy、Uploader、Scheduled Task**；Mashup 尚未部署，P2 可靠性仍未验收。

## 2026-11-23 — Mashup 范围决策：三个非必需指标转 Deferred Backlog
- 用户确认 `TotalBytes`、`TotalRows`、`MaxIODataBytesPerSecond` **不是当前 Gateway Monitoring POC/计划 PROD 监控的必需指标**，决定采用 [方案 A](../docs/MASHUP_DESIGN_OPTIONS.md)。
- [ADR-009](DECISIONS.md) 正式记录范围决定，[GW-009](ISSUES.md) 单独跟踪这三个指标的 JSON 数组语义、浮点精度及当前 Parsed null。**非必需不等于已修复**；不再为三字段阻塞 P1，也暂不向微软询问、不重复同类统计查询。
- GW-001 继续跟踪**必需**的 Mashup/其它 Parser 字段正确性；原 Mashup Candidate 仍 Blocked，Live Function / Schema / Update Policy / PROD 均未修改。本次仅修改 GitHub 项目文档、范围与验收口径。
- 下一步核查剩余必需的 Mashup 指标及实际新数据是否正确，若确实缺失再讨论最小修复；随后重点推进 Collector 端到端可靠性和 PROD 前置条件。

## 2026-11-23 09:50–09:55（UTC+8）— Mashup Array/Decimal Shape 已验证，进入设计范围评审
- 用户的本地 Eventhouse Remote MCP 按工作分支运行三份只读查询：Query 05 原文成功，Query 04/06 的 `floor(x)` 报 `bin() expects 2 arguments`，本地修为 `floor(x,1.0)` 后成功。**已把最小语法修正提交 GitHub 的只读 KQL**。
- Raw 1 天窗口 **36 条**：TotalBytes 23 条（17 单、6 多）共 **44/44 long 可转换元素**；TotalRows 12 条（4 单、8 多）共 **35/35 long 可转换元素**；两个数组同时出现 **6/6 长度相等**，但含义与是否可以聚合仍未证实。MaxIO 14 有值，**12/14** 含小数；目标表 long 无法精确保留。
- 独立检查 Fork 版本化 Fabric Report `report.json`、6 个业务 Semantic Model TMDL 与 KQL Dashboard JSON，**未发现** `MashupEvaluationReport` / `TotalBytes` / `TotalRows` / `MaxIODataBytesPerSecond` 的**直接文字消费**；本地 AI GitHub Code Search=0 不是资产不存在的证明。不能推断已部署报表/PBIT/其它消费者均未使用。
- 归档 [2026-11-23 原始报告与独立评审](../evidence/2026-11-23/README.md)，新增 [Mashup 范围与设计选项](../docs/MASHUP_DESIGN_OPTIONS.md)；**Mashup Function、Candidate、Schema、Policy、PROD 均未改**，旧版 Blocked 约束不变。
- **不再反复跑同类形状统计**；下一步先确认这三个字段对 POC/PROD 报表是否必要，若必要再向 Microsoft 询问数组元素/聚合定义，之后才决定标量规则或另建明细结构。

## 2026-11-20 16:27–16:44（UTC+8；报告结束时间元数据有差异）— Mashup Precheck Blocked
- 本地 `eventhouse-remote` MCP **只读**检查旧版 Mashup Function / Transactional Update Policy，现场未执行 Function DDL、Schema 或 Policy 修改。59 Raw Envelope / **379 内嵌记录**，查询均执行成功（两条辅助查询先报语法错再只读修正）。
- **三个有值字段在现有 Candidate 中全部转换失败：** TotalBytes **243/243** (`long`；Raw 为 JSON 数组字符串，212 单元素+31 多元素)、TotalRows **34/34** (`real`；14 单元素+20 多元素)、MaxIODataBytesPerSecond **127/127** (`long`；Raw 是小数字符串，`toreal` 可读)。旧 Parsed 2862 行三列均 0 非空；**不是新部署造成的回归，但现有 Candidate 也修不好**。
- 当前五个 Memory/Time 字段与另外三项 CPU/IO 指标样本转换正常；此前 17/17 Schema Match 只证明列结构，不能掩盖实际数值失败；`ChangedByCast=0` 在转换为 null 时没有精度验收意义。
- 已归档 [用户报告](../evidence/2026-11-20/gateway-mashup-precheck-20261120-1626-SGT.sanitized.md) 和 [独立评审](../evidence/2026-11-20/MASHUP_PRECHECK_REVIEW.md)；GW-001 继续 Blocked（Mashup 子项）、GW-005 增加多值与数值类型设计问题。两项已通过的 Streaming/Aggregation 不变。
- 新增 3 个只读 KQL，检查数组形状、长度关联、数组内部元素类型与 MaxIO 小数；[本地 MCP 输出规范](../kql/poc-parser-fix-20261117/value-checks/MASHUP_SHAPE_FOLLOWUP_AI_PROMPT.md)。**未改 Mashup Candidate / 正式 KQL Schema，未动 PROD。**

## 2026-11-20 15:47–15:51（UTC+8）— Aggregation POC 新数据功能回归 Passed
- 用户通过本地 `eventhouse-remote` MCP 只读执行 09 Activity Gate → 06/07/08；真实 PatchUtc `2026-11-20T07:24:24.1258875Z`。8/8 Raw 类型在 Patch 后有 Envelope；Aggregation Raw **2 Envelope / 7 行**、Parsed **7 行**。
- QueryExecution Duration 三列 Raw/Parsed 均 **7/7 非空**；DataProcessing Duration 三列均 **1/7 非空**（Raw 原本有 6/7 空值）。组合 Key **2/2** 命中；六字段的非空值均匹配，**0 mismatch**。两条以上记录共享组合 Key，**并非 7/7 逐行配对**。
- Live Function Body 与 Candidate 一致，Update Policy Enabled/Transactional，Post-Patch `0 observed` ingestion failures。首次 Inconclusive 被真实新数据解决；**POC 字段功能回归 Passed，长期可靠性未验收**。
- 归档 [原始脱敏 Follow-up](../evidence/2026-11-20/gateway-aggregation-followup-20261120-1547-SGT.sanitized.md) 与 [独立评审](../evidence/2026-11-20/AGGREGATION_FOLLOWUP_REVIEW.md)。此前 Query 07 字段交叉引用 Bug 指控已被正式更正。
- **仅已验证 Aggregation Function** 已同步至 fork 的 `src/Gateway/Gateway Monitoring.KQLDatabase/DatabaseSchema.kql`；Streaming 修复保留，Mashup 等其它 Parser、PROD、Uploader/Policy 均未改。
- 下一项为 Mashup **额外六项数值转换只读 Precheck**，特别关注旧 Parsed `TotalBytes` 全 null。已准备完整本地 MCP AI Prompt；通过前不得部署。

## 2026-11-20 15:24–15:29（UTC+8）— Aggregation 已部署，首次回归 Inconclusive
- 用户在 POC 手工执行 `parse_QueryExecutionAggregationReport` Candidate；真实 Patch UTC `2026-11-20T07:24:24.1258875Z`。本地 MCP 15:25–15:29 验证：Body 逐字节匹配 Candidate，SHA256 `626cb510…c7f21`，Policy Enabled/Transactional；Patch 后 Raw=0、Parsed=0、Policy failures 0 observed。**因无新记录，六个 Duration 尚不能判定修复成功。**
- 已归档 [原报告](../evidence/2026-11-20/gateway-aggregation-postcheck-20261120-1525-SGT.sanitized.md) 和 [独立评审](../evidence/2026-11-20/AGGREGATION_POSTCHECK_REVIEW.md)。暂不回滚，也**不修改 Mashup / PROD**。
- 独立逐项审计 GitHub 查询 `07-aggregation-postdeploy-keycheck.kql`：6 个指标 × Matches/Mismatches **12/12 引用正确**，报告第 8 节复制粘贴 Bug 指控与当前实际文件不符，因此**未修改正确的查询表达式**。
- 之前 Streaming Follow-up 的 Raw 活跃记录早于 Aggregation Patch，不能当作本次 Patch 后证据。新增只读 [Activity Gate](../kql/poc-parser-fix-20261117/controlled-rollout/09-aggregation-activity-gate.kql) 与 [AI Follow-up Prompt](../kql/poc-parser-fix-20261117/controlled-rollout/AGGREGATION_FOLLOWUP_AI_PROMPT.md)。
- GitHub 正式 `DatabaseSchema.kql` **仅 Streaming 已同步**；Aggregation 因尚未完成新数据验收，仍不修改正式 Schema。

## 2026-11-20 14:22–14:48（UTC+8）— Streaming Parser 部署后新数据回归 Passed
- 本地 `eventhouse-remote` MCP 只读 Follow-up：现场 Function Body **逐字节匹配 Candidate**（规范化 SHA256 `587bba1a…703ac`）；Policy Enabled/Transactional，未观察到新摄入失败。
- **Patch 后 Streaming Raw 5 个 Envelope / 32 行，Parsed 新增 32 行**；两项 Duration 均 **32/32 非空**，通过 KeyCheck 与 Raw **32/32 逐值一致，0 mismatch**。历史 2675 Parsed 行依然 null。
- 8/8 类 Report Raw 在 Patch 后均有新 Envelope，证明当时链路有活动，**不是** Uploader 文件级无丢失/无重复验收。
- 归档 [完整脱敏 Follow-up](../evidence/2026-11-20/gateway-streaming-followup-20261120-1445-SGT.sanitized.md) 与 [独立评审](../evidence/2026-11-20/STREAMING_FOLLOWUP_REVIEW.md)；功能性 Streaming 修复通过，但长时间运行与历史回填仍待独立处理。
- 将**仅已验证 Streaming Function** 同步到 fork 的 `src/Gateway/Gateway Monitoring.KQLDatabase/DatabaseSchema.kql`；保留原版 2026-11-17 基线，不重装 Jumpstart，不动其它 Parser / PROD。
- 下一轮针对 Aggregation 进行单函数受控 POC 修复；Mashup 先额外检查 TotalBytes 等潜在缺陷。

## 2026-11-20 14:19–14:22（UTC+8）— Streaming Candidate 已手工应用，Postcheck 暂不能判定

- 用户在确认的 POC Fabric UI 手工执行 `parse_QueryStreamingReport` Candidate，回归 UTC 起点 **2026-11-20T06:19:18.0313089Z**。
- 本地 `eventhouse-remote` MCP 的 Postcheck 执行成功：Live Function 存在、Body hash 与旧 Preflight 不同、Update Policy 仍 Enabled/Transactional、相关 ingestion failures **0 observed**；但 Patch 后 Raw **0**、Parsed **0**、KeyCheck **0**，因此严格结论为 **Inconclusive**。
- 最新旧 Raw `logDate` 06:11:46Z、Parsed 最新摄入 06:11:47Z，都在 Patch 之前。Streaming 本就低频；现在没有故障证据，但**没有修复成功的证据**。
- 已归档 [脱敏原报告](../evidence/2026-11-20/gateway-streaming-postcheck-20261120-1422-SGT.sanitized.md)、[独立复核](../evidence/2026-11-20/STREAMING_POSTCHECK_REVIEW.md)，新增两份只读检查：Raw 所有类型活跃度 / Parsed 全表 Duration 非空率，以及 [本地 AI 提示词](../kql/poc-parser-fix-20261117/controlled-rollout/FOLLOWUP_AI_PROMPT.md)。**尚未回滚、未动 Aggregation / Mashup / PROD**。

## 2026-11-20 10:36–10:47（UTC+8）— Eventhouse MCP Preflight 通过，进入 POC Patch 人工评审
- 本地 AI 通过 `eventhouse-remote.executeQuery` 只读执行 4/4 Preflight；三个目标 Function Body 与 2026-11-17 基线 **3/3 SHA256 精确一致**；3/3 Update Policies 均 Enabled/Transactional/Source Raw；Raw+Parsed 6/6 有数据；最近 2 小时 `0 observed` ingestion failures。**没有 Eventhouse 写操作**。
- 报告额外发现：Aggregation 6 列/Streaming 2 列新数据仍全 null；Mashup `TotalBytes` 2797/2797 null，Raw 存在该字段；Streaming 有 3 个 Raw-only 字段在目标 Parsed schema 中缺失。
- 报告对 `parse_QueryStartReport_csv` Folder 有内部矛盾，登记 GW-008，不影响本轮三个目标 Parser 的 Body 比对。
- 已归档 [脱敏报告](../evidence/2026-11-20/README.md) 和 [独立评审](../evidence/2026-11-20/PRE_DEPLOYMENT_REVIEW.md)，准备 [单个 Streaming Function 的受控 Runbook](../kql/poc-parser-fix-20261117/controlled-rollout/README.md) 与新数据/故障只读检查；**尚未执行任何 DDL**。
- 下一阶段仅建议单函数 POC 手动变更（用户确认后），并为 Mashup 增加 `TotalBytes` 等指标的补充只读验证。PROD 暂不启动。

## 2026-11-17 17:11（UTC+8）— DataSource 只读等价验证通过
- 用户执行了 `2-datasource-compatibility.kql`，导出 `export (12).csv`，已归档为 `DataSourceCheck-LegacyVsCandidate.csv`。Aggregation 96/96、Streaming 139/139，两类共 235 条 Raw 的 Legacy / Candidate DataSource 输出完全一致（`DifferentSerialized=0`）；两者动态类型集合均为 `array` 和 `string`。
- Mashup 103 条 Raw 的 `DataSource` 为 string，实际 JSON 中 `DataSourceKinds`、`DataSourcePaths` Key 均 0/103，纳入 GW-005 设计风险；**不在本轮最小 Parser 修复中强行生成字段**。
- 新增 `kql/poc-parser-fix-20261117/mcp-handoff/`：为本地 Eventhouse Remote MCP 准备 4 个只读 Preflight KQL 和中文 AI 交接指令。**尚未用该本地 MCP 实测**，无 POC Function DDL、无 PROD 变更。

## 2026-11-17 16:54（UTC+8）— 数值/单位只读验证通过
- 用户执行 `value-checks/1-parser-value-validation.kql`，导出 `export (11).csv`，已在 evidence 归档。共 **15 项：13 个数值转换指标 + 2 个 DataSource 非空率检查**，涉及 **338 条 Raw 记录**（Mashup 103 / Aggregation 96 / Streaming 139）。
- 13 个数值项共 **1,276 次非空值转换**：`ConversionLoss=0`、`ChangedByCast=0`（阈值 0.0001），Mashup 的 `TotalProcessorTime(ms)` 经 timespan 毫秒回算未发现超阈值差异。DataSource 的 235 次非空检查未发现空值损失，但**未证明内容等价**。
- 新增 DataSource 原版 bag_unpack vs Candidate 只读比较与 Mashup Raw Key 存在性检查，**尚未执行**；没有修改 POC Function、Update Policy 或 PROD。

## 2026-11-17 16:37～16:38（UTC+8）— Parser Schema 检查通过
- Aggregation 13/13、Streaming 9/9、Mashup 17/17，**39 个 IsMatch 均为 true**；已归档 CSV (8)/(9)/(10)。
- 新增只读 value-checks 查询，尚未执行；没有修改实际 Eventhouse Function。

## 2026-11-17 — 项目结构精简与中文统一
- 按单人项目精简组织：根目录仅保留总览 / 状态 / AI 交接，`tracking/` 集中 Roadmap、Issues、Decisions、Changelog；不设 Owner / Assignee。
- 核对已归档的 Eventhouse Overview **JSON + Markdown 脱敏快照**，统一项目文档为中文；保持上游代码及实际导出原文。
- KQL 三组候选仅用于只读验证；**未修改 Eventhouse、Uploader 或 PROD**。

## 2026-11-17 — Parser 数据质量验证
- 完整备份 15 个实际部署 Function Body、142 行 Schema、Update Policy 快照、MCP Overview；15/15 Function Body 与 GitHub 源码一致。
- QueryExecutionAggregation Dry-run：77 条，Query Duration 三列各 77 非空，Data Processing Duration 三列各 56 非空。
- QueryStreaming Dry-run：113 条，两个时长指标各 104 非空。
- MashupEvaluation Dry-run：91 条，五个 Memory / Processor 指标均 91 非空。
- 已生成 Schema Check KQL、Candidate 与 Rollback；**全部尚未在 POC 执行 Function DDL**。

## 2026-11-16 — Eventhouse Raw 样本
- Raw 快照包含 9 种 `logType`、647 条 JSON Message、3693 条 CSV Record；字段统计有 106 个 Type/Field 组合。不是全量数据完整性证明。

## 2026-11-13 — 实际 Gateway 日志上传
- 8 类 Report 经 `Move → Temp → Eventstream HTTP 201 → History`；Gateway 之后重新创建同名文件；定时任务切换至私有 `Config.live-poc.json` 运行。
- PROD 首次回传计划使用单独 Staging、按真实事件时间筛选；未验证高可靠性。

## 2026-11-12 及之前 — 基本功能
- Eventstream、Eventhouse、Semantic Model、Gateway Report 已安装并进行基础验证；详情在 [POC 实测记录](../docs/POC_STATUS.md)。

**维护原则：** 每次有重要测试/决策，附日期与证据链接，再更新 `STATUS`，不复制一整篇历史结论。
