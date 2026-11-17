# 本地 AI 执行指令｜Gateway Monitoring POC Eventhouse Preflight

你已经连接了本机 **eventhouse-remote MCP**。请进行一次 **POC Gateway Monitoring KQL Database 上线前只读检查**，执行完成后向我交付**可供另一个 ChatGPT 实例独立复核的完整脱敏证据报告**。

## 1. 正确的仓库与版本

- 仓库：`https://github.com/ZeonZheng/fabric-toolbox`
- **必须使用工作分支：** `docs/gateway-monitoring-poc-20261112`，不是 `main`。
- **完整查询目录（先打开）：** https://github.com/ZeonZheng/fabric-toolbox/tree/docs/gateway-monitoring-poc-20261112/monitoring/fabric-platform-monitoring/gateway/project/kql/poc-parser-fix-20261117/mcp-handoff
- 任务说明：https://github.com/ZeonZheng/fabric-toolbox/blob/docs/gateway-monitoring-poc-20261112/monitoring/fabric-platform-monitoring/gateway/project/kql/poc-parser-fix-20261117/mcp-handoff/README.md
- 部署前实际 Function Body 基线（共 15 个）：https://github.com/ZeonZheng/fabric-toolbox/blob/docs/gateway-monitoring-poc-20261112/monitoring/fabric-platform-monitoring/gateway/project/evidence/2026-11-17/DeployedFunctions.full.export.csv

从上述**同一分支**读取以下四份 KQL，按编号分别执行：

1. `01-live-parser-definitions.kql` — https://github.com/ZeonZheng/fabric-toolbox/blob/docs/gateway-monitoring-poc-20261112/monitoring/fabric-platform-monitoring/gateway/project/kql/poc-parser-fix-20261117/mcp-handoff/01-live-parser-definitions.kql
2. `02-update-policies.kql` — https://github.com/ZeonZheng/fabric-toolbox/blob/docs/gateway-monitoring-poc-20261112/monitoring/fabric-platform-monitoring/gateway/project/kql/poc-parser-fix-20261117/mcp-handoff/02-update-policies.kql
3. `03-recent-data-snapshot.kql` — https://github.com/ZeonZheng/fabric-toolbox/blob/docs/gateway-monitoring-poc-20261112/monitoring/fabric-platform-monitoring/gateway/project/kql/poc-parser-fix-20261117/mcp-handoff/03-recent-data-snapshot.kql
4. `04-update-policy-failures.kql` — https://github.com/ZeonZheng/fabric-toolbox/blob/docs/gateway-monitoring-poc-20261112/monitoring/fabric-platform-monitoring/gateway/project/kql/poc-parser-fix-20261117/mcp-handoff/04-update-policy-failures.kql

如果访问 GitHub 失败，可从**已经 checkout 到上述工作分支**的本地克隆读取相同路径，但先核对 `git branch --show-current` 与实际文件内容；不允许静默改用默认分支。如果基线无法读取，明确报告 `Inconclusive`，不能声称 Function 与基线一致。

## 2. 工具和操作限制

- 先识别 eventhouse-remote MCP **真实暴露的工具名及输入 Schema**；只使用其支持的只读 KQL 查询。不要猜测工具参数。
- 确认目标为 **POC** 数据库，不能在 PROD 执行。
- 只读：不得执行 `.create`、`.alter`、`.set`、`.drop`、`.delete`、`.ingest`、Jumpstart 更新、Uploader/Task 修改，也不得自动运行 `candidates/` 或 `rollback/`。
- 若查询失败，保留错误摘要与查询编号；不得通过执行写操作“修复”或绕过。
- **禁止输出敏感明文：** Function Body 全文、QueryText、DataSource 真实值/路径、连接字符串、SAS/Token、账号、原始 Gateway 日志、未经脱敏的数据库/集群 ID。

## 3. 必须返回的结果与证据

请将结果写入**一个中文 Markdown 文件**：`gateway-preflight-YYYYMMDD-HHmm-SGT.md`（文件名使用真实执行时间）。必须包含以下内容：

