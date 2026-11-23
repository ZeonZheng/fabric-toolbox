# GW-002 HTTP Retry 配置快照及机制评审｜2026-11-23

> 用户在 POC VM 的 PowerShell 7.6.10 只读读取 `D:\Ang\FabricGatewayMonitoring\configs\Config.live-poc.json` 中三个非敏感字段，返回脱敏结果。本助手没有直接访问配置或 VM，没有执行写入或真实 HTTP 请求。

## 配置证据

| 设置 | 现场值 |
| --- | --- |
| ConfigFound | `True` |
| RetryConfigExists | `True` |
| `ConnectionProperties.MaximumRetryCount` | **3** |
| `ConnectionProperties.RetryIntervalSec` | **1 秒** |

## 独立源码对应

- `Utils.psm1:362–369` 中 `Add-MsgEventHub` 调用 `Invoke-RestMethod`，使用 `$config.ConnectionProperties.MaximumRetryCount` 与 `RetryIntervalSec`；现场顶层配置存在且完整，早前怀疑的**缺少这两个属性**问题在本 POC 实际配置中排除。
- 根据 [Microsoft Learn PowerShell 7.6 Invoke-RestMethod](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-restmethod?view=powershell-7.6)，`-MaximumRetryCount` 是 HTTP **400–599 或 304** 响应情况下的附加重试次数。配置 3 表示首次尝试之外最多 3 次重试；`RetryIntervalSec=1` 为通常重试间隔，HTTP 429 若返回 Retry-After 则按该响应头调整。不要把此机制等同于网络连接超时必然重试、事务成功、持久化日志 checkpoint。
- `Utils.psm1:240–291` 的 `Add-LogToEventHub` 以 `Import-Csv` 读取整个 Report 文件，从内存 `$minIndex` / `$maxIndex` 处理每个 Batch；每次成功发送后递增下标，但没有保存持久化 Batch offset、稳定文件 ID、已确认的 Batch receipt。
- `UploadGatewayLogs.psm1:236–242` 每次新轮询递归扫描剩余 Report 文件，`ProcessLogFiles:86–120` 仅当函数正常返回后进入 History Merge；`Run-UploadGatewayLogs.ps1:69–98` 由外层 catch 记录错误并在后续轮询重试。若早期批次已经成功但较后批次最终失败，**该文件再次处理时从第一行重发**；存在重复风险。不同阶段错误导致的文件是否完整保留、何时可扫描，需隔离测试，不据源码宣称一定重复。
- `P2-T1` 已证实同路径 Move 成功：不应再次把同路径操作当作必然报错。

## 下一步决定

**不改现有 Retry 配置**，停止重复配置检查。下一步 P2-T2 应设计一个**离线模拟发送端**，以合成 CSV 两个及以上 Batch 复用实际 `Add-LogToEventHub` 分批逻辑，模拟第一批接收成功、第二批失败、再重新处理同一文件；所有请求交给内存 Mock，**绝不能访问真实 Eventstream、Gateway 日志或触发 Schedule Task**。只输出发送 Batch 数、批次编号/范围与重复统计。

此测试只验证**应用代码的失败/恢复行为**，不是生产 Eventhouse 去重机制的性能证明；之后再独立决定是否需要受控端到端验证。
