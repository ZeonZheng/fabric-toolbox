# Mashup Array / Decimal 补充诊断（只读）

**背景：** 2026-11-20 Mashup Precheck 发现原 Candidate 的 `TotalBytes`（243/243）、`TotalRows`（34/34）、`MaxIODataBytesPerSecond`（127/127）均发生 100% 的 Raw 非空→Candidate null 损失。旧版 Parsed 同样存在相关 null，**这不是已经部署的回归，而是尚未修复的存量缺陷**。

使用 [本地 MCP 完整任务说明](MASHUP_SHAPE_FOLLOWUP_AI_PROMPT.md) 执行下列查询：

| KQL | 只读检查 |
| --- | --- |
| [4-mashup-array-shape.kql](4-mashup-array-shape.kql) | 是否数组、单/多元素分布、双数组同时出现、小数 IO |
| [5-mashup-array-length-distribution.kql](5-mashup-array-length-distribution.kql) | `TotalBytes` 与 `TotalRows` 的数组长度交叉分布（不展示原值） |
| [6-mashup-array-element-validation.kql](6-mashup-array-element-validation.kql) | 各数组元素数值/整数转换、分数/负数/大整数统计（不展示原值） |

这三份 KQL **已在 2026-11-23 POC MCP 成功执行**，04 与 06 首次遇到 `floor()` 参数错误，经 `floor(x, 1.0)` 修正后运行成功；修复后的只读源码已同步 GitHub。**不意味着原 Candidate 获得部署许可。**统计结果无法单独证明多元素数组应求和、取首项或保留列表；需要明确业务语义与 Schema 决策。

**当前阶段不修改：** 目标表、Update Policy、旧 Mashup Function、`candidates/3-parse_MashupEvaluationReport.kql` 和正式 `src/Gateway/.../DatabaseSchema.kql`。

回到 [项目状态](../../../STATUS.md) · [2026-11-23 Evidence](../../../evidence/2026-11-23/README.md) · [设计选项](../../../docs/MASHUP_DESIGN_OPTIONS.md)。
