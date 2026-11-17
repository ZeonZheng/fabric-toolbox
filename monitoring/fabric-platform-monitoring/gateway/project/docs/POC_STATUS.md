# Gateway Monitoring｜POC 进展与验证证据

> **记录截至 2026-11-17。** 这是一次 Windows On-premises Data Gateway **单 Cluster、单 Node** POC 的实测结果；不能代表所有 Gateway 节点或 Fabric Platform Monitoring 模块。

[返回总览](../README.md) · [继续部署](DEPLOYMENT_RUNBOOK.md) · [已发现问题](TROUBLESHOOTING.md)

## 一页结论

**已完成：** 核心端到端链路；NodeInfo、Heartbeat 实测；8 类真实 POC Performance Logs 首次 Move/上传/History 归档成功，Gateway 重新生成同名日志后继续写入。  
**进行中（2026-11-13）：** `Gateway-UploadLogs` 采用 `Config.live-poc.json` 已启用并显示 Running，Heartbeat Running、NodeInfo Ready。**任务状态不等于数据完整性验收。**  
**新发现（2026-11-17）：** Eventhouse Raw 中已见 9 类 logType；Parsed 的 3 个关键数值字段全部为 null（源 Raw 有非空值），已证实部署 Parser 字段映射与 Raw 不一致。已备份源码及三个部署 Function Body，**修复未应用**。\n**尚未完成：** Parser 全字段修复/回归、连续多轮采集无遗漏/重复、失败恢复、Running Jobs / Job Details、多 Cluster/Node、PROD 首次回传及长期治理。

## 1. 验证矩阵

| 功能 | 实测结果 | 状态 |
| --- | --- | --- |
| Jumpstart Fabric Items | 两个 Eventstreams、Eventhouse/KQL、Semantic Model、Report 已创建并可用 | ✅ |
| Notebook | 已在 POC 中完成配置生成；但 fork 里又做了兼容性修正，**提交版尚未重新运行** | 🟡 |
| PowerShell 环境 | VM 上已验证 PowerShell 7.6.10（原 7.6.9），四个依赖模块可见 | ✅ |
| Gateway Performance Reports | 服务账户目录中存在 **8 类性能报告**，无需 Additional logging | ✅ |
| SPN Secret | DPAPI 解密成功，长度与再次粘贴的 Secret 完全匹配 | ✅ 本机验证 |
| SPN Gateway 权限 | 添加 Gateway Cluster Admin 后可见 Cluster **0 → 1**、匹配 Node **0 → 1** | ✅ |
| Gateway Node Metadata | Eventstream HTTP 201，KQL 中 Cluster/Node/Version 完整 | ✅ |
| Heartbeat | Eventstream HTTP 201，KQL 有 `Responding=true` 的实时记录 | ✅ |
| Heartbeat Scheduled Task | `Running`、`LastTaskResult = 267009`（正在运行） | ✅ |
| NodeInfo Scheduled Task | 执行后 `Ready`、`LastTaskResult = 0`（成功完成） | ✅ |
| Performance Logs 上传 | 静态副本 4 类曾核对 Raw/Parsed；2026-11-13 真实来源 8 类 Move 并通过 Eventstream HTTP 201 | ✅ 静态副本；✅ 真实首轮传输 |
| Gateways / System Counters | Cluster/Node、CPU/Memory 趋势可见 | ✅ |
| Jobs / Queries | 补充 QueryStart 和 QueryExecution 后在历史日期出现数据 | ✅ |
| Running Jobs / Job Details | 未展示端到端验证证据 | ◻️ 未验证 |
| 活动日志的长期采集 | 首轮移动后源目录自动重新创建同名文件并继续写入；Task 已启动持续观察 | 🟡 运行中、未完成长期验收 |
| Gateway-UploadLogs Scheduled Task | 已由 `Config.poc.json` 调整为 `Config.live-poc.json`，Task Scheduler 显示 Running | 🟡 观察中 |
| 多 Node / 多 Cluster | 未测试 | ◻️ |
| Capacity Events / Activity Events / Inventory | **未参与此次 POC** | ◻️ |

