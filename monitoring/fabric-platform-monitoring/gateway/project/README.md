# Gateway Monitoring｜项目入口

> **范围：** Microsoft Fabric Platform Monitoring / Gateway Monitoring；单节点 POC → 生产可行性验证。  
> **当前阶段：** P1 — 数据解析准确性验证。**PROD 尚未获准部署。**  
> **最后更新：** 2026-11-23（Asia/Singapore）  
> **工作分支：** `docs/gateway-monitoring-poc-20261112`（`main` 不修改）

## 30 秒了解项目

- **P0 Functional POC 已完成：** Gateway VM → PowerShell Uploader → Eventstream → Eventhouse Raw / Parsed → Semantic Model → Power BI Report 的单节点基础功能已跑通；已观察到 8 类 Gateway Performance Reports、Heartbeat、NodeInfo。**上传端的无丢失、无重复和故障恢复尚未验证。**
- **P1 Data Correctness 正在收尾：** Streaming Parser 与 Aggregation Parser 已部署 POC 并通过新数据回归（分别 32 条与 7 条样本）；两者已同步 fork 正式 KQL。Mashup P1 最小 Candidate **2026-11-23 已在 POC 手工部署**，现场 Body 与 Candidate 匹配、Policy Enabled/Transactional，但首轮 Postcheck 的新 Raw/Parsed **均为 0**，结论 **INCONCLUSIVE**，未获新记录功能验收。详见 [STATUS](STATUS.md) 与 [Postcheck 独立评审](evidence/2026-11-23/MASHUP_POSTCHECK_1045_REVIEW.md)。
- **已缩小 P1 范围：** `TotalBytes`、`TotalRows`、`MaxIODataBytesPerSecond` 不是当前必需指标；[ADR-009](tracking/DECISIONS.md) 已将其转为 [GW-009 Deferred](tracking/ISSUES.md)，不声称数据已修复。
- **P2 Collector Reliability 是下一重点：** 源文件、Batch、Raw、Parsed 的完整性对账，活动文件 Move/轮转、失败重试与恢复、长时间连续运行。P3 PROD Architecture 规划中，P4 Limited PROD Pilot **尚未批准**。
- **唯一下一步：** 有真实新 Mashup Raw 后才复跑 Postcheck 10/11/12；先用 [Activity Gate 13](kql/poc-parser-fix-20261117/controlled-rollout/13-mashup-postdeploy-activity-gate.kql) 判断。可以并行准备 P2 的可靠性测试。**PROD 尚未获准部署。**

## 快速导航（以此处为准）

| 问题 | 唯一权威入口 |
| --- | --- |
| **现在做到哪一步？我要做什么？** | [STATUS.md](STATUS.md) |
| 问题、风险与状态 | [tracking/ISSUES.md](tracking/ISSUES.md) |
| Phase 与进入 PROD 的验收条件 | [tracking/ROADMAP.md](tracking/ROADMAP.md) |
| 已决定事项与设计取舍 | [tracking/DECISIONS.md](tracking/DECISIONS.md) |
| 按日期追踪本次及历史进展 | [tracking/CHANGELOG.md](tracking/CHANGELOG.md) |
| 人员 / AI 接手需要的固定背景 | [AI_CONTEXT.md](AI_CONTEXT.md) |
| 架构、部署、排障等技术文档 | [docs/INDEX.md](docs/INDEX.md) |
| 实测证据、Eventhouse Overview、Schema 备份 | [evidence/README.md](evidence/README.md) |
| KQL Dry-run / Schema Check / Candidate / Rollback | [kql/poc-parser-fix-20261117/README.md](kql/poc-parser-fix-20261117/README.md) |
| 我们适配的 Notebook | [notebooks/](notebooks/) |

## 目录边界：我们的工作 vs 上游源码

- **我们的项目资产仅放在此 `project/` 下**：状态、问题、架构记录、验收、KQL 候选、证据、适配 Notebook。
- **上游项目保留原路径**：[`PowerShellScript/`](../PowerShellScript/)、[`TaskSchedulers/`](../TaskSchedulers/)、[`PBI Report/`](../PBI%20Report/)、[`src/Gateway/`](../../src/Gateway/)。
- 上游脚本已有一项在 fork 中验证过的修改：Heartbeat `$body → $msg`；见 [差异清单](docs/UPSTREAM_CHANGES.md)。
- **GitHub 是工程资产与决策的 Source of Truth**；实际 Eventhouse 变更必须同步回 Git。Raw 样本、QueryText、Credentials、SAS Keys 和环境专属敏感数据不进入仓库。
- 此项目只需维护我们两人协作的记录；**不设置 Owner / Assignee、RACI、无意义的审批角色字段**。如未来加入多人再按需引入。

**阅读原则：** `STATUS` 只写现状和唯一下一步；`tracking` 写状态/决策/历史；`docs` 记录技术过程；`evidence` 只放有时间和来源的证据。避免一段信息在多个文件反复维护。

[返回 Gateway 目录](../README.md)