### A. 执行元信息
- 实际查询执行时间（UTC 与 UTC+8）、目标为 POC 的核实依据、MCP 工具名称、GitHub 仓库、分支、所用文件路径或 Commit SHA。
- 每个查询的执行状态：`PASS / FAIL / UNSUPPORTED / INCONCLUSIVE`，并说明错误或限制。

### B. Function Drift（3 行）
表格列：`FunctionName | CurrentExists | BaselineExists | BodyMatch | CurrentSHA256 | BaselineSHA256 | Notes`。
- 只比对：`parse_QueryExecutionAggregationReport`、`parse_QueryStreamingReport`、`parse_MashupEvaluationReport`。
- 在本地**对实际 Body 进行严格相同比较**（如空白规范化，要明确标注方法）；比较 SHA256 时保证两侧使用同一编码及规范化规则。
- 报告 SHA256 和匹配结果，不粘贴完整 Body。若无法获取基线或无法算 Hash，就写 `Not verified`。

### C. Update Policies（3 行）
表格列：`FunctionName | PolicyFound | Enabled | Source | IsTransactional | Result | Notes`。
- 预期 `Enabled=true`、`Source=GatewayReports-Raw`、`IsTransactional=true`，实际以现场读取为准。
- 需要列出缺失、禁用、引用了其他函数或其他源表的情况；只查到部分结果不能记为 PASS。

### D. Raw / Parsed 当前快照（每种类型 2 行，理想共 6 行）
表格列：`Layer | logType | RowCount | MessageCount | LatestTimeUTC | TimeKind | Notes`。
- 三种类型：QueryExecutionAggregationReport、QueryStreamingReport、MashupEvaluationReport。
- Raw 和 Parsed 保留期与时间字段语义不同，**不要直接比较累计 Rows 认定丢失/无重复**；只报告实际值。Raw 没有数据或时间字段无法查询时明确写 `No data` / `Not verified`。

### E. 过去 2 小时 Update Policy failures
表格列：`TargetTable | FailureKind | FailureCount | FirstFailureUTC | LastFailureUTC`，并列出返回的**总失败条数**。
- 若没有结果写 `0 observed`，**不是** `no ingestion problems guaranteed`；如果查询不支持写 `UNSUPPORTED`。
- 错误详情只提供脱敏摘要，不能泄露日志正文。

### F. 结论与交接
- `Overall: Preflight Passed / Blocked / Inconclusive` 三选一。
- `Blockers`：明确列出异常、配置漂移、失败记录和无法验证的事项。
- `What is verified` 与 `What is NOT verified` 分开列出（不要将此 Preflight 声称为端到端完整性、无重复或 PROD Ready）。
- `Next decision for ChatGPT`：只写**是否满足人工评审 POC Function Patch 的条件**，不要主动部署。
- 在附录中附上**四个查询的脱敏结果表/计数**，便于 ChatGPT 不依赖你的结论独立复核。不得只给“全部成功”或主观总结。

## 4. 判定规则

- `Preflight Passed`：四项查询均成功，3 个 Function 与基线匹配，3 个 Policy 符合预期，没有观察到 Update Policy ingestion failures，且 Raw/Parsed 的可用性有充分证据；并不代表可以自动部署。
- `Blocked`：Function 明确不一致、关键 Policy 配置错误、出现相关摄入失败，或确定执行在非 POC。
- `Inconclusive`：MCP 不支持关键查询、无法获取基线、Raw/Parsed 缺少必要证据或结果不完整。不得把未知判为通过。

## 5. 最终交付（最重要）

**请直接向我返回生成的完整 Markdown 报告文件，并在回复中提供可复制的 Markdown 全文或该文件的真实本地路径。** 我会把它上传到当前 ChatGPT 对话，由 ChatGPT 复核、更新 GitHub 及决定下一步。若无法保存文件，直接在聊天中完整贴出 A–F 六部分和附录，不要只给结论。

**最后再次确认：只执行只读操作，不修改任何 Eventhouse 对象。**
