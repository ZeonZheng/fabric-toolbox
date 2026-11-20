# 项目进展时间线

> 时间统一采用 **Asia/Singapore（UTC+8）**；较早的数据时间若标注 UTC 则保留原始时区。这里记录历史事件；**最新状态只看 [STATUS](../STATUS.md)**。

## 2026-11-20 10:36–10:47（UTC+8）— Eventhouse MCP Preflight 通过，进入 POC Patch 人工评审
- 本地 AI 通过 `eventhouse-remote.executeQuery` 只读执行 4/4 Preflight；三个目标 Function Body 与 2026-11-17 基线 **3/3 SHA256 精确一致**；3/3 Update Policies 均 Enabled/Transactional/Source Raw；Raw+Parsed 6/6 有数据；最近 2 小时 `0 observed` ingestion failures。**没有 Eventhouse 写操作**。
- 报告额外发现：Aggregation 6 列/Streaming 2 列新数据仍全 null；Mashup `TotalBytes` 2797/2797 null，Raw 存在该字段；Streaming 有 3 个 Raw-only 字段在目标 Parsed schema 中缺失。
- 报告对 `parse_QueryStartReport_csv` Folder 有内部矛盾，登记 GW-008，不影响本轮三个目标 Parser 的 Body 比对。
- 已归档 [脱敏报告](../evidence/2026-11-20/README.md) 和 [独立评审](../evidence/2026-11-20/PRE_DEPLOYMENT_REVIEW.md)，准备 [单个 Streaming Function 的受控 Runbook](../kql/poc-parser-fix-20261117/controlled-rollout/README.md) 与新数据/故障只读检查；**尚未执行任何 DDL**。
- 下一阶段仅建议单函数 POC 手动变更（用户确认后），并为 Mashup 增加 `TotalBytes` 等指标的补充只读验证。PROD 暂不启动。

## 2026-11-17 17:11（UTC+8）— DataSource 只读等价验证通过
- 用户执行了 `2-datasource-compatibility.kql`，导出 `export (12).csv`，已归档为 `DataSourceCheck-LegacyVsCandidate.csv`。Aggregation 96/96、Streaming 139/139，两类共 235 条 Raw 的 Legacy / Candidate DataSource 输出完全一致（`DifferentSerialized=0`）；两者动态类型集合均为 `array` 和 `string`。
- Mashup 103 条 Raw 的 `DataSource` 为 string，实际 JSON 中 `DataSourceKinds`、`DataSourcePaths` Key 均 0/103，纳入 GW-005 设计风险；**不在本轮最小 Parser 修复中强行生成字段**。
- 新增 `kql/poc-parser-fix-20261117/mcp-handoff/`：为本地 Eventhouse Remote MCP 准备 4 个只读 Preflight KQL 和中文 AI 交接指令。**尚未用该本地 MCP 实测**，无 POC Function DDL、无 PROD 变更。

## 2026-11-17 16:54（UTC+8）— 数值/单位只读验证通过
- 用户执行 `value-checks/1-parser-value-validation.kql`，导出 `export (11).csv`，已在 evidence 归档。共 **15 项：13 个数值转换指标 + 2 个 DataSource 非空率检查**，涉及 **338 条 Raw 记录**（Mashup 103 / Aggregation 96 / Streaming 139）。
- 13 个数值项共 **1,276 次非空值转换**：`ConversionLoss=0`、`ChangedByCast=0`（阈值 0.0001），Mashup 的 `TotalProcessorTime(ms)` 经 timespan 毫秒回算未发现超阈值差异。DataSource 的 235 次非空检查未发现空值损失，但**未证明内容等价**。
- 新增 DataSource 原版 bag_unpack vs Candidate 只读比较与 Mashup Raw Key 存在性检查，**尚未执行**；没有修改 POC Function、Update Policy 或 PROD。

## 2026-11-17 16:37～16:38（UTC+8）— Parser Schema 检查通过
- Aggregation 13/13、Streaming 9/9、Mashup 17/17，**39 个 IsMatch 均为 true**；已归档 CSV (8)/(9)/(10)。
- 新增只读 value-checks 查询，尚未执行；没有修改实际 Eventhouse Function。

## 2026-11-17 — 项目结构精简与中文统一
- 按单人项目精简组织：根目录仅保留总览 / 状态 / AI 交接，`tracking/` 集中 Roadmap、Issues、Decisions、Changelog；不设 Owner / Assignee。
- 核对已归档的 Eventhouse Overview **JSON + Markdown 脱敏快照**，统一项目文档为中文；保持上游代码及实际导出原文。
- KQL 三组候选仅用于只读验证；**未修改 Eventhouse、Uploader 或 PROD**。

## 2026-11-17 — Parser 数据质量验证
- 完整备份 15 个实际部署 Function Body、142 行 Schema、Update Policy 快照、MCP Overview；15/15 Function Body 与 GitHub 源码一致。
- QueryExecutionAggregation Dry-run：77 条，Query Duration 三列各 77 非空，Data Processing Duration 三列各 56 非空。
- QueryStreaming Dry-run：113 条，两个时长指标各 104 非空。
- MashupEvaluation Dry-run：91 条，五个 Memory / Processor 指标均 91 非空。
- 已生成 Schema Check KQL、Candidate 与 Rollback；**全部尚未在 POC 执行 Function DDL**。

## 2026-11-16 — Eventhouse Raw 样本
- Raw 快照包含 9 种 `logType`、647 条 JSON Message、3693 条 CSV Record；字段统计有 106 个 Type/Field 组合。不是全量数据完整性证明。

## 2026-11-13 — 实际 Gateway 日志上传
- 8 类 Report 经 `Move → Temp → Eventstream HTTP 201 → History`；Gateway 之后重新创建同名文件；定时任务切换至私有 `Config.live-poc.json` 运行。
- PROD 首次回传计划使用单独 Staging、按真实事件时间筛选；未验证高可靠性。

## 2026-11-12 及之前 — 基本功能
- Eventstream、Eventhouse、Semantic Model、Gateway Report 已安装并进行基础验证；详情在 [POC 实测记录](../docs/POC_STATUS.md)。

**维护原则：** 每次有重要测试/决策，附日期与证据链接，再更新 `STATUS`，不复制一整篇历史结论。