## 2. Raw → Parsed：已核对的数据量

以下是历史副本上传后的结果，**不是持续运行的统计**。

| 类型 | Eventstream 消息批次 | Raw 日志记录数 | KQL 解析后记录数 |
| --- | ---: | ---: | ---: |
| QueryExecutionAggregationReport | 2 | 1,306 | 1,306 |
| SystemCounterAggregationReport | 27 | 53,538 | 53,538 |
| QueryExecutionReport | 3 | 2,055 | 2,055 |
| QueryStartReport | 7 | 2,055 | 2,055 |

首次两类聚合日志共发送 **29 批 / 54,844 条**，全部对应到解析表。随后补充查询明细，使 Jobs / Queries 页面可用。

**不要混淆：** HTTP `201 Created` 仅证明接收端接受消息；Raw 和解析后的计数一致，是另一层独立验证。它仍不能证明未来持续采集会 Exactly-once。

### 复查 KQL（可直接复制）

```kusto
['GatewayReports-Raw']
| where logType in ("QueryStartReport", "QueryExecutionReport",
                    "QueryExecutionAggregationReport", "SystemCounterAggregationReport")
| summarize Messages=count(), Rows=sum(array_length(log)) by logType
```

```kusto
GatewaysHeartbeat
| summarize LastSeen=max(ingestion_time()),
            EventsLast5Min=countif(ingestion_time()>ago(5m))
    by GatewayId
| extend AgeSeconds=datetime_diff("second", now(), LastSeen)
```

## 3. 报表验收详情

| 页面 | 实测观察 |
| --- | --- |
| Gateways | 显示 POC Gateway Cluster、Node、Gateway 版本、逻辑核心数、内存、OS |
| System Counters | System CPU、Gateway CPU、System Memory、Gateway Memory 的历史趋势正常 |
| Jobs | 将日期选到 **2026-10-19**，显示 **3 Jobs、1 Node、5 Queries、P90 12.44 秒** |
| Queries | 看到 **5 Queries、3 Jobs** 和数据源分类（如 ODBC / SQL / Web） |
| Running Jobs / Job Details | 未提供测试结果，不能标记完成 |

**重要经验：** 日期过滤使用**日志事件时间**，而不是上传到 Eventhouse 的日期。此次如果只看 2026-11-11，会误以为 Jobs / Queries 没有数据。Model 参数配对后还必须执行 **Edit credentials**。

## 4. 本机文件持续写入：直接实测

在 2026-11-12 多次检查，文件名保持 `20261019T110120.log`，但文件大小不断增长。

| Report | 第一次（bytes） | 最后一次（bytes） | 差值 |
| --- | ---: | ---: | ---: |
| SystemCounterAggregation | 6,668,473 | 6,670,766 | **+2,293** |
| QueryExecutionAggregation | 561,365 | 563,900 | **+2,535** |
| ConcurrentOperationAggregation | 374,348 | 374,568 | **+220** |
| QueryStart | 4,482,959 | 4,482,959 | 无变化 |

这证明至少有部分报表正在**以同名文件继续追加**。并没有证明被外部移动后会如何重建，因此不能安全假设 Uploader 可以直接 `Move-Item` 活动文件。

## 5. 2026-11-13 真实日志 Move 与自动任务验证（新增）

本节是**真实活动日志**实验，区别于上表历史副本的 Raw/Parsed 计数；当前没有重新核算真实批次的 Eventhouse 总行数，不得把 HTTP 201 等同于 Exactly-once。

