# 本地 AI / Eventhouse Remote MCP｜Mashup 数组与数值语义诊断（只读）

**状态：** 2026-11-20 16:27–16:44 Precheck = **Blocked**。旧版 `parse_MashupEvaluationReport` 仍在运行；**禁止部署现有 Candidate 或改动目标表 Schema**。

## GitHub 精确来源

- 仓库：https://github.com/ZeonZheng/fabric-toolbox
- **指定分支（不是 main）：** `docs/gateway-monitoring-poc-20261112`
- 原始阻断报告及技术复核：`monitoring/fabric-platform-monitoring/gateway/project/evidence/2026-11-20/`
- 当前状态：https://github.com/ZeonZheng/fabric-toolbox/blob/docs/gateway-monitoring-poc-20261112/monitoring/fabric-platform-monitoring/gateway/project/kql/poc-parser-fix-20261117/value-checks/../../../STATUS.md
- **本轮三份只读 KQL：**
  1. https://github.com/ZeonZheng/fabric-toolbox/blob/docs/gateway-monitoring-poc-20261112/monitoring/fabric-platform-monitoring/gateway/project/kql/poc-parser-fix-20261117/value-checks/4-mashup-array-shape.kql
  2. https://github.com/ZeonZheng/fabric-toolbox/blob/docs/gateway-monitoring-poc-20261112/monitoring/fabric-platform-monitoring/gateway/project/kql/poc-parser-fix-20261117/value-checks/5-mashup-array-length-distribution.kql
  3. https://github.com/ZeonZheng/fabric-toolbox/blob/docs/gateway-monitoring-poc-20261112/monitoring/fabric-platform-monitoring/gateway/project/kql/poc-parser-fix-20261117/value-checks/6-mashup-array-element-validation.kql
- 原版/候选 Function（仅读取，不执行）：https://github.com/ZeonZheng/fabric-toolbox/blob/docs/gateway-monitoring-poc-20261112/monitoring/fabric-platform-monitoring/gateway/project/kql/poc-parser-fix-20261117/candidates/3-parse_MashupEvaluationReport.kql

## 执行要求

1. 确认 `eventhouse-remote` 指向**用户已确认的 POC Gateway Monitoring KQL Database**。不得对 PROD 发出任何调用。
2. 逐个读取并执行上述 3 份 **只读** KQL，记录 Query 成功/失败、原始脱敏聚合结果和实际查询时刻（UTC+8）；如果语法报错，可以在本地修正只读查询并附上修改前后差异与原因，但**不要擅自修改 GitHub 文件**，不要修改 Eventhouse。
3. 重点核实：TotalBytes/TotalRows 单元素与多元素的数量及元素可转类型；数组长度之间是否存在匹配关系；MaxIO 是否含小数、负数和非数值异常；**不要输出任何真实数组数值、DataSource、QueryText、ConnectionId 或用户/组织信息**。
4. 从现有 POC Report / Semantic Model / Gateway 源文件解析逻辑（若本地有已授权访问）**只读检查**是否存在对 TotalBytes / TotalRows 的业务定义或数组展开规则。引用原文所在路径/文件/列名即可，不要上传可能敏感的原始 CSV；若无法确认，则标记 `SemanticMeaning=Unverified`。**不要因为两个数组长度相等就自动假定可以求和、取首元素或按索引关联。**
5. **不自动修改候选函数**；不自动决定 MaxIO 的小数取整规则和 `long→real` 目标 Schema 变化。只提出带证据的可选路径及风险。

## Output Contract

请生成完整中文 Markdown 文件 `gateway-mashup-shape-followup-YYYYMMDD-HHmm-SGT.md`，提供真实文件路径或全文交给用户上传 ChatGPT。包含：

- A. POC、真实查询时间、MCP 工具、GitHub 分支/Commit、三份 KQL 状态；
- B. 两字段数组单/多元素总数、长度分布与共同出现数量；
- C. 数组内元素 Numeric/LongConvertible/Fractional/Negative/large-integer 计数；
- D. MaxIO 小数样本数量、整数部分/非数值样本数量、现有 `long` Schema 的精度影响（**不要求输出实际值**）；
- E. 如能确认的数据来源或报表/字段消费关系，及 `SemanticMeaning=Verified/Unverified` 的依据；
- F. `Blocked / Ready for candidate redesign discussion / Inconclusive`，哪些问题仍需要方案选择、哪些可以保持原列、尚缺什么证据。

**禁止：** Function DDL、Rollback、修改 Update Policy/表 Schema、Uploader/Task 修改、对 PROD 操作。即使所有数值可转换，也不得直接批准 Mashup Candidate 部署。
