# Gateway Monitoring Report V2 — 字段消费审计与 UI 设计初稿

> **依据：** 2026-11-23 直接读取 fork `src/Gateway/Gateway Monitoring.Report/report.json`、`src/Gateway/Gateway Monitoring.SemanticModel/definition/{tables,expressions}` TMDL 与 `src/Gateway/Gateway Monitoring.KQLDatabase/DatabaseSchema.kql`。**只代表 GitHub 版本化资产，尚未连接 Live Semantic Model、渲染报表或测试任何新 Visual。** 当前只提交设计与审计，不覆盖已部署原版，不修改 PROD。

## 1. 源码审计：现有 Report

Report 共 **6 页 / 58 Visual**：

| 页面 | Visual 数 |
| --- | ---: |
| Gateways | 2 |
| System Counters | 10 |
| Jobs | 11 |
| Queries | 18 |
| Running Jobs | 6 |
| Job Details | 11 |

- 现有视觉结构以多个 Slicer、Matrix、Line Chart 和 Table 为主；Queries 单页 18 Visual，在 1280×720 画布上信息密集。用户反馈视觉设计不理想。
- 原版 Report 实际 Semantic Model 主来源只有四条 KQL 路径：
  - `Gateway Node Info` ← `GatewayNodeStatusWithInfo()` ← `GatewayNodeInfo` / `GatewayNodeStatus`（NodeInfo + Heartbeat）
  - `System Counters` ← `SystemCounters()` ← `SystemCounterAggregationReport`
  - `Query Execution` ← `QueryExecutionUnified()` ← `QueryStartReport` + `QueryExecutionReport` + GatewayNodeInfo
  - `Query Connections` ← `QueryConnections()` ← `QueryStartReport` 的 DataSource
- **目前原版 Semantic Model 不直接消费** `QueryStreamingReport`、`QueryExecutionAggregationReport`、`MashupEvaluationReport`。因此 P1 已验证的三项 Parser 修复不是原版六页的直接输入。未来 V2 若添加这三类图表，必须先在模型中新增已定义的 KQL Query / Tables / Measures，并测试。
- 原版模型通过 DirectQuery + 动态 M 参数 `DateToSearch`、`DaysBack`、`SelectedGateways`、`SelectedClusters` 查询 KQL；改版必须保留有效的日期/节点/集群筛选及其参数绑定，不能简单删除 Slicer。
- 现有 `Execution Measures` 中两项名称 `Query Avg. Execution - DirectQuery (ms)` 和 `Query Avg. Execution - Refresh (ms)`，DAX 实际调用 `Perc. 0.9 Query Duration (s)`，返回 **P90 秒**，并不是平均值/毫秒。V2 应修正显示名称与单位，或者新增语义一致的 Measures。不得继续使用误导性标签。
- `# Queries (running)` 由总数减 Success 减 Error 得到；它是**状态差额推断**而非事件表明确给出的实时运行 Query 状态，不宜当作无需免责声明的真实 Active Running 数量。

## 2. 建议的 V2 Information Architecture

**V2 目标：为 Gateway Operations 用户提供“先概览，后定位，再钻取”监控工作流。暂定 5 页，不一次性新增全部数据层。**

| V2 页面 | 目的 | 首版可复用来源 | 需要新增/注意 |
| --- | --- | --- | --- |
| **01 Overview** | 一屏判断 Gateway / Query 是否健康 | Gateway Node Info、Execution Measures、Query Execution | Responding 节点、# Jobs、# Queries、# Queries errors、P90 Duration；Error Rate 若需新增 DAX |
| **02 Gateway Health** | 定位 Cluster/Node CPU/Memory 问题 | System Counters、Gateway Node Info | 节点过滤、网关 CPU 需要按现有 `Avg. Counter Value` 逻辑处理 Logical Cores |
| **03 Query Performance** | 分析查询量、P90/最大 Duration、Refresh vs DirectQuery | Query Execution、Execution Measures | 修正原版两个误标的 Average/(ms) 名称；避免把 P90 叫 Average |
| **04 Failures & Diagnostics** | 查看失败查询、错误类型、Service / DataSource 与时间分布 | Query Execution、Query Connections | 防止在共享视觉中直接暴露 QueryText / DataSourcePaths、WorkspaceId / ItemId |
| **05 Job Detail (Drillthrough)** | 从异常 Job/Query 钻取细节 | Query Execution、Query Connections | RootActivityId / QueryTrackingId 用作 Drillthrough、按权限显示详情 |

**设计策略：** V2 第 1 阶段只使用现有已版本化模型功能；Mashup、Streaming、QueryExecutionAggregation 扩展作为**后续选项**，不应阻塞界面改版，也不能假装这些表当前模型已经有。

## 3. 页面设计原则（可实施）