- **测试前：** `Gateway-UploadLogs` Disabled；`\FabricGatewayMonitoring\Gateway-UploadLogs` Action 指向 `configs/Config.poc.json`；备份了真实 `Report` 目录中的 8 类性能日志及文件状态 CSV。
- **测试配置：** 保留原始上传代码，创建单独的本机私有 `Config.live-poc.json`；`GatewayLogsPath` **直接指向** `C:\Users\powrbi\AppData\Local\Microsoft\On-premises data gateway\Report`；`ReportSendInterval=60`、`ReportRetention=3650`（仅 POC 临时保留）、`EventHubs.UploadReports=true`、两种 Lakehouse 上传为 false。
- **首轮执行：** 2026-11-13 约 **06:46 UTC**，扫描到 **8 个**匹配文件；每个 `Move-Item` 到 `Report\Temp` 成功，随后拆分 CSV 消息发送 Eventstream，所有展示的 HTTP 响应为 **201 Created**；每类后续合并到 `Report\History` 的独立 `*_f.log` 文件。首轮之后再次轮询曾显示 `Gateway Report log count: 0`。
- **源端行为：** Move 后的 `Report` 根目录重新生成 8 个同名文件，至少一部分文件随后继续增长（例如 `QueryExecutionReport` 重新创建于 2026-11-13，随后仍有写入）。新生成文件名称仍包含 **20261019T110120**，说明文件名时间戳不是物理创建时间；其底层命名策略尚未证实。
- **Task 更新：** 使用正确的任务路径 `\FabricGatewayMonitoring\` 成功导出 `Gateway-UploadLogs.backup.xml`（1528 bytes）。`Set-ScheduledTask` 修改 Action 时出现 `The user name or password is incorrect`，改用 Task Scheduler GUI 修改参数至 `Config.live-poc.json` 并启用运行。截图确认：**Gateway-Heartbeat Running；Gateway-NodeInfo Ready；Gateway-UploadLogs Running**。
- **仍需复核：** 24–48h／3–7d 持续运行、每轮记录数与 Eventhouse Raw/Parsed 一致性、重复/遗漏、失败及重试恢复、资源/磁盘占用、用户刷新是否受影响。

**安全边界：** 本次真实 Move 实验仅在 POC Gateway 完成；尚未执行对 PROD Gateway 活动日志的 Move。

### 2026-11-13 源码机制确认

- 原始 `UploadGatewayLogs.psm1` 对 Performance Report 使用 `Get-ChildItem *Report_*.log -Recurse -Exclude *_f.log`，**无 LastRun/LastWriteTime 记录级筛选**；每轮移动整个匹配文件到 `Temp`，逐文件先上传再执行 `Merge-ReportFiles`。
- `Utils.psm1` 的上传将 CSV 读为记录、通常 2000 条为初始批量并根据 JSON 大小收缩，向 Eventstream EventHub-compatible endpoint 发送。`Merge-ReportFiles` 在**本地 History** 按第一个下划线前的 Report Type 匹配 `$report*_f.log`，选择当日创建且小于 **100 MiB** 的文件追加；不存在则新建。成功合并后删除 Temp；History 按 `ReportRetention` 清理。**History 不作为 Eventstream 再次上传源。**
- **两套独立规则：** Gateway 自身 `ReportFileCount=10`、`ReportFileSizeInBytes=104857600`（100 MiB）管理原始文件；Uploader 的 History 合并阈值是 `Utils.psm1` 中硬编码的 `100mb`。上传间隔也不依赖源文件写满 100 MiB。
- **风险仍存在：** 逐批上传部分成功后失败会存在重传重复的可能；递归扫描包含 `Temp` 文件，恢复路径未经过失败测试；同名源文件反复创建不代表事件没有遗漏。故当前只批准 POC 观察，不宣称生产级可靠性。

### PROD 首次采集计划（尚未实施）

PROD 的样例 VM 发现数十个 Performance Log 历史文件，部分接近 100 MiB。先完成 POC 连续运行观察，再使用**独立 Staging 目录**和明确的首次截止点，只采集所需时间窗口（初步为“昨天 00:00 至今”）。由于文件名日期可能早于实际事件时间，不能仅按文件名筛选；需要逐 Report Type 按 CSV 真实事件时间筛选或制定可审计的等效切分方案。**不要在 PROD 的 `Report` 目录内建 Archive 并假设会被跳过**（原 uploader 递归扫描），也不要直接移动或删除历史文件；先估算量、验证首次回传与后续在线采集衔接。

## 6. 保留的环境说明（仅用于解释此次 POC）

- 网关展示名：`POC Data Gateway`；Node：`Power BI Gateway POC`。
- Gateway 版本：`3000.338.2`。
- Gateway 服务以技术账号运行；实际 Report 根目录位于该账号的用户 Profile 下，而不是默认 PBIEgwService 路径。
- 曾在 Heartbeat 刚启动时看到 `AgeSeconds = 17` 和最近 5 分钟 `7` 条记录；这不是连续五分钟稳定运行的基准值。
- 为安全起见，**没有把 Tenant ID、App ID、Gateway ID、Client Secret、Eventstream Key 或原始日志上传到仓库**。

## 7. 下次续建 Backlog

- [ ] 观察原始 Uploader 持续运行 24–48 小时，再延长至 3–7 天；按每轮 History/Raw/Parsed 数量验证无遗漏/重复。
- [ ] 在 POC 对上传中断、部分批次成功后失败和 Temp 恢复流程做受控测试；若可靠性不达标，再比较企业日志平台、Fluent Bit/Vector/C# Tail Collector。
- [ ] 验证所有 8 类 CSV（含引号、多行 QueryText、UTF-8、文件截断/轮转）。
- [ ] 验证增量 Offset/Checkpoint、批量发送、重试、恢复、EventId 去重。
- [x] POC 单节点验证 Move 后 Gateway 自动重新创建文件，并启动 UploadLogs 后台 Task（2026-11-13）。
- [ ] 完成多日连续数据完整性与 Gateway 可用性验证，方可决定生产部署。
- [ ] 增加 Collector 健康、任务失败、磁盘积压、消息延迟的告警。
- [ ] 验证 Running Jobs、Job Details、多 Gateway Nodes/Clusters 和最终数据完整性。
- [ ] 复核网络、Service Account 权限、敏感 QueryText、Eventhouse Retention 与 Fabric 成本。

**当前不应称为“Production Ready”**。本页记录的是已完成的功能性 POC 与正在观察的原始 Uploader 持续采集，不能替代生产可靠性验收。


## 8. 2026-11-17 — Eventhouse Raw / Parsed 数据质量专项验证

- **Raw 入库快照（2026-11-16 ~02:59 UTC）：** 9 种 `logType`、647 个 Envelope、3,693 条 CSV Records。其中 8 种为 Gateway Report，`GatewayNodeInfo` 来源独立脚本。该快照不能证明全量上传、无重复。
- **Raw 字段清单：** 106 个类型/字段组合，详情见 [字段备份](../evidence/2026-11-17/RawFieldInventory.export.csv)；同类样本字段存在，但某些值正常为空。
- **Parsed（2026-11-17 ~06:24 UTC）：** `QueryExecutionReport.QueryExecutionDuration_ms_` 4,595/4,595 非 null；`QueryExecutionAggregationReport.AverageQueryExecutionDuration_ms_` 0/2,935；`QueryStreamingReport.StreamingDuration_ms_` 0/2,346；`MashupEvaluationReport.[AverageCommit byte]` 0/1,892。
- **Raw 的对应字段仍有值（~06:31 UTC）：** `AverageQueryExecutionDuration(ms)` 73/73、`StreamingDuration(ms)` 81/90、`AverageCommit(byte)` 77/77。不同层级留存/采集时间窗口不可混为一谈。
- **KQL 配置：** 导出的 14 条表 Update Policy 中，10 个 Parsed Target Policy 开启，来自 `GatewayReports-Raw`；24 小时 `OriginatesFromUpdatePolicy` ingestion-failure 查询为空。这不代表无静默字段映射错误。
- **实际部署函数：** 3 个 `.show functions` Function Body 与分支中的 `DatabaseSchema.kql` 对应源码一致。**仅这 3 个函数已保存实际部署备份；并非完整生产 Schema 导出。**
- **本地 Report 文件分布：** 42 个（40 `History` merge 文件，2 `Source`），覆盖 8 种 Report。`History` 不是逐文件上传审计证据。
- **安全/变更：** 2026-11-17 未修改运行中的 Parser、Table 或 Update Policy；仅 GitHub 文档和证据备份。`main` 未修改。未经全面验收不升级为 PROD。

详细问题、复现查询、后续修复路线：**[PARSER_SCHEMA_AUDIT.md](PARSER_SCHEMA_AUDIT.md)**；部署源码和实际导出：**[Baselines](../evidence/2026-11-17/README.md)**；PROD 追踪设计需求：**[Upload Manifest](PROD_UPLOAD_MANIFEST_DESIGN.md)**。


## 9. 2026-11-17 — 完整部署 KQL Baseline + 修复候选（尚未部署）

- 用户提供完整 `.show functions` CSV，**15/15 部署 Function Body** 与 fork `DatabaseSchema.kql` 源码（忽略空白）一致。并提供 `.show database schema` **142 行**；MCP Overview 记录 **12 Tables / 10 transactional Update Policies / 2 healthy MVs / 156,878 rows** 的瞬时库存。
- 备份已写入 [2026-11-17 baselines](../evidence/2026-11-17/README.md)，删除数据库/群集/Policy Owner ID 等环境标识，保留 15 个原始 Function Body 和 Schema 列/类型/顺序。不是整库物理备份或生产配置导出。
- 完成 [8 个 Report 类型 + GatewayNodeInfo 原始字段与 Parsed 表列对照](SCHEMA_FIELD_MAPPING_AUDIT.md)；确认 QEA 6 个 Duration 列、QueryStreaming 2 个 Duration 列、Mashup 5 个待映射列。不同于这些缺陷，OpenConnection 和 QueryExecution 的 `(ms)` 字段目前由旧 parser 正确读取，再按目标顺序输出，不应整体盲改。
- 创建 **3 套候选 Patch / 3 套只读 Dry-run / 3 套 Rollback**：[kql/poc-parser-fix-20261117](../kql/poc-parser-fix-20261117/README.md)。目前只是 GitHub 文件，**未在 Eventhouse 执行任何 DDL / 修复 / 回填**，也尚未通过实际 KQL Dry-run。
- 本轮新关注点：Raw 观察到的 QueryExecution 2 个独有 Duration 字段、QueryStreaming 3 个独有字段、Mashup 的原始 DataSource/RequestId 不保存在标准 Parsed Tables；PROD 是否需要保留全部 Raw 字段要独立决策。

**下一行动：** 在 POC Queryset 分别运行三个 dry-run，检查非空率、列名/列序/类型和 DataSource/单位样本。先核验 POC、再审查变更、最后部署。保留 GitHub 为 Source of Truth。


## 10. 2026-11-17 只读 Parser Candidate 测试通过（不是最终验收）

- 三个 KQL dry-run 在 POC Queryset 执行成功：QueryExecutionAggregation 77 rows（Query Duration 3列各 77 非空，Processing Duration 3列各56非空）；QueryStreaming 113 rows（2列各104非空）；MashupEvaluation 91 rows（5列各91非空）。
- 未部署任何 `.create-or-alter function`，原有表、Update Policies 未更改。下一步核对候选列顺序/类型和样本值/单位，特别是 Mashup `TotalProcessorTime(ms)`。
- 正式证据摘要：[DRYRUN_RESULTS](../evidence/2026-11-17/DRYRUN_RESULTS.md)。当前权威状态：[STATUS](../STATUS.md)。
