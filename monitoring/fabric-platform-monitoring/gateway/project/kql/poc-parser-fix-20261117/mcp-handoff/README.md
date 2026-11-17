# Eventhouse Remote MCP｜POC 上线前只读检查

> **适用对象：** 已连接 `eventhouse-remote MCP` 的本地 AI。**日期：** 2026-11-17 (Asia/Singapore)。
> **目标：** 自动执行最小只读 Preflight，不修改 Gateway / Eventhouse / 上游文件。不处理真实 DataSource、QueryText、凭据或源日志正文。

## 直接交给本地 AI 的执行任务

在当前 **POC Gateway Monitoring KQL Database** 中，先检查 Eventhouse Remote MCP 暴露的工具及其输入 Schema；仅使用实际支持的**只读 KQL 查询**能力，不要猜测工具名或参数。请按顺序运行本目录四份 KQL 文件（每份单独请求，不把以 `.show` 开头的管理查询与其他语句混入同一请求）：

1. [01-live-parser-definitions.kql](01-live-parser-definitions.kql) — 捕获当前三个已部署的 Parser Body，和 GitHub 2026-11-17 原始部署快照对比，标记是否有变更。**无需输出任何原始 DataSource 数据**。
2. [02-update-policies.kql](02-update-policies.kql) — 三个相关 Update Policies 是否为 Enabled、Source=`GatewayReports-Raw`、IsTransactional=true。不得改写 Policy。
3. [03-recent-data-snapshot.kql](03-recent-data-snapshot.kql) — 记录 Raw 最近可查询的三类日志与 Parsed 三表的**各自**行数/最新时间。Raw 的 `logDate` 是 Envelope 时间；Parsed 的结束/开始时间属于业务日志时间；Retention 不同，**不做跨层总数相等判定**。
4. [04-update-policy-failures.kql](04-update-policy-failures.kql) — 检查过去两小时 Update Policy ingestion failures。如果命令不被 MCP 支持，直接报告 `unsupported`，不要尝试绕过。

输出一份简短的**中文** Markdown 结果，包含：运行时间（注明时区）、MCP 名称、每项结果及查询状态、三种 Parser 是否和 baseline 一致、三条 Policy 状态、Raw / Parsed 的各自 Rows / 最新时间、最近两小时 failures、阻塞项、结论（仅允许 `Preflight Passed`、`Blocked` 或 `Inconclusive`）。如能自动保存结果，请将**脱敏汇总**存为 Markdown/JSON；不要把完整原始日志、SourceFile、QueryText、DataSource 路径、SAS Token、个人/内部账号信息输出到共享记录。没有数据的检查要写 `No data / Not verified`，不要写成 Pass。

**严格禁止：** `.create`、`.alter`、`.delete`、`.drop`、`.set`、`.ingest`、修改 Uploader/Task、触发 Jumpstart 重新安装、自动执行 `candidates/` 或 `rollback/`。即使所有 Preflight 通过，也**不要自行部署**：只返回报告，由人工复核下一步。

## 已完成的检查（请勿重复）

- Dry-run 3/3；Schema 39/39；13 个数值指标无丢失、无超阈值变化。
- DataSource Legacy vs Candidate：Aggregation 96/96、Streaming 139/139，`DifferentSerialized=0`；Mashup 的 `DataSourceKinds`、`DataSourcePaths` 在这次 103 条 Raw 样本中都不存在。
- 最终确认当前 Gateway POC Function 变更之前，必须重新获取**现场**状态，防止 Git 与实际部署发生漂移。

证据：[2026-11-17](../../../evidence/2026-11-17/README.md) · [状态](../../../STATUS.md) · [GW-001](../../../tracking/ISSUES.md)。

**说明：** 这四份 Preflight KQL 作为本地 AI 执行输入，已提交 GitHub，但尚未通过用户的 `eventhouse-remote` MCP 实际调用验证。如果 MCP 不支持管理查询，说明受限并仅执行其支持的只读操作。


## 可直接发给本地 AI 的完整 Prompt（含 URL 和输出契约）

请使用 [LOCAL_AI_PROMPT.md](LOCAL_AI_PROMPT.md)。其包含**完整 GitHub 地址、指定工作分支、四份查询链接、15 Function 基线地址、逐项状态与结果表格、单个中文 Markdown 报告及 ChatGPT 交接要求**。本 README 保留简版技术说明；对外执行指令以该 Prompt 为准。
