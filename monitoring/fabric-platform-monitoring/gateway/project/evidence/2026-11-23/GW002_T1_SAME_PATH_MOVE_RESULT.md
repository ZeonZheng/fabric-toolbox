# GW-002 P2-T1 — PowerShell 7.6.10 同路径 Move-Item 隔离实测

> **来源：** 用户在 POC Gateway VM PowerShell 7.6.10 手工运行 ChatGPT 提供的隔离脚本，并粘贴控制台完整结果；ChatGPT 未直接在 VM 执行。此测试仅修改并清理 `$env:TEMP\GW002-T1-<GUID>` 中的合成测试文件，不调用真实 Uploader、Eventstream 或 Gateway 日志。

## 输入与结果

- 在唯一路径下创建 `MashupEvaluationReport_TEST_x.log`，将 `$fileOutputPath` 设为其**自身绝对路径**。
- 执行 `Move-Item -Path $logFile.FullName -Destination $fileOutputPath -Force -PassThru -ErrorAction Stop`，复现 `ProcessLogFiles` 对已在 Temp 目录文件的同路径 Move 形状。
- 实际：`PowerShellVersion=7.6.10`，`Attempts=1`，`FileReady=True`，`FileStillExists=True`，`Error=""`。

## 独立判断与对先前假设的更正

**P2-T1 单项行为测试 PASS：同一路径 Move 在该环境/该测试条件下立即成功，且文件保留。**

- **明确排除：** “Temp 残留文件因为 Source=Destination 相同而必然 Move 失败”，这一先前假设在 PowerShell 7.6.10 实测**不成立**。
- 不能由此证明所有 Temp 恢复场景成功：例如文件锁、ACL、目标目录/磁盘异常仍可能触发 Move 报错；源码 `UploadGatewayLogs.psm1` 中 `fileReady=false` 后仍执行 Merge 的缺陷路径在静态层面存在，但故障后实际是否造成数据损失**未被本次实验验证**。
- 因此**下调“同路径 Move”风险优先级**；不再对其重复测试。重心转向有明确设计缺口的 **HTTP Batch 部分成功/重试幂等、无持久化 checkpoint 和文件级可追踪性**。
- **P2 Collector Reliability 整体仍未通过**。单项 T1 成功≠上传完整性验证。

## 推荐下一步（最小只读前置检查）

在 POC VM 仅读取现场 `Config.live-poc.json` 的 `ConnectionProperties` **存在性与非敏感的 `MaximumRetryCount` / `RetryIntervalSec` 数值**（不输出其他配置内容或密钥）。对应源码 `Utils.psm1:362–369` 从 `$config.ConnectionProperties` 读取该参数，而 `UploadGatewayLogs.psm1:222–229` 仅在局部 `$ConnectionProperties` 中指定默认值；需澄清实际 POC 是否启用 HTTP 重试。该只读配置检查后由 ChatGPT 设计针对部分 Batch 失败的隔离、无网络模拟测试。

相关：[源码评审](UPLOADER_SOURCE_RELIABILITY_REVIEW.md) · [VM 递归快照](GW002_VM_RECURSIVE_FOLLOWUP_20261123.md)。
