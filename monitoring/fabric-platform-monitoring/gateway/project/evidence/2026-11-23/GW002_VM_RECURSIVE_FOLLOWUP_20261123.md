# GW-002 Gateway VM 递归目录与 Task Action 复查｜2026-11-23

> 用户在 POC VM `RNGVMC0046` 使用 PowerShell 7.6.10 手工运行只读查询并粘贴终端输出。本助手未直接访问 VM。无文件或任务修改。

## 结果

- 对 `C:\Users\powrbi\AppData\Local\Microsoft\On-premises data gateway\Report` 使用 `Get-ChildItem -Recurse -File` 枚举符合 `*Report_*.log` 的文件，按 Source / Temp / History 及 Report Type 分组。
- 输出**仅 History 类型**，八种类型各 **11 个**，总计 **88** 个；按类型 MB 汇总：
  - ConcurrentOperationAggregationReport 0.59 MB，最新 2026-11-23 05:45:16 UTC；
  - MashupEvaluationReport 1.16 MB，最新 2026-11-23 03:29:16 UTC；
  - OpenConnectionReport 1.22 MB，最新 2026-11-23 03:29:16 UTC；
  - QueryExecutionAggregationReport 0.83 MB，最新 2026-11-23 03:30:16 UTC；
  - QueryExecutionReport 1.84 MB，最新 2026-11-23 03:29:16 UTC；
  - QueryStartReport 7.01 MB，最新 2026-11-23 03:29:17 UTC；
  - QueryStreamingReport 1.45 MB，最新 2026-11-23 03:19:17 UTC；
  - SystemCounterAggregationReport 10.59 MB，最新 2026-11-23 05:45:16 UTC。
- **Source/Temp 在本次递归扫描时没有符合命名模式的 Report 文件**。这是时点快照，不保证这些目录持续为空，也不代表目录内不存在其他文件。
- Scheduled Task `\FabricGatewayMonitoring\Gateway-UploadLogs` Action 摘要：`Executable` 返回 `pwsh.exe"`（末尾引号为命令存储/显示问题待核实，本轮不据此认定错误）；`UploaderScriptMatched=True`、`LivePocConfigMatched=True`、`WorkingDirectoryConfigured=True`。上一轮 Task `Running`、LastTaskResult=267009 (0x41301)。
- 此轮仍未展示 Task Action 全部参数、未访问敏感配置内容、未查询 Eventhouse、未读取 CSV/QueryText。

## 独立判断

1. 已解决上一轮**非递归目录扫描的证据缺口**：当时没有可见的待上传或 Temp Report 文件积压。
2. `History` 有持续更新证据，不能推导已完整上传 Eventhouse、没有数据遗漏或 Exactly-once。
3. 当前没有 Temp 残留事件可供直接检查；源码中 `fileReady=false` 后仍执行 `Merge-ReportFiles` 的路径风险未被现场证伪/证实。**停止反复查询同一快照指标。**
4. 下一步先在与 Gateway Log Root 完全隔离的临时测试文件夹验证 **PowerShell 7.6.10 `Move-Item -Path same -Destination same -Force -PassThru` 实际行为**，不执行实际 `ProcessLogFiles`，不发 HTTP，不动 VM Source/Temp/History/Task。根据是否抛出错误，再设计针对 `fileReady=false` / Merge 的受控单元测试。
5. 不修改 PROD，也不对现有 Gateway 日志实施故障注入。

相关：[上一轮 VM 快照](GW002_VM_READONLY_SNAPSHOT_20261123.md) · [Uploader 源码评审](UPLOADER_SOURCE_RELIABILITY_REVIEW.md)。
