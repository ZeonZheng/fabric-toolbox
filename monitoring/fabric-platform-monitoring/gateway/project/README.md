# Gateway Monitoring｜项目入口

> **范围：** Microsoft Fabric Platform Monitoring / Gateway Monitoring；单节点 POC → 生产可行性验证。  
> **当前阶段：** P1 — 数据解析准确性验证。**PROD 尚未获准部署。**  
> **最后更新：** 2026-11-20（Asia/Singapore）  
> **工作分支：** `docs/gateway-monitoring-poc-20261112`（`main` 不修改）

## 30 秒了解项目

已验证 Gateway VM → Eventstream → Eventhouse → Semantic Model → Power BI 的功能链路。真实日志的持续上传正在观察，**尚未证明不丢失、不重复**。发现上游 KQL Parser 与实际 Gateway CSV 字段名不匹配，导致部分 Parsed Table 指标全为 null；3 组只读修复候选已恢复非空，**Schema 39/39 列已通过**、数值/时间单位 13 项当前样本无超阈值差异；DataSource 235/235 条一致。2026-11-20 本地 MCP Preflight **4/4 已通过**（3/3 Function 严格一致），2026-11-20 用户已在 POC **手工应用 Streaming Candidate**；14:22 首轮因无新样本为 Inconclusive；**14:48 Follow-up 已通过：32/32 新 Parsed 行两项 Duration 恢复且 Raw/Parsed 值完全一致**。Streaming 的 GitHub 正式 Schema 已同步；Aggregation 2026-11-20 15:24 已手工在 POC 部署，但因首轮没有新数据而 Inconclusive；Mashup 尚未部署。长期采集可靠性仍未验收。详见 [Streaming 验证报告](evidence/2026-11-20/STREAMING_FOLLOWUP_REVIEW.md)。

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
