# 项目进展时间线

> 时间统一采用 **Asia/Singapore（UTC+8）**；较早的数据时间若标注 UTC 则保留原始时区。这里记录历史事件；**最新状态只看 [STATUS](../STATUS.md)**。

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
