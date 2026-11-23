# Gateway Monitoring｜当前状态

> **更新时间：** 2026-11-23（Asia/Singapore）｜**阶段：** P2 Collector Reliability（P1 必需 Parser 功能验收已通过）｜**POC：** 单节点｜**PROD：** 未批准

| 工作项 | 当前结论 | 关键证据 / 限制 |
| --- | --- | --- |
| Gateway → Fabric → Report | **功能 POC 通过** | 曾观察到八类 Report Raw 连续出现，但未验证文件级不丢失、不重复 |
| QueryStreamingReport | **已部署，POC 功能 Passed** | 32/32 新 Parsed 两个 Duration 非空且与 Raw 数值一致，0 mismatch |
| QueryExecutionAggregationReport | **已部署，POC 功能 Passed** | Raw/Parsed 新样本各 7 行，组合 Key 2/2 对照、六项有值指标 0 mismatch；非 7/7 逐行对账 |
| **MashupEvaluationReport** | **POC 功能 Passed（2026-11-23 11:06–11:08 SGT）** | 新 Raw 2 Envelope / 4 行、Parsed 4 行，五项 Memory/Time 都是 4/4 非空，KeyCheck 4/4、0 签名差异、0 unmatched；Policy failures 0 observed。样本小，非文件级无丢失保证 |
| Mashup 数组形状/类型 | **2026-11-23 只读诊断完成** | 新 Raw 36 行；TotalBytes 44/44、TotalRows 35/35 个数组元素均可转 long；两数组共同存在 6/6 行长度相等；MaxIO 12/14 有小数 |
| Mashup 三个非必需字段 | **范围已决定：Deferred（GW-009）** | 元素语义/聚合规则未验证、MaxIO 的 long 与小数冲突；未来有实际需求时才重新评估，不影响其余必需字段验收 |
| **当前版本化 Report / Semantic Model 消费** | **未见对三个问题字段的直接引用** | 检查 GitHub Fabric Report JSON、可读 Semantic Model TMDL、KQL Dashboard JSON；**不证明已部署资产/二进制 PBIT/其他下游完全无依赖** |
| GitHub 正式 KQL Schema | **Streaming + Aggregation + Mashup 三个修复均已同步** | 仅已验证的三段 Parser Function 同步至 fork 正式 DatabaseSchema.kql；Mashup 对应 commit f4304485。Parsed Schema、Policy、Uploader、PROD 无本次变更 |
| 历史回填与 PROD 可靠性 | **未验收** | 历史 null、Uploader Manifest、重试幂等、Retention、Multi-Node、长期稳定性仍开放 |

## 2026-11-23 最新证据

- [Mashup 本地 MCP Shape Follow-up 原始脱敏报告](evidence/2026-11-23/gateway-mashup-shape-followup-20261123-0949-SGT.sanitized.md) · [独立技术评审](evidence/2026-11-23/MASHUP_SHAPE_FOLLOWUP_REVIEW.md)。3 份统计查询成功（查询 4/6 经 `floor(x,1.0)` 最小修正）；当前 GitHub 只读脚本已同步修正。
- 36 行中 TotalBytes 23 行（17 单、6 多）、TotalRows 12 行（4 单、8 多）；同现 6 行 6/6 等长。**结构不等于业务定义**；元素整数转换成功仅说明拆分在技术上可行。
- MaxIO 14 个值中 12 个包含小数；Kusto `real` 可承载浮点数，但**不是保证任意十进制值精确无损**。
- 本地 AI 的 Github Code Search 返回零不能证明无相关项目源码；检查仓库下的 Fabric Report/SM/Dashboard 可读文件未发现问题字段的**直接**消费。

## 2026-11-23 Mashup 最小 Precheck 新证据

- 用户提交本地 MCP 只读报告：五个目标 Memory/Time 指标（4 × 33/33、1 × 36/36）ConversionLoss 0、ChangedByCast 0，目标 Schema 17/17；当前 Live Function 仍旧版，Update Policy 仍 Transactional。参见 [独立评审](evidence/2026-11-23/MASHUP_MINIMAL_PRECHECK_REVIEW.md)。
- v2 Candidate 的函数体**与旧 Blocked Candidate 完全相同**，区别在三个非必需指标明确延期（GW-009）；该 Precheck 阶段尚未验证另外 12 列兼容性，已在下述单独 Gate 中补足。

## 2026-11-23 Mashup 12 列兼容性 Gate 结果

