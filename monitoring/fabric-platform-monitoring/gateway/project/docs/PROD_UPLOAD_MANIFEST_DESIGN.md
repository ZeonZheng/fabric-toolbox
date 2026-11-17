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

## 必须满足的限制与验收点

- 活动文件仍会追加写入，直接计算 Hash / 行数会产生不稳定结果。必须先做可重复的稳定快照，或定义 Byte Offset 和 File Identity 的固定读入边界。
- Hash 不是“所有行已到 Eventhouse”的证明；需 Batch 级身份、Raw 与 Parsed 的下游对账。
- 不确定是否成功的 HTTP 请求重试可能造成重复。必须明确 **At-least-once + Idempotency / Deduplication** 策略，不虚称 Exactly-once。
- 多节点隔离、按真实事件时间的首次 Backfill 截止点、QueryText / DataSource 的隐私、磁盘积压、Retention / ACL、成本都要纳入设计。
- Manifest 持久化位置、Eventhouse Schema、Uploader 更改和部署关口仍待决定；本轮**没有修改任何 Uploader 代码**。

关联：[GW-004](../tracking/ISSUES.md) · [Parser 审计](PARSER_SCHEMA_AUDIT.md) · [架构运维](ARCHITECTURE_OPERATIONS.md)。

**PROD 前置条件：** 先完成 Parser 字段准确性和连续采集可靠性验证，再审议 Manifest 的具体实现。
