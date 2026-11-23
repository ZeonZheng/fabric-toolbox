# PROD 设计候选｜Upload Manifest 与端到端审计

> **决策记录：** 2026-11-17。此文仅记录**拟采用的设计方向**；尚未实施、未批准在 PROD 上线。当前 POC 仍使用 `Move-Item → Upload → Merge-ReportFiles` 原始机制。

## 业务与技术问题

当前 Eventstream 消息 Envelope 有 `logType`、`log`（CSV 记录数组）与 `logDate`，缺乏源文件身份、记录位点及批次标识。`HTTP 201` 只表明接收请求，不代表 Eventhouse Raw/Parsed 完整摄入或 Exactly-once；本地 `History/*_f.log` 合并后也难以追踪原始文件边界。

## 候选 Manifest 字段（设计草案，不是已部署 Schema）

| 字段 | 预期用途 |
| --- | --- |
| `ManifestId`、`NodeId` | 稳定唯一标识、来源节点；不应只依赖文件名 |
| `SourceFileName`、`SourceFileIdentity` | 文件原始标识、截断/替换/轮转检测 |
| `ReportType`、`OriginalFileSizeBytes` | 类型清单、大小及异常排查 |
| `ContentSHA256`、`RecordCount` | **固定快照**下的完整性校验 |
| `FirstEventUtc`、`LastEventUtc` | CSV 内事件实际时间范围，不能由文件名推断 |
| `ExpectedBatchCount`、`AcceptedBatchCount` | 部分上传进展；每批仍需 BatchId |
| `State` | `Discovered → Snapshotted → Sending → Accepted → RawVerified → ParsedVerified`；另有 `Failed/RetryPending` |
| `Attempts`、`LastError`、`UpdatedUtc` | 重试和失败审计 |
| `ParserVersion`、`SchemaVersion` | 区分文件传输完整性和后端解析兼容性 |

## 2026-11-23 P2-T2 实证对设计的影响（仍为候选方案）

- [P2-T2 离线测试](../evidence/2026-11-23/GW002_T2_OFFLINE_PARTIAL_BATCH_FAILURE_PASS.md) 使用真实 `Add-LogToEventHub` 分批函数 + 内存 Mock，在“Batch1 已接受、Batch2 失败、下一轮重处理文件”情况下观察到 **Batch1 被接受两次**。这是**应用重发行为复现**，不是实际 Eventhouse 已产生重复行的证明。
- 由于源消息的 `logDate` 每次发送即时生成，**不能用 `logDate` 判断是否是同一 Batch 的重试**；现有消息没有固定 `SourceSnapshotId`、`BatchId` 或行身份。
- **最小候选设计需要同时包含三部分，缺一不可：**
  1. **固定 Source Snapshot + 稳定身份**：NodeId、SnapshotId（不可依赖仅文件名）、BatchId（确定性批次划分）、RecordOrdinal/RecordId；重试必须重用同一身份。
  2. **持久化可恢复状态**：Upload Manifest 记录快照行数、批次数、每批尝试/确认/失败状态；`HTTP accepted` 不直接等于 `RawVerified`。
  3. **下游幂等及审计**：在 Raw/Parsed/最终报告的明确位置去重或实现幂等写入，并核对各 Snapshot 的期望行数、BatchId/RecordId、已到 Raw 和 Parsed 的行数；对重复重试和 Source 自身合法的相同行值做区分。
- **仅保存上一次成功的 Batch 下标不足以保证无重复**：若服务器接受请求但客户端超时，状态可能来不及持久化。因此仍需稳定身份 + 下游幂等/去重或其他已证明机制。
- **目前暂不选定 KQL 物化/查询时去重、外部 Durable Queue、还是上传层幂等协议**，因其可用性、成本与现有 Parsed Update Policy 兼容性尚未评估。先由 ChatGPT 形成 POC 级方案与验收矩阵，再决定是否修改源码/表结构；PROD 不获准自动部署。

## 必须满足的限制与验收点

- 活动文件仍会追加写入，直接计算 Hash / 行数会产生不稳定结果。必须先做可重复的稳定快照，或定义 Byte Offset 和 File Identity 的固定读入边界。
- Hash 不是“所有行已到 Eventhouse”的证明；需 Batch 级身份、Raw 与 Parsed 的下游对账。
- 不确定是否成功的 HTTP 请求重试可能造成重复。必须明确 **At-least-once + Idempotency / Deduplication** 策略，不虚称 Exactly-once。
- 多节点隔离、按真实事件时间的首次 Backfill 截止点、QueryText / DataSource 的隐私、磁盘积压、Retention / ACL、成本都要纳入设计。
- Manifest 持久化位置、Eventhouse Schema、Uploader 更改和部署关口仍待决定；本轮**没有修改任何 Uploader 代码**。

关联：[GW-004](../tracking/ISSUES.md) · [Parser 审计](PARSER_SCHEMA_AUDIT.md) · [架构运维](ARCHITECTURE_OPERATIONS.md)。

**PROD 前置条件：** 先完成 Parser 字段准确性和连续采集可靠性验证，再审议 Manifest 的具体实现。
