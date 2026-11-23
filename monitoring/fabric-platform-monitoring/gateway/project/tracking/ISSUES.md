# 问题与风险台账

> 更新：2026-11-23（Asia/Singapore）｜当前由我们维护，**不设责任人字段**。一个问题使用固定 ID；详细证据留在 `docs/`、`evidence/`，此处只记录状态与下一动作。

| ID | 优先级 | 状态 | 首次记录 | 最后复核 | 问题 / 风险 | 下一动作 | 证据 |
| --- | --- | --- | --- | --- | --- | --- | --- |
| **GW-001** | P1 | **Closed — POC 必需 Parser 功能性验收 Passed（2026-11-23）** | 2026-11-17 | 2026-11-23 | Streaming 32/32 Duration、Aggregation Raw/Parsed 各 7 行（复合 Key 2/2、六项指标 0 mismatch）、Mashup 4/4 五项指标 Raw/Parsed 签名 0 mismatch；三项已部署且功能性通过。**非全字段/历史回填/Collector 无丢失保证** | 本轮范围已闭环；后续 Collector 完整性归 GW-002、Retention/Replay 归 GW-003、其余字段覆盖归 GW-005、三个不必需指标归 GW-009；无需再重复同类 Parser 测试 | [Mashup PASS](../evidence/2026-11-23/MASHUP_POSTCHECK_PASS_1105_REVIEW.md) · [Streaming](../evidence/2026-11-20/STREAMING_FOLLOWUP_REVIEW.md) · [Aggregation](../evidence/2026-11-20/AGGREGATION_FOLLOWUP_REVIEW.md) |
| **GW-002** | P1 | **P2 进行中 — T1 同路径 Move 测试通过，Batch Retry 待核查** | 2026-11-12 | 2026-11-23 | VM 目录递归扫描 Source/Temp 无残留，Task Action 与 live-poc 匹配。隔离 P2-T1：PowerShell 7.6.10 `Move-Item` 同源同目的 1 次成功、FileReady=True、文件仍存在，**此前“同路径必失败”假设被实测否定**。源码仍缺持久化 Batch Checkpoint/去重；非同路径原因的 Move 失败后 Merge 路径仍需另评估 | 下一步仅**只读确认 Live POC 配置**中 `ConnectionProperties` / `MaximumRetryCount` / `RetryIntervalSec`；之后由 ChatGPT 设计无网络的 Batch 部分失败模拟，不直接运行生产 Uploader | [P2-T1 证据](../evidence/2026-11-23/GW002_T1_SAME_PATH_MOVE_RESULT.md) · [独立源码评审](../evidence/2026-11-23/UPLOADER_SOURCE_RELIABILITY_REVIEW.md) |
| **GW-003** | P1 | 待决策 | 2026-11-17 | 2026-11-17 | Raw 仅保留 1 天，回放及长期审计受限 | 评估 Retention / Archive 成本与恢复策略 | [Eventhouse Overview](../evidence/2026-11-17/EventhouseOverview.MCP.sanitized.md) |
| **GW-004** | P2 | 设计中 | 2026-11-17 | 2026-11-17 | PROD 需 Upload Manifest 追踪 File、Hash、Rows、Batch Status | 确定快照、幂等与持久化设计 | [设计草案](../docs/PROD_UPLOAD_MANIFEST_DESIGN.md) |
| **GW-005** | P2 | **Schema 覆盖缺口待评估** | 2026-11-17 | 2026-11-23 | Streaming Raw 三字段未投影；Mashup Kinds/Paths 原 Raw 无对应 Key。Mashup 三个非必需指标的类型/语义冲突独立转 GW-009 | 结合实际业务消费决定其余 Schema 覆盖要求；不为无源 Key 或非必需指标强制变更 Schema | [Nov23 Review](../evidence/2026-11-23/MASHUP_SHAPE_FOLLOWUP_REVIEW.md) · [决策](DECISIONS.md) |
| **GW-009** | P3 | **Deferred / 当前非阻塞** | 2026-11-23 | 2026-11-23 | `TotalBytes`、`TotalRows` 为 JSON 多元素数组，当前标量转换全 null；`MaxIODataBytesPerSecond` 含小数，与 long 类型不兼容。三个指标当前 POC/计划中 PROD 监控非必需，保留数据完整性缺口，不视为修复 | 暂不解析/修改/联系微软；仅当明确有下游需求时复核实际消费、数组元素语义、精度、Schema 和迁移方案 | [决策 ADR-009](DECISIONS.md) · [设计选项](../docs/MASHUP_DESIGN_OPTIONS.md) · [证据](../evidence/2026-11-23/MASHUP_SHAPE_FOLLOWUP_REVIEW.md) |
| **GW-006** | P2 | 待验证 | 2026-11-13 | 2026-11-17 | 多节点隔离与 PROD 首轮 Backfill 边界 | 按事件时间和独立 Staging 规划 | [架构运维](../docs/ARCHITECTURE_OPERATIONS.md) |
| **GW-007** | P2 | 待验证 | 2026-11-17 | 2026-11-17 | QueryText、DataSource、Secrets 与安全审计 | PROD 之前开展权限与隐私评估 | [架构运维](../docs/ARCHITECTURE_OPERATIONS.md) |
| **GW-008** | P3 | **待复核** | 2026-11-17 | 2026-11-20 | CopyExecutionReport 无源样本；部分 Parser DocString；2026-11-20 Preflight 报告称 `parse_QueryStartReport_csv` Folder 有差异，但另有全库 Folder 一致的相反表述 | 单独用实际 `.show functions` 复核该元数据差异，不影响本次目标三个 Parser 的 Body | [Preflight](../evidence/2026-11-20/PRE_DEPLOYMENT_REVIEW.md) |

**状态定义：** 验证中 = 已有候选方案但未最终通过；待验证 = 尚无合格的实测结果；待决策 = 需要权衡方案；设计中 = 尚未实施；已关闭 = 附日期、验证证据及结果。不要根据“脚本执行成功”提前关闭问题。

如将来真正使用 GitHub Issues，只保留一个权威状态来源，本文件改为索引，避免双重维护。
