# 项目进展时间线

> 时间统一采用 **Asia/Singapore（UTC+8）**；较早的数据时间若标注 UTC 则保留原始时区。这里记录历史事件；**最新状态只看 [STATUS](../STATUS.md)**。

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
