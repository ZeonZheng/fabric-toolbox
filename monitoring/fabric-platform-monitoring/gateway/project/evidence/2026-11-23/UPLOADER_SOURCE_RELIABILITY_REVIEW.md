# Gateway Uploader 源码可靠性独立评审｜2026-11-23

> **执行人：** ChatGPT 直接读取 GitHub 工作分支源码（未将任务委托本地 AI）。**范围：** 仅静态源码分析，不代表已在 POC VM 触发或观察到故障；未执行故障注入、VM 文件操作或 Eventhouse DDL。

## 范围及源码

- [`UploadGatewayLogs.psm1`](https://github.com/ZeonZheng/fabric-toolbox/blob/docs/gateway-monitoring-poc-20261112/monitoring/fabric-platform-monitoring/gateway/PowerShellScript/modules/UploadGatewayLogs.psm1)
- [`Utils.psm1`](https://github.com/ZeonZheng/fabric-toolbox/blob/docs/gateway-monitoring-poc-20261112/monitoring/fabric-platform-monitoring/gateway/PowerShellScript/modules/Utils.psm1)
- [`Run-UploadGatewayLogs.ps1`](https://github.com/ZeonZheng/fabric-toolbox/blob/docs/gateway-monitoring-poc-20261112/monitoring/fabric-platform-monitoring/gateway/PowerShellScript/Run-UploadGatewayLogs.ps1)

## 源码确定的处理路径

1. 扫描 `$path\*Report_*.log`，`-Exclude "*_f.log"`，`-Recurse`；没有按 LastRun / LastWrite 过滤 Report，也没有按时间排序（`UploadGatewayLogs.psm1:231–242`）。
2. 按文件 foreach 串行处理；Report 先 Move-Item 到目录 `Temp`，遇文件已在 Temp 就把目标设为自己的原路径，再继续执行 `Move-Item`（`UploadGatewayLogs.psm1:18–68`）。
3. `$fileReady` 仅决定是否调用 `Add-LogToEventHub` / Lakehouse；后面 `if ($isReport)` 的 Merge 分支**不依赖 `$fileReady`**（`UploadGatewayLogs.psm1:86–120`）。
4. `Add-LogToEventHub` 从 Import-Csv 全量读取文件，按约 2000 行和约 980 KiB 请求上限分组，将 `logType`、`log`、**发送时生成的** `logDate` POST 给 Eventstream（`Utils.psm1:227–293`）。
5. HTTP REST 在 `Invoke-RestMethod` 调用处配置 `MaximumRetryCount`、`RetryIntervalSec`，但未看到与 CSV 记录对应的持久化 Batch offset/ID 或远端摄入确认。入参读取 `$config.ConnectionProperties`，需要核对 POC 实际配置是否指定（`Utils.psm1:295–371`；`UploadGatewayLogs.psm1:222–229` 仅创建了不同对象上的局部默认配置）。
6. 成功返回上传调用后，`Merge-ReportFiles` 将 Temp CSV append 到本地同类型 History，最后对 Temp `Remove-Item`（`Utils.psm1:406–440`）；Merge 后调用历史清理；主入口 `$ErrorActionPreference="Stop"`，catch 记录 `GatewayMonitoring.log`，连续 1 分钟过 5 次错误可能退出进程（`Run-UploadGatewayLogs.ps1:26,69–100`）。

## 我自己的风险判断（按先后次序，不等同于现场故障）

| 优先 | 风险 | 源码基础 | 尚未证实的部分 |
| --- | --- | --- | --- |
| **1** | **Temp 残留文件可能被错误归档/删除，而没有重新上传** | Temp 文件会被递归扫描；对自身执行 `Move-Item`；失败可置 `fileReady=false`，后续 Merge 不受该状态保护 | 必须在受控 POC 确认 `Move-Item` 同路径在当前 PowerShell 版本如何表现，以及是否进入 Merge；不能由源码断定实际已经丢数据 |
| **2** | **部分 Batch 成功后异常退出，后续从文件首行重传可能导致重复** | 请求后只推进内存下标，没有持久化 Batch checkpoint 或已发送消息标识 | 失败后 Temp 真实恢复路径还要验证；HTTP 请求超时也可能是服务已收、客户端未收到响应 |
| **3** | **HTTP 接收成功 != Eventhouse 入库成功** | 本地归档依据请求返回，不做 Raw/Parsed 的稳定文件/Batch 粒度核对 | 需要有 ingestion metrics / 原始文件可审计 ID 才能证明 end-to-end |
| **4** | **大文件 / Merge 中断 / 非严格错误处理** | Import-Csv 一次加载整文件；Merge Append 后 Remove-Item 忽略删除错误 | 大文件内存压测、部分 append、重复归档需要单独实验；不能称已发生 |

## 本轮不做的事情

- 不直接委派本地 AI 阅读代码并判断架构；**技术分析由本助手负责**。
- 不做故障注入、修改 Task Scheduler/上传脚本/网关日志、重跑上传或 PROD 变更。
- 不从 `GatewayReports-Raw` 的短窗口统计推断源文件全量 Exactly-once。

## 下一步建议（唯一、低风险）

**先做 POC VM 的只读现场快照**：确认 `Gateway-UploadLogs` 实际 Action 目标、PowerShell 版本、Task LastRunResult、`Report\Temp` 是否有遗留文件及文件名/大小/时间戳、近期 `GatewayMonitoring.log` 的错误数和匿名分类，并比对 `Source/History` 文件计数与大小。**不修改任何文件/任务，也不输出 Raw 或 Credentials**。

只有确认真实环境和风险触发条件之后，再由本助手设计**一个受控隔离的**故障实验，明确是否需要用户手动批准。Eventhouse 的短窗口与源码审计不能替代 VM 证据。若本地 AI 没有 VM 文件访问能力，改由用户在 VM 上运行一份审核过的只读命令收集摘要，而不是假定 AI 可访问。