- 用户报告在 POC 原样只读执行 [compatibility.kql](kql/poc-parser-fix-20261117/value-checks/7-mashup-legacy-candidate-compatibility.kql)：**Legacy 36 / Candidate 36、DifferingSignatureGroups 0、UnmatchedRows 0、PASS**；五项目标字段的独立 Dry-run 也已通过。见 [验收证据](evidence/2026-11-23/MASHUP_LEGACY_CANDIDATE_COMPATIBILITY_PASS.md)。
- 经过独立评审，当前 POC **部署前功能 Gate 已通过**，已准备专门的 [P1 POC Candidate](kql/poc-parser-fix-20261117/candidates/4-parse_MashupEvaluationReport_P1_minimal_POC.kql) 与 [受控部署/回归 Runbook](kql/poc-parser-fix-20261117/controlled-rollout/MASHUP_RUNBOOK.md)；现有现场旧版 Rollback 文件已确认存在。
- **注意：这不是已部署，也不是 Mashup Postcheck Passed。** 事务性 Update Policy 风险、历史 Parsed null、GW-009 延期及 PROD 可靠性风险仍存在。

## 2026-11-23 Mashup POC 手工部署确认

- 用户报告已经在 Gateway Monitoring POC 手工执行 Candidate，提供实际 **RecordedAfterPatchUTC = `2026-11-23T02:39:15.4152749Z`**，附件 Function 导出显示 `parse_MashupEvaluationReport` 已使用 `mv-expand record = log` 及五项 Memory/Time 显式映射。见 [部署摘要证据](evidence/2026-11-23/MASHUP_MANUAL_DEPLOYMENT_20261123.md)。
- **只证明用户报告的 POC Function 已更新，Postcheck 尚未完成**；未独立验证部署后 Policy 状态、ingestion failures 与新 Raw / Parsed 值匹配。三项 GW-009 Deferred 不变。

## 2026-11-23 10:47–10:50 SGT Mashup 首轮 Postcheck 结论

- 用户提供本地 MCP 报告：Live Function 与 P1 Candidate 逐字节/规范化 SHA256 一致，Update Policy 仍 Enabled、Transactional、Source Raw、Query `parse_MashupEvaluationReport()`。
- PatchUTC `2026-11-23T02:39:15.4152749Z` 之后 **Raw 0 / Parsed 0，Keycheck `INCONCLUSIVE_NO_NEW_RAW`**；观察到相关 Policy Failure 0，但这不是成功摄入/无上传故障的证明。最新 Mashup Raw Envelope 发生在部署前约 13.2h。见 [独立评审](evidence/2026-11-23/MASHUP_POSTCHECK_1045_REVIEW.md)。
- 本轮仅本地 AI **只读查询**；不回滚、不再次运行 Candidate、不将 Mashup 标记 POC Passed、不更新正式 `DatabaseSchema.kql`。

## 2026-11-23 11:06–11:08 SGT Mashup Postcheck 功能验收 PASS

- 用户本地 Eventhouse MCP 只读报告：13 Gate `READY_FOR_POSTCHECK`、Mashup 新 Raw **2 Envelope / 4 Records**、其他 Report 18 Envelope；10 Summary Raw/Parsed 各 **4 行**，五项必需 Memory/Time 指标两层均 **4/4 非空**；11 KeyCheck **RawRows=4、ParsedRows=4、DifferingGroups=0、UnmatchedRows=0、PASS**；12 Policy failures **0 observed**。
- 前一轮现场 Function Body 与 Candidate 严格匹配、Policy Enabled/Transactional。**当前样本的 Mashup POC 功能通过**；和早先 Streaming、Aggregation 的功能 PASS 合计，P1 已定义三项 Parser 验收范围完成。见 [独立评审](evidence/2026-11-23/MASHUP_POSTCHECK_PASS_1105_REVIEW.md)。
- 已将 Mashup 已验证 Function 同步 fork 正式 `src/Gateway/Gateway Monitoring.KQLDatabase/DatabaseSchema.kql`（commit `f4304485`）；没有对现场 Eventhouse 再执行任何 DDL。历史 Parsed null 不回填，GW-009 三个指标仍 Deferred。
- **注意：** 4 条 Mashup 样本不能证明长期采集、源 CSV 至 Raw 的无遗漏或 Exactly-once；GW-002、GW-003、GW-004 等依然开放。

## 2026-11-23 P2 Uploader 可靠性源码审计

- 已由 ChatGPT **直接读取并分析 GitHub 源码**（不是交给本地 AI 判定）：`UploadGatewayLogs.psm1`、`Utils.psm1`、`Run-UploadGatewayLogs.ps1`。
- **原始静态风险分析（已由后续 P2-T1 修正）：** Report 扫描包括 Temp、已在 Temp 的文件对自身 `Move-Item`；但 2026-11-23 P2-T1 实测同路径 Move **成功**，故“同路径必失败”假设已排除。其它真正 Move 失败时 `fileReady=false` 仍无法阻止后续 Merge；Batch 无持久化 Checkpoint、HTTP 重试重复与归档确认是当前更重要的可靠性风险。
- 完整 [源码独立评审](evidence/2026-11-23/UPLOADER_SOURCE_RELIABILITY_REVIEW.md)，这是 **代码路径分析，不是已观察的故障**。

