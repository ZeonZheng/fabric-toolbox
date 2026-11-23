# Mashup Parser POC 功能回归｜2026-11-23 11:06–11:08 SGT

> **来源：** 用户上传本地 AI / `eventhouse-remote` 的只读执行报告 `gateway-mashup-postcheck-pass-20261123-1105-SGT.md`。本记录为**独立审阅用户提供的查询摘要**；本助手未直接运行 Eventhouse 查询。不存在本次新增 DDL、Policy、Uploader 或 PROD 修改。

## 配置与执行

- **目标：** Gateway Monitoring POC / 单节点。
- **实际 PatchUtc：** `2026-11-23T02:39:15.4152749Z`。
- **验证窗口：** UTC `2026-11-23T03:06–03:08`（SGT `11:06–11:08`）。
- **已执行：** 13 Activity Gate → `READY_FOR_POSTCHECK` → 10 Summary / 11 KeyCheck / 12 Ingestion Failures；无查询错误，仅替换 PatchUtc。
- **Function / Policy：** 上轮 10:45 SGT 核实 Live Function 与 P1 Candidate Body 逐字节一致、SHA256 `7f333c39791810d52531b3340cbbce0ffb998d2925f0f0a1d594edf8e82813`；Policy Enabled、Raw source、Transactional。本轮**没有重新取 SHA**，新 Raw→Parsed 功能验证与先前配置检查一致。

## 本轮实际结果

| 项目 | 结果 |
| --- | --- |
| Gate 13 | **2 Mashup Raw Envelope / 4 内嵌 Records**；其他 Report Envelope 18；Gate=`READY_FOR_POSTCHECK` |
| Query 10 / Raw | **4 行**，五个 Memory/Time 必需字段均 **4/4 非空** |
| Query 10 / Parsed | **4 行**，相同五项均 **4/4 非空** |
| Query 11 | RawRows=4、ParsedRows=4、DifferingGroups=0、UnmatchedRows=0、**PASS** |
| Query 12 | `OriginatesFromUpdatePolicy=true` 的 Patch 后失败查询**无记录（0 observed）** |

**独立验收：PASS（限定当前样本的 Mashup Parser POC 功能正确性）。** 签名多重集包含业务标识 + 五项目标值，可排除在本轮 4 行中存在不同数值而仅“非空计数一样”的情况。Streaming 与 Aggregation 已有独立的 POC 功能通过记录，因此 **GW-001 的当前三个目标 Parser 功能修复均已完成**。

## 严格限制

1. 仅 **4 条 Mashup 记录、2 个 Envelope**，不是长期稳定性、全部 Gateway CSV 字段或任何环境的通用保证。
2. Raw `logDate` 与 Parsed `ingestion_time()` 使用不同的时间语义；本次对账**不构成文件级端到端完整性/Exactly-once**；Collector 可靠性单独在 GW-002/P2 验证。
3. 三个非必需指标 `TotalBytes`、`TotalRows`、`MaxIODataBytesPerSecond` 明确留在 **GW-009 Deferred**，不会被当作已修复；其它 Schema 覆盖评估仍归 GW-005。
4. 历史 Parsed null 不自动回填；Raw 只保留 1 天，相关 Retention/Replay 风险保留在 GW-003。
5. 本轮只读报告没有重新返回 Function Body SHA；此前 10:45 的版本校验与当前无失败的功能行为组合起来满足既定 POC Runbook Gate，但不是 11:08 时再次逐字节确认。

## 工程同步

- 已将**经 POC 验证的 Mashup Function 逻辑**同步至 fork 正式 `src/Gateway/Gateway Monitoring.KQLDatabase/DatabaseSchema.kql`，仅替换这一段函数，未改 Parsed Table Schema / Update Policy / 其它 Parser，见 Git commit `f430448595e65fae890a142969bfcea26ce4eea2`。
- 历史 Blocked Candidate 和 POC-only Candidate 均保留，方便追溯范围变化（ADR-009）和版本回滚。
- **P1 Data Correctness（已明确列入验收范围的三个 Parser）完成功能性 POC 验证；进入 P2 Collector Reliability。** 与 P1 源码修复不同，P2 不能只验证 Parsed 同 Raw 一致；必须研究源文件、批次重试、轮转、重复/遗漏和长期稳定性。

## 下一步：P2 的第一步

开展**只读 Collector Reliability Baseline**：先审计现有 Uploader 的文件扫描/Move/Temp/Batch/History/Retry 行为与可用元数据，结合 POC Gateway VM（如能访问）及 Eventhouse（如能访问）建立对账可行性分析。**严禁自动注入故障、改 Task Scheduler、移动或修改活动日志、修改 PROD。**

相关：[STATUS](../../STATUS.md) · [ROADMAP](../../tracking/ROADMAP.md) · [ISSUES](../../tracking/ISSUES.md)。
