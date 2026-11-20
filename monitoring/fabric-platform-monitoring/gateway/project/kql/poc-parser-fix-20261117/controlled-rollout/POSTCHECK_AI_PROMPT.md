# 本地 AI / Eventhouse Remote MCP｜Streaming Parser 部署后只读回归

> **仅在用户已手动完成 POC `parse_QueryStreamingReport` 修改之后运行。禁止自动写操作。**

**GitHub 仓库**：https://github.com/ZeonZheng/fabric-toolbox  
**分支（不是 main）**：`docs/gateway-monitoring-poc-20261112`  
**查询目录**：https://github.com/ZeonZheng/fabric-toolbox/tree/docs/gateway-monitoring-poc-20261112/monitoring/fabric-platform-monitoring/gateway/project/kql/poc-parser-fix-20261117/controlled-rollout

## 输入（必须让用户提供真实值）

- `RecordedAfterPatchUTC = <RecordedAfterPatchUTC>`（用户执行 `print RecordedAfterPatchUTC=now()` 获得的 UTC 时间，**不得自行虚构或沿用示例**）
- `UserConfirmedPOC = true/false`（只有用户在 Fabric UI 人工确认后才设 true）

若没有实际 UTC 值或未确认 POC，**不要开始回归**，报告 `Inconclusive`。

## 查询与执行

你已连接 `eventhouse-remote` MCP。先确认工具及只读调用参数，验证目标确为用户确认的 POC Gateway Monitoring 数据库，再从**指定工作分支**打开以下文件：

1. [01-streaming-postdeploy-summary.kql](01-streaming-postdeploy-summary.kql)
2. [02-streaming-postdeploy-keycheck.kql](02-streaming-postdeploy-keycheck.kql)
3. [03-streaming-ingestion-failures.kql](03-streaming-ingestion-failures.kql)

对三份 KQL 中的示例时间 `2099-01-01T00:00:00Z`，仅替换成用户提供的**真实 UTC**时间；保留其它逻辑，不执行写命令。每份分别通过 MCP 的只读 KQL 工具运行。如出现语法/运行错误，应标注 `ERROR` 并返回错误摘要，不得自行修改 Eventhouse 配置。

**特别注意过滤语义：** Raw 的 `logDate` 是 envelope 到达时间；Parsed 的 `StreamingStartTimeUTC` 是日志业务时间，两者不能视作相等。Keycheck 的关联也可能因 ID 复用/旧事件时间导致漏匹配。须将这一限制写入报告；若只有不充分证据，标记 `Inconclusive`，不能把行数不等直接判为数据丢失。

另外可用只读命令确认 `.show functions` 中 `parse_QueryStreamingReport` 的 BodySHA256 是否不同于 2026-11-20 Preflight 原始 SHA256 `d946208e13a5009a2695a8b52f6ce0c8ed1d2e46d8d148cb069b826bc34f805c`。注意：**Hash 不同只能证明函数变化，不单独证明修复成功**。

## 输出契约（交给 ChatGPT，勿仅回复“成功”）

请生成**一份中文 Markdown 文件** `gateway-streaming-postcheck-YYYYMMDD-HHmm-SGT.md`，并提供其真实路径/完整文本。必须包含：

| 部分 | 必须报告 |
| --- | --- |
| 1. Environment & Change | 用户已核实 POC 与否、实际 `RecordedAfterPatchUTC`、MCP 工具、仓库分支、3 个查询的成功/失败 |
| 2. Function Check | Function 仍存在、Body SHA256、与旧版 Hash 是否不同 |
| 3. Raw & Parsed | 各自 Rows、WithStreaming、WithSpooling、最新时间和时间字段名称；区分历史数据 vs Patch 后数据 |
| 4. Sample Key Check | Raw Keys 数、MatchedParsed、两项数值 Matches/Mismatches；标注 KeyJoin 的限制 |
| 5. Ingestion Failures | 修改后 Update Policy failures 的条数、类型、发生时间（脱敏） |
| 6. Verdict | `Passed / Blocked / Inconclusive`、有证据的结论、未验证项、建议下一人工决策 |

- `Passed` 须有部署后新摄入 Raw 及对应 Parsed 样本、两项 Duration 值恢复且值合理、无相关摄入失败，确认现场函数已改变。
- `Blocked` 用于确认有新的 Update Policy 失败、字段转换异常/持续 null 或其它实际影响。
- 没有新 Raw 事件、MCP 关键查询不可用、无法排除时区/时间字段导致的漏匹配：`Inconclusive`。
- 不粘贴 Function Body、Raw 日志内容、QueryText、真实 DataSource、源文件路径、用户信息、Token。附**脱敏查询结果/计数表**以便 ChatGPT 独立复核。

**严格禁止**：自行执行 `.create-or-alter`、任何修改 Policy 的命令、Rollback、Uploader 变更、PROD 操作。

完成后把该 Markdown 文件交给用户上传 ChatGPT，由 ChatGPT 做独立分析与 GitHub 状态更新。