## 2026-11-23 POC Gateway VM 快照（用户只读执行）

- VM `RNGVMC0046` / PowerShell 7.6.10；Task `Gateway-UploadLogs` Running，LastTaskResult `267009`（0x41301，表示任务运行）。
- `Report\History` 顶层 88 文件 / 24.69 MB，8 种 Report 各 11；`Report` 顶层与 `Report\Temp` 顶层各 0；指定 `GatewayMonitoring.log` 路径未找到。**这些检查没有递归，也未核对当前 Task Action，不能直接得出 Temp 清空、无错误或上传无丢失结论**。见 [只读快照记录](evidence/2026-11-23/GW002_VM_READONLY_SNAPSHOT_20261123.md)。

## 2026-11-23 VM 递归补充检查已完成

- `Report` 全树递归枚举 `*Report_*.log`，只发现 **History 88 份**（8 类各 11），**没有匹配的 Source/Temp 文件**；这一点修正先前“顶层零条、子目录未知”的局限。按类型分类见 [递归快照证据](evidence/2026-11-23/GW002_VM_RECURSIVE_FOLLOWUP_20261123.md)。
- Task Action 通过脱敏条件检查：`Run-UploadGatewayLogs.ps1` / `Config.live-poc.json` 均匹配，Working Directory 已配置。当前无可观察的 Temp 故障案例；**不意味着风险已解除**。
- 不再重复无新信息的目录状态检查；下一步仅在隔离临时文件夹复现同路径 Move 的 PowerShell 行为。

## 2026-11-23 P2-T1 单项行为实测通过（修正源码风险判断）

- 用户在 POC VM PowerShell **7.6.10** 的独立 `$env:TEMP` 测试文件上，将 `Move-Item` 的 Source/Destination 均指定为同一完整路径；**Attempts=1，FileReady=True，FileStillExists=True，Error 为空**。
- 说明**同路径 Move 本身在当前环境不会触发失败**，我此前将其列为最优先可触发故障的假设不成立，应下调该具体风险优先级。源码中 Move 因其他原因失败后 `fileReady=false` 仍无法阻止 Merge 的分支仍存在，但不能凭本次结果证明有数据丢失。见 [P2-T1 实测](evidence/2026-11-23/GW002_T1_SAME_PATH_MOVE_RESULT.md)。
- **Collector Reliability 未验收**；下一步聚焦 HTTP Batch 失败后无持久化进度/幂等机制的风险，先核对 POC 实际 Retry 配置，不反复做同路径 Move 测试。

## 2026-11-23 POC Retry 配置已核实

- 用户在 VM 上只读确认 `Config.live-poc.json`：`ConfigFound=True`、`RetryConfigExists=True`、`MaximumRetryCount=3`、`RetryIntervalSec=1`；对应 `Utils.psm1:362–369` 的 HTTP 请求参数。原先怀疑配置缺失的风险**在本 POC 已排除**。见 [Retry 配置评审](evidence/2026-11-23/GW002_HTTP_RETRY_CONFIG_REVIEW_20261123.md)。
- PowerShell 自动重试是 HTTP 400–599/304 响应级重试（429 的 Retry-After 可能覆盖配置），**不是 Batch 进度 checkpoint 或端到端幂等机制**。核心未验证风险是已成功 Batch 在后续失败后整文件重传导致重复；不需再检查或调整 3/1 配置。

## 2026-11-23 P2-T2 部分 Batch 失败：重复发送已在 Mock 中复现

- 用户在 POC VM 通过**本地 `Utils.psm1` 的实际 `Add-LogToEventHub` 分批函数 + 内存 Mock `Add-MsgEventHub`** 完全离线测试：合成 CSV 2,001 行，首次 Run 成功“接受”第一批 2,000 行、第二批模拟失败；下一次运行从头重发；Mock 接受 Run1 2,000、Run2 2,000 + 1 行，`DuplicateBatchObserved=True`，第一批重复接受两次。见 [独立验收](evidence/2026-11-23/GW002_T2_OFFLINE_PARTIAL_BATCH_FAILURE_PASS.md)。
- **技术结论：应用层重复发送风险已获受控复现；未实测 Eventstream/Eventhouse 的真实重复摄入，更不能称端到端可靠性通过。** 原始函数没有持久化 Batch Checkpoint 或稳定去重 ID；HTTP Retry=3/1 无法填补这一缺口。未触碰实际网关日志、Task、配置、Eventhouse 或 PROD。
- T1 同路径 Move 成功、T2 部分失败重复发送已足够证明当前优先级，停止反复运行相同模拟；下一步重点转向最小幂等/对账设计与后续可验证门槛。