- 页面比例 16:9，统一最上层 Navigation + Time Window / Cluster / Node 筛选；不要每页重复大量散落 Slicer。
- 左侧紧凑页签导航或顶部主导航（二者择一，保持所有页一致）。
- Overview：顶部 4–6 个 KPI，中部 2 个清晰主趋势，底部异常/节点摘要。不要在概览放原始 GUID / QueryText。
- Gateway Health：以 Node 维度分组 CPU、Memory 时序；下方配简洁节点表，明确 CounterName 的单位和条件格式。
- Query Performance：Query Count / Errors / P90 Query Duration 与业务趋势；OperationType 分组、Top Slow Queries 表，支持点击钻取。
- Failures & Diagnostics：优先列示失败次数、失败率（需新 measure）、最近失败 / ErrorMessage（审查敏感信息），避免堆叠不必要 Donut/Pie。
- Job Detail：置于 Drillthrough，保留工作区/Item/Trace ID 的运维价值，但不作为首页大量展示。
- 中性浅背景 + 深色文本 + 少量蓝色强调；只对状态使用绿/橙/红。统一边距、字号、对齐与空值显示；不预填/伪造实际环境数值。

## 4. 字段消费与 PROD 解析风险评审

**原版报表的重要数据链路**是 NodeInfo / Heartbeat / SystemCounter / QueryStart / QueryExecution。用户此前 P1 修复的三类 Parser 通过了预期关键字段验收，但**不代表所有字段已写入 Parsed**，也不代表当前六页报表用到了这些修复。详细缺口见 [字段映射审计](SCHEMA_FIELD_MAPPING_AUDIT.md) 与 [PROD Readiness Gap](PROD_READINESS_GAP_20261123.md)。

建议按 V2 页面实际字段对照：
1. 是否在 Semantic Model/TMDL 中存在；该列/Measure 是哪一 KQL Function 输出？
2. 源 Parsed 列是否存在、数据类型/单位是否正确？若对应新修复字段，检查 POC 后新数据值与 Raw。
3. 真实 Live 模型是否和 GitHub 版本相同？DAX/M 参数、模型关系及安全权限有无漂移？
4. 用户可见的 KPI 是否有时间语义和可解释定义（尤其 Last Status、Running Jobs、Average vs P90）？

## 5. 实现格式与 Live Model 边界

- **当前源码的 `Gateway Monitoring.Report/report.json` 是 PBIR-Legacy**；另有 `gateway/PBI Report/Gateway Monitor.pbit`，二者并非同一份可直接编辑源码。旧 `report.json` 不建议直接手工修改以制作新视觉，Microsoft 对外部编程编辑的文档化支持主要面向 [Enhanced PBIR](https://learn.microsoft.com/en-us/power-bi/developer/embedded/projects-enhanced-report-format)；原版文件不能证明 Live Report 与 GitHub 版本一致。
- `definition.pbir` 的 `datasetReference.byConnection` 包含模板/占位的 `00000000-0000-0000-0000-000000000000`，不能据此认定可以直接连接当前真实模型。
- **V2 实施路径：** 保留 Live Semantic Model，创建**独立的 Report（thin/live-connected report）**，在 Power BI Desktop 中保存为 **PBIP + Enhanced PBIR**，之后可按公开 Schema 进行页面/Visual 文件级优化。Microsoft 文档：[连接已有 Semantic Model](https://learn.microsoft.com/en-us/power-bi/connect-data/desktop-report-lifecycle-datasets) 与 [PBIR 文件结构](https://learn.microsoft.com/en-us/power-bi/developer/projects/projects-report)。
- **连接 Live Model 的正确时机：** 版本化源码审计和页面设计已可离线完成；实际实施/测试 Report 时才需用具备 Build 权限的用户连接**真正部署的 Model**，核实可能的版本漂移、DateToSearch 等动态 M 参数、KQL 查询行为、Measure 输出和 RLS/敏感数据控制。
- 本设计仅是设计文件，**还没有生成可打开的 V2 PBIP/PBIR**；不得向用户声称 Report 已构建或已发布。

## 6. 具体交付与验收顺序（避免大范围重做）

- **Step 1 — GitHub 源码审计：已开始。** 本文已检查页面、四个 KQL Source 路径和主要 Measures；下一步逐个细化 V2 必需指标清单。
- **Step 2 — UI 方案：** 新建独立 `Gateway Monitoring V2` Report，第一版优先复用现有模型及真实 Measures；必要时只新增少量正确命名的 DAX。
- **Step 3 — Live Check：** 只有验证最终数据和交互时才使用 Power BI MCP / Semantic Model 权限检查或在 Desktop 中连接；确认模型 ID、模型字段、DirectQuery 参数、查询响应，不要求本地 AI 做技术决策。
- **Step 4 — 功能验收：** 5 页均可加载、关键 KPI 与原版/原始 KQL 计数一致、Gateway/Node/Date filters 正常、Drillthrough 正常、敏感信息不意外暴露；Report 布局可读且快于/不显著劣于原版（需实际测量）。
- **Step 5 — 是否上线：** V2 使用独立版本和部署，原 Report 留作对照和回滚；未经用户授权不得覆盖原版、发布 PROD 或更改模型。

**核心边界：先把新版 Report 设计好，再决定是否扩展 Semantic Model。界面改版不应该强迫现在重做全部 Parser、Uploader 或上传管道。**
