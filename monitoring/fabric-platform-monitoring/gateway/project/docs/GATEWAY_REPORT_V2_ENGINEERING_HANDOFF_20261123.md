# Gateway Monitoring V2 — Engineering Candidate / 2026-11-23

> **状态：** 已完成 Enhanced PBIR 的**页面/Visual 内容候选**，并完成自动化离线结构检查；**尚未获得 Power BI Desktop / 官方 CLI 的完整验证，不是可发布的 PBIP**。本轮没有访问或修改用户的 Fabric Workspace、Gateway VM、原版 Report、Semantic Model、Uploader 或 PROD。

## 本轮实际完成

- 基于用户认可的五页交互式原型，制作可重现的 **Python PBIR 内容生成器**（在当次 ChatGPT 对话以 ZIP 交付；本次 GitHub 仓库暂仅保存工程状态和设计依据）。
- 生成 **5 个 Page、158 个 Visual 的 PBIR 内容定义**：Overview 39；Gateway Health 29；Query Performance 34；Failures & Diagnostics 28；Job Detail 28。其中 Shape 58、Textbox 48、Cards 18、Slicers 22、Line Charts 6、Tables 6。
- 页面范围 1280×720，所有 Visual 均位于画布内；字段引用严格依据版本化 Semantic Model TMDL，复用 `Execution Measures`、`Query Execution`、`System Counters`、`Gateway Node Info`、`Period`、`Calendar`。
- 重要语义更正：`Execution Measures[# Nodes]` = `DISTINCTCOUNT('Query Execution'[GatewayObjectId])`，**并不是当前 Heartbeat 在线节点数**。工程候选将其显示为 **Nodes with queries**；如需真实在线节点 KPI，须新增经实际验证的 Measure。
- 原模型 `Perc. 0.9 Query Duration (s)` 正确标记为 P90 秒，未沿用错误的 Avg/ms 标签；内存计数器采用可选择 `CounterName`，不猜测单位。敏感 `QueryText` 未被绑定到任何 Visual。
- Build 工具先调用微软 `powerbi-report-author scaffold` 创建合法 PBIP 工程外壳，再写入 Page/Visual，并调用 `powerbi-report-author validate`；未随意人工伪造未知的 PBIP Shell。也支持 `--mode assets` 完全离线生成可检查内容。
- 用户可以向 Build 工具提供**真实部署 Workspace + Model GUID**，由工具写入 `definition.pbir` 的 Live Connection（不在 GitHub 明文放置真实模型标识）。本轮没有获取该 GUID，也没有连接模型。

## 自动化检查

| 检查 | 结果 |
| --- | --- |
| Python script 编译 | **PASS** |
| 5 个页面与 158 个 Visual JSON 成功生成 | **PASS** |
| JSON 解析、唯一 Visual ID、Page ID、1280×720 边界 | **PASS** |
| 使用已检查模型字段白名单，避免将 QueryText 绑定到 Visual | **PASS** |
| ZIP 文件完整性 | **PASS** |
| Microsoft CLI `scaffold` + `validate` | **BLOCKED** — ChatGPT 环境 npm registry DNS 不通、未安装官方 CLI |
| Power BI Desktop 实际打开 / Live 数据 / Dynamic M 参数 / Drillthrough | **NOT RUN** |
| 发布 PROD / 覆盖原版 Report | **NOT PERFORMED** |

## 下一工程关口 — GW-011

在有 Power BI Desktop 的受控工作站，下载本轮对话的 `Gateway_Monitoring_V2_PBIR_Engineering_Candidate.zip`，安装/启用组织允许的 `@microsoft/powerbi-report-authoring-cli`，在**本地独立目标目录**执行项目生成器的 `--mode build`（初步可不绑定），取得官方 `validate` 结果。**本地 AI 只负责执行明确的命令并回传结果；模型字段映射与异常判断由 ChatGPT 负责。** 若 CLI 结构验证不通过，先修复候选内容，不要部署；通过后再接入真实语义模型并在 Desktop 验证 Overview。

### 已知限制

- ZIP 由本轮对话交付，不等于已进入 GitHub 源码树；若决定纳入 CI，需要单独上传生成器并核查仓库安全边界。
- `Calendar.Date` / `Period.Days Back` / Cluster / Node 的 Dynamic M 参数联动必须真实测试；静态字段引用检查不能替代运行时模型/权限验证。
- 明确**不是**一个打开即可发布的完整 PBIP；未取得真实 workspace/XMLA 信息，CLI 创建项目外壳仅可在授权工作站执行。
- Job Detail RootActivityId drillthrough 是候选实现，未经过 Power BI Desktop 导航测试。

相关：[V2 字段合同](GATEWAY_REPORT_V2_FIELD_CONTRACT.md) · [V2 设计](GATEWAY_REPORT_V2_BLUEPRINT_20261123.md) · [GW-011](../tracking/ISSUES.md)。