## 2026-11-23 Report 消费审计与 V2 改版启动

- 直接读取 GitHub `src/Gateway/Gateway Monitoring.Report/report.json` 和对应 Semantic Model TMDL：**6 页、58 Visual**；原版模型四个 KQL 功能来源为 `GatewayNodeStatusWithInfo`、`SystemCounters`、`QueryExecutionUnified`、`QueryConnections`。
- **重要依赖关系：** 当前版本化模型**不直接引用** P1 新修复的 Streaming / QueryExecutionAggregation / Mashup 三表；这三项修复不会自动在原版报表新增指标。模型中的 `Query Avg. Execution ... (ms)` DAX 实际计算 P90 秒，有标签/单位不一致，需要 V2 修正。**尚未核实实际部署模型/报表与 GitHub 完全一致**；PBIT 是另一个二进制资产。
- 用户希望重新设计一版清晰易用的 Gateway Monitoring Report；新建 [V2 字段消费/页面蓝图](docs/GATEWAY_REPORT_V2_BLUEPRINT_20261123.md)，以 5 页独立报告为候选，不覆盖现有 POC Report，模型先复用、后按必要性扩展。新增 GW-011 独立跟踪。
- **本轮仅 GitHub 源码分析与蓝图，无 Live Semantic Model 连接、PBIP 文件生成、部署或 PROD 修改。**

## 2026-11-23 用户确认生产化优先级调整

- **P1 必需 Parser 修复已验收，不等于所有字段修复**：8 类 Gateway Performance Report 已在 `GatewayReports-Raw` 观察到；标准 Parsed 未保留全部 Raw-only 字段，GW-005 / GW-009 继续跟踪；`CopyExecutionReport` 缺少实际源样本，不能宣称已通过。Gateway Basic/Verbose/Additional Logs 不包含在当前 Performance Report Eventstream 上传范围，POC Lakehouse UploadLogs=false。
- **GW-010 单独 Deferred：** P2-T2 离线 Mock 复现部分 Batch 成功后再处理重复发送，但用户明确此类少见异常不是有限监控 Pilot 的最高优先级；保留风险证据，不需立即做 Batch Checkpoint/完全幂等改造。完整 Manifest GW-004 留长期增强，不再作为有限监控 Pilot 的默认硬前提。
- **更实际的 Pilot 前剩余工作**：确认实际报表必需字段及页面、代表性连续窗口数据新鲜度和无不可解释静默遗漏、Gateway 资源/业务影响、PROD 多节点与初次 Backfill 范围、QueryText 安全/RBAC、Raw 保留与回放、告警/回滚。见 [Readiness Gap](docs/PROD_READINESS_GAP_20261123.md) 和 [ADR-010](tracking/DECISIONS.md)。
- **边界：** 这是一项优先级和验收范围调整，并没有批准/开始 PROD Pilot，也没有修改 POC/PROD Uploader。

## 当前唯一下一步

**开展有限 PROD Pilot Readiness 最小核对（先只读）**：确定需要展示的 Report 页面/指标，并以当前工作分支 Schema 和已验证的 8 类 Raw 覆盖为基线，列出真正缺少的必需字段及上线最小差距；并行安排连续运行/节点部署、安全 Retention/Backfill/告警的可验证检查。**不再优先做异常重发/完整 Manifest 设计，也不直接变更 PROD。**

- **范围决定：** `TotalBytes`、`TotalRows`、`MaxIODataBytesPerSecond` 暂不修复、暂不额外重复统计、不联系微软；[ADR-009](tracking/DECISIONS.md) 已确认，[GW-009](tracking/ISSUES.md) 独立追踪。已知 null 不代表值为零或已修复。
- **之后优先：** 按有限 PROD Pilot Readiness 优先验证重要数据的完整性/新鲜度、业务影响、安全、Retention、节点/Backfill、告警和停止条件；GW-010 重复发送与 GW-004 完整 Manifest 延期。PROD 仍未批准。
- **现场边界：** 2026-11-23 用户仅在 **POC** 手工更新过 Mashup Live Function；随后 Postcheck 全程只读。本轮将已验证逻辑同步至 GitHub 正式 KQL 源码；Parsed Schema、Update Policy、Uploader、PROD 均无本次修改。

[问题台账](tracking/ISSUES.md) · [进展记录](tracking/CHANGELOG.md) · [设计决定](docs/MASHUP_DESIGN_OPTIONS.md)。
