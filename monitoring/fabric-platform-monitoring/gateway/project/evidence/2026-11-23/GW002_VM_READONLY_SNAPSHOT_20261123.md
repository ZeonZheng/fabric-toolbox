# GW-002 Gateway VM 只读运行快照｜2026-11-23

> 用户在 POC Gateway VM 上手工执行 ChatGPT 提供的 PowerShell 7.6.10 只读检查，并粘贴结果；本助手没有直接接入 VM 或更改配置。

## 实际返回结果

- **VM：** `RNGVMC0046`；PowerShell `7.6.10`；`C:\Users\powrbi\AppData\Local\Microsoft\On-premises data gateway\Report` 存在。
- **Scheduled Task：** `\FabricGatewayMonitoring\Gateway-UploadLogs`，State=`Running`，LastRunTime=`2026-11-22 06:44:44`（PowerShell 未附时区信息），LastTaskResult=`267009` (`0x41301`，任务正在运行)，NextRunTime 无值。
- **Source（Root 非递归 `*Report_*.log`）：** 0 文件 / 0 MB。
- **Temp（`Report\Temp` 非递归 `*Report_*.log`）：** 0 文件 / 0 MB。
- **History（`Report\History` 非递归 `*Report_*.log`）：** 88 文件 / 24.69 MB；8 类 Performance Report 各 **11** 文件：ConcurrentOperationAggregation、MashupEvaluation、OpenConnection、QueryExecutionAggregation、QueryExecution、QueryStart、QueryStreaming、SystemCounterAggregation。
- **History 的最早/最新 LastWriteTimeUtc：** 显示 `2026-11-13 15:01:44` / `2026-11-23 05:45:16`（以 VM 实际报告值为准）。
- **Uploader 错误日志：** `D:\Ang\FabricGatewayMonitoring\logs\GatewayMonitoring.log` 未找到；未查实际启动参数/脚本的 `logFolder`，不可推断没有错误。

## 判断与限制

1. Task 当前 Running、History 有文件且最新写入在本日，是运行/本地归档的正向证据；`267009` 并非失败代码。单凭 Task Running 不是 HTTP 成功或端到端无丢失保证。
2. **先前快照脚本只对三目录直接执行 `Get-ChildItem -File`，不是递归。** Uploader 实际用 `-Recurse` 扫描 `*Report_*.log` 并排除 `*_f.log`。因此“Source=0 / Temp=0”只表示**顶层匹配文件为零**，不能排除子目录的残留与其它实际源文件名，尤其是 Temp 可能位于扫描根路径以外的层级结构。
3. Temp 未观察到残留不代表源码中 `fileReady=false` 与无条件 Merge 组合无风险；不会在 Live Source 人为制造失败。
4. History 每类 11 份与 `ReportRetention=10` 配置的具体清理语义需另独立核对；创建/写入日期与原始事件日期不同，不据此推断无遗漏。
5. Error Log 路径不存在可能是正常无错误、实际 `logFolder` 不同，或其它日志渠道；不宣布“无错误”。
6. 截图/复制中未提供任务 Action 的准确当前参数与递归文件枚举；下一项只读补充检查即可。

## 下一步 — 单次只读补充快照

只读取得 Scheduled Task Action 的可脱敏摘要（脚本名、是否引用 `Config.live-poc.json`，不输出完整可能带 Secret 的命令行）及 `Report` 的递归 `*Report_*.log` 分类，区分 Source/Temp/History（其中 `*_f.log` 为归档文件）。只报告计数、总大小和最近修改时间，不输出 CSV 内容、QueryText、文件路径明文或配置密钥。完成后由 ChatGPT 独立判断下一 POC 实验是否可以安全开展。