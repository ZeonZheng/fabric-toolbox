# 本地 AI / Eventhouse Remote MCP｜Aggregation 部署后只读回归

GitHub 仓库：https://github.com/ZeonZheng/fabric-toolbox  
**分支**：`docs/gateway-monitoring-poc-20261112`（不是 main）  
**当前目录**：https://github.com/ZeonZheng/fabric-toolbox/tree/docs/gateway-monitoring-poc-20261112/monitoring/fabric-platform-monitoring/gateway/project/kql/poc-parser-fix-20261117/controlled-rollout

仅在用户完成 **POC** 中 `parse_QueryExecutionAggregationReport` 的手动 DDL 后运行。用户必须提供实际 `RecordedAfterPatchUTC` 和 POC 人工确认。无准确时间或确认即 `Inconclusive`，不得猜。

**只读操作（3 个查询，逐个运行）：**
- [06-aggregation-postdeploy-summary.kql](06-aggregation-postdeploy-summary.kql)；
- [07-aggregation-postdeploy-keycheck.kql](07-aggregation-postdeploy-keycheck.kql)；
- [08-aggregation-ingestion-failures.kql](08-aggregation-ingestion-failures.kql)。

将文件中的示例 `2099-01-01T00:00:00Z` 替换为用户实际 UTC。额外通过 `.show functions` 取得现场 Body，与 [GitHub Candidate](https://github.com/ZeonZheng/fabric-toolbox/blob/docs/gateway-monitoring-poc-20261112/monitoring/fabric-platform-monitoring/gateway/project/kql/poc-parser-fix-20261117/candidates/1-parse_QueryExecutionAggregationReport.kql) 做同一规范化规则的内容比较（不得只靠新旧 Hash 不同）；检查 Policy Enabled / Source Raw / Transactional。

**输出：** 中文 Markdown `gateway-aggregation-postcheck-YYYYMMDD-HHmm-SGT.md`，必须提供真实文件路径或完整文本供用户上传 ChatGPT。内容包含：POC 身份与真实执行时间、MCP 工具、BodyMatchesCandidate、Update Policy、Raw/Parsed 两层新行数与六字段逐项非空率、复合 Key 样本数与逐字段 Matches/Mismatches、Post-Patch ingestion failures、`Passed/Blocked/Inconclusive`、剩余限制和脱敏结果表。

**特别注意：** Raw 数据中 Data Processing Duration 允许有真实空值；应只对 Raw 有值的记录要求 Parsed 值匹配。组合 Key 可能重复，不能据此声称 Exactly-once。Raw `logDate` 不等于 Parsed `AggregationEndTimeUTC`。无新事件为 `Inconclusive`。任何相关摄入失败、值错误或 Function 不一致为 `Blocked`。

**禁止**执行 DDL、回滚、修改 Uploader/Update Policy、操作 PROD；不得泄露 DataSource、QueryText、原始 IDs/日志/Token。
