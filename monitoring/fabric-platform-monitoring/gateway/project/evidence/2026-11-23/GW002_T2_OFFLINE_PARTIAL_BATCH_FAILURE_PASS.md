# GW-002 / P2-T2 — 离线 Partial Batch Failure 缺陷复现 PASS

> **证据来源：** 用户在 POC Gateway VM、PowerShell 7.6.10 手动执行由 ChatGPT 提供的隔离测试并返回完整 PowerShell 输出。用户运行的测试仅读取本地 `D:\Ang\FabricGatewayMonitoring\Modules\Utils.psm1` 中 `Add-LogToEventHub` 函数定义，动态加载该函数体，通过**内存 Mock `Add-MsgEventHub`** 替代实际网络发送。使用 `$env:TEMP\GW002-T2-<GUID>` 的合成 CSV；未连接 Eventstream/Eventhouse；未修改真实 Gateway Report/Temp/History、Task、Uploader 配置或 PROD。

## 测试设置

- 使用 `2001` 行合成 CSV（`RowId=1..2001`），按源码自身的分批逻辑：第一批 2000 行（RowId 1–2000）、第二批 1 行（RowId 2001）。
- 首次运行在第二次调用 Mock `Add-MsgEventHub` 时抛出 `SIMULATED_BATCH_2_FAILURE`；第一批已由 Mock 标记 Accepted。
- 第二次调用同一实际 `Add-LogToEventHub` 函数，从同一个 CSV 文件开头重新执行；Mock 不抛异常。

## 用户返回的实际输出

| Run | Accepted Rows | FirstRow | LastRow |
| --- | ---: | ---: | ---: |
| 1 | 2000 | 1 | 2000 |
| 2 | 2000 | 1 | 2000 |
| 2 | 1 | 2001 | 2001 |

- `FirstRunError=SIMULATED_BATCH_2_FAILURE`
- `AcceptedBatchCount=3`
- `FirstBatchCopies=2`
- `DuplicateBatchObserved=True`
- **累计 Mock accepted 4001 行，其中重复批次的 2000 行在模拟接收端出现 2 次；原 CSV 唯一行数 2001。**

## 独立技术判定

**P2-T2 = PASS（成功复现重复发送的应用行为）；GW-002 / P2 Collector Reliability = NOT PASSED。**

源码级机制：`Utils.psm1` 的 `Add-LogToEventHub` 在内存中推进 `$minIndex`/`$maxIndex`，没有把已成功 Batch 的进度、稳定 `BatchId` 或 `RecordId` 持久化。重新处理同一 CSV 时从第一行重新生成 Batch 并再次调用发送端。P2-T1 已验证 PowerShell 7.6.10 的 Temp 同路径 `Move-Item` 不报错，因此对残留 Temp 文件再次上传的路径在现实中具备执行条件，但**本次 T2 没有完整调用 ProcessLogFiles/Task，不能声称已复现完整重启链路**。

关键限制：Mock 的 `Accepted` 是**模拟接收结果**，不是 Eventstream HTTP 201，也不是 Eventhouse Raw/Parsed 重复记录的实测。真实网络重试或下游去重行为尚未验证；不可写成“Eventhouse 已发生重复”或“有 2000 行真实重复记录”。

## 方案推论（非已实现）

- 增加 `MaximumRetryCount` 不能解决“发送成功但随后流程中断/应答不确定”的幂等问题。
- 源消息当前包含 `logType`、`log`、`logDate`，其中 `logDate` 按调用时钟生成，不能作为稳定重试去重键；需确定源文件固定快照与确定性 `BatchId`/`RecordId`（例如 Node + Snapshot/Version + RowOrdinal），并决定在哪一层做幂等/去重与对账。谨防将相同业务值的合法不同记录误删。
- `Manifest` 中仅保存“已提交”状态不够覆盖“服务器已接收而客户端未获确认”的不确定窗口；需要配套稳定 ID 与下游去重/可验证机制。
- **下一步不是继续重复模拟重复问题**，而是由 ChatGPT 分析当前 [Upload Manifest 设计草案](../../docs/PROD_UPLOAD_MANIFEST_DESIGN.md)，给出 POC 可行的最小幂等/对账方案及验收门槛。后续实现必须单独获得批准；PROD 仍禁止变更。

相关：[GW-002](../../tracking/ISSUES.md) · [P2-T1](GW002_T1_SAME_PATH_MOVE_RESULT.md) · [Uploader 源码独立评审](UPLOADER_SOURCE_RELIABILITY_REVIEW.md)。
