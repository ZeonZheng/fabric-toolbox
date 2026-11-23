# Gateway Monitoring｜2026-11-23 PROD Readiness Gap（基于已验证 POC）

> **用途：** 明确 POC 已完成、有限 PROD Pilot 最低前置条件、可以延期的功能。用户决定**异常中断/部分 Batch 成功后的重复上传不作为当前最高优先级**，单独跟踪为后续可靠性强化；不把它当作已解决。**本文件是上线技术评审，不是 PROD 部署批准。**

## 1. 当前覆盖准确表述

- **8 类已实测进入 Eventhouse Raw：** ConcurrentOperationAggregationReport、MashupEvaluationReport、OpenConnectionReport、QueryExecutionAggregationReport、QueryExecutionReport、QueryStartReport、QueryStreamingReport、SystemCounterAggregationReport。2026-11-16 历史只读快照 Raw 有 9 类 logType（这 8 类加 GatewayNodeInfo）、647 JSON Envelope / 3693 CSV Records；Eventstream/Heartbeat 另有独立链路。**不等于源端全部记录无遗漏，也不保证每时每刻 8 类都有事件**。
- **完成 P1 必需 Parser 修复：** QueryStreaming 两项 Duration，QueryExecutionAggregation 六项 Duration，Mashup 五项 Memory/Time。已 POC 手工应用、新数据只读回归通过、同步 fork 正式 `DatabaseSchema.kql`。**这是目标字段修复通过，而非所有 CSV 字段逐值验收通过**。
- **其他类型：** QueryExecutionReport / OpenConnectionReport 的已检查 Duration 原版正确；SystemCounterAggregationReport / ConcurrentOperationAggregationReport 的样本字段结构一致但未逐值核验所有字段；QueryStartReport 允许基于 EvaluationContext 派生多行。CopyExecutionReport 已有表/Policy，但历史 Raw 无样本，不能宣称已验证该类型。
- **Raw ≠ Parsed 全字段覆盖：** QueryExecutionReport 独有 `DataReadingDuration(ms)` / `DataSerializationDuration(ms)` 不保存到 Parsed；QueryStreamingReport 独有 `StreamingDelayDuration(ms)` / `StreamingPacketsDuration(ms)` / `StreamingTrackingId` 不保存到 Parsed；Mashup 原始 DataSource/RequestId 与目标 Kinds/Paths 未对齐。Mashup `TotalBytes` / `TotalRows` / `MaxIODataBytesPerSecond` 明确 GW-009 Deferred，Parsed 仍可能 null。无证据表明当前 Report 必须消费这些字段；正式部署资产并未做完备消费审计。
- **Raw 也不是无条件字节级原样备份：** 原 `Utils.psm1` 使用 `Import-Csv` → 转 JSON；遇单条 QueryStart Message 超大时可将 `QueryText` 替换为 `QueryText too large!`，并且 Raw 默认只保留约 1 天。
- **当前 Eventstream 性能日志上传并不包含 Gateway Basic/Verbose/Additional Logs：** 这些不是当前 `*Report_*.log` Report 分类；POC `Lakehouse.UploadLogs=false`，需要单独决定是否纳入方案，不能说“所有网关日志都上传了”。

## 2. 有限 PROD Pilot 的真正最低技术关口（建议按重要性）

| 事项 | 已有证据 | Pilot 前建议动作 |
| --- | --- | --- |
| **目标报表指标够不够 / GW-005** | P1 核心指标通过；部分 Raw-only 与 Deferred 字段存在 | 用实际 PROD 监控目标/报表逐项比对需要的字段和页面；若不需要非必需字段，可接受并记录范围 |
| **持续采集与 Gateway 影响 / GW-002** | POC Task Running、8 类 Raw、History 连续更新；移动后 Gateway 会自动重建文件 | 有代表性的连续窗口内，验证各类数据新鲜度、Gateway 业务刷新正常、无不断增长的 Temp 积压、Uploader/机器 CPU/Memory/Disk 可接受；检查 Silent Data Loss 风险；不要求本轮先实现全套 Batch 幂等 |
| **真实 PROD 多节点 / GW-006** | 当前仅单节点 POC | 建立 Gateway Cluster/Node 列表、每节点独立部署与 Node ID 区分、目标权限及统一 Dashboard 视图；首轮历史 Backfill 定义真实 CSV 事件时间和 cutoff、独立 Staging，避免全量旧日志冲入 PROD |
| **数据安全 / GW-007** | QueryText/DataSource 可能包含敏感信息；当前源码保留部分文本 | 确定 Eventhouse/Lakehouse/Report RBAC、数据敏感分类、凭证保护、合规保留和访问范围，不把明文或内部 ID 写入 GitHub |
| **Raw 保留/恢复 / GW-003** | Raw 约 1 天、Parsed 3650 天 | 决定有限 Pilot 是否接受 1 天 Raw、是否需要短期延长或 Archive、Parser 错误时如何重放以及预算 |
| **运维和退出 Pilot** | POC 有任务、运行和基础报表证据 | 在 PROD 限定节点和观测周期、监测上传延迟/Task 异常/解析失败、设置停止条件、回滚/撤销步骤和明确验证结果 |

## 3. 允许延期的事项（并不等于没有风险）

- **GW-010：部分 Batch 成功后异常中断可能重复发送** — P2-T2 仅 Mock 复现了应用重发，真实 Eventhouse 没有重复记录证据；用户决定延期强化，单独追踪。**若业务要审计级精确计数、计费或自动化告警，必须重评此风险对正式使用的影响。**
- **GW-004：完整 Upload Manifest、持久化 Batch Checkpoint、稳定 RecordId、去重/Exactly-once 级别改造** — 目前候选设计未实现。若目标仅是低风险、人工监督的监控 Pilot，可以不把完整 Manifest 当绝对前置条件，但应保留失败告警、范围限制和缺口声明；不可声称端到端零丢失/无重复。
- **GW-009：Mashup 三个非必需字段** — 业务已决定延期。
- **Gateway Basic/Additional Logs 采集** — 当前方案不覆盖，是否需要由正式目标决定；避免范围蔓延。

## 4. 阶段判断

**P0 功能 POC Done；P1 本轮目标 Parser 修复 Done；下一重点为 P2/P3 合并推进的 PROD Pilot Readiness：字段范围确认 + 持续采集/无静默丢失的合理验证 + 节点扩展/Backfill + 安全/Retention + 运行监测/回滚。P4 Limited PROD Pilot 尚未批准。**

不因“已修三个 Parser”推导“所有字段已在 Parsed”，也不因“Mock 重发风险存在”将整个项目无限期卡在幂等设计上。