# Mashup Shape Follow-up｜2026-11-23 独立技术复核

> **输入：** [本地 AI 原始只读报告](gateway-mashup-shape-followup-20261123-0949-SGT.sanitized.md)。执行时段 **2026-11-23 09:50–09:55 SGT**；当前 Raw 表保留 1 天，本次只分析 **36 条**新窗口记录。  
> **阶段结论：** `Ready for candidate redesign discussion` = **允许讨论重新设计候选，而不是允许部署**。运行中的 Mashup Function 仍为旧版，没有 DDL/Schema/Policy/Uploader/PROD 变更。

## 样本数据（重新核算长度）

| 检查项 | 结果 | 说明 |
| --- | --- | --- |
| TotalBytes 有值 | 23/36（17 单元素、6 多元素） | 17×1 + (4×2 + 5×2 + 3×1 + 6×1)=**44** 个元素 |
| TotalRows 有值 | 12/36（4 单元素、8 多元素） | 4×1 + 2×2 + 4×2 + 5×2 + 3×1 + 6×1=**35** 个元素 |
| 两数组同时出现 | 6/36；6/6 长度相等 | 仅说明结构，**不证明 index 配对、同一数据源、求和或首项含义** |
| 数组元素 | TotalBytes **44/44**、TotalRows **35/35** 为数字且 long 可转，当前样本无负数/小数/超过 2^53 | 数值解析在此样本可行，不能以此推断所有将来版本的整数边界 |
| MaxIODataBytesPerSecond | 14/36 有值，14/14 double 可转，12/14 含小数 | **不能精确写入现有 long**；`real` 能保存近似小数，**不是十进制数学意义无损**（浮点精度仍需评估） |
| 语义证据 | `SemanticMeaning=Unverified` | 不能确定字节与行数数组的“每元素”代表什么 |

## 复核并修正 KQL 语法

- 原 GitHub `4-mashup-array-shape.kql` 使用了 `floor(IO)`，`6-mashup-array-element-validation.kql` 使用了两处 `floor(Number)`，本地 AI 在 KQL 引擎发现 `bin(): function expects 2 argument(s)`。
- 在**不改变统计意图**的条件下已将 GitHub 两个 KQL **只读脚本**修为 `floor(IO, 1.0)` / `floor(Number, 1.0)`，与本地 AI 成功执行的版本保持一致；`5-mashup-array-length-distribution.kql` 未改。
- 新的 KQL 脚本变更**只发生在 GitHub**，没有重新执行现场命令，也没有修改任何 Function。

## GitHub Fork 下游源码检查（本次额外完成）

本地 AI 原报告的 `GitHub Code Search=0` 不能证明没有可检查的项目源码：工作分支**确实包含**：
- `monitoring/fabric-platform-monitoring/src/Gateway/Gateway Monitoring.SemanticModel/definition/tables/*.tmdl`；检查 6 个业务 TMDL（Execution Measures、Gateway Node Info、Query Connections、Query Execution、System Counters、Calendar），**没有发现** `MashupEvaluationReport`、`TotalBytes`、`TotalRows`、`MaxIODataBytesPerSecond` 直接引用；
- `monitoring/fabric-platform-monitoring/src/Gateway/Gateway Monitoring.Report/report.json`：无上面四个字段或表名的**直接文字引用**；出现 `Mashup_profiles_Background` 图片资源和关于 Mashup container 的说明不等于消费该表；
- `monitoring/fabric-platform-monitoring/src/Gateway/Gateway Monitoring.KQLDashboard/RealTimeDashboard.json`：同样未发现四个字段/表名直接引用；
- `monitoring/fabric-platform-monitoring/src/Gateway/Gateway Monitoring.KQLDatabase/DatabaseSchema.kql`：**有** Mashup Parsed Table、Function、Update Policy；其存在不证明它是当前 Report 的必要输入。

**重要限制：** 尚未验证本地 POC/PROD 当前部署版本和 repo 完全一致；未解析仓库二进制 `gateway/PBI Report/Gateway Monitor.pbit`，也未排除通过 Query、动态表达式或第三方报表间接消费。**只能说当前可读 Fabric 版本化资产未观察到直接依赖，不能宣称 Mashup 字段全业务无用。**

## 方案分支（尚未决策）

| 方案 | 对现有 Schema 的影响 | 适用条件 / 风险 |
| --- | --- | --- |
| **A. P1 最小化**：仅修复 Memory/Time 等已证实标量字段，对三个问题字段保持已知 null；对多值语义另建 Backlog | 无 Schema 变更 | **必须明确接受不完整性**，不可声称 Mashup 全字段修复或关闭相关问题。可保留 Eventhouse Parsed Table 结构，仍需全量 Schema/Dry-run 回归 |
| **B. 有定义的标量聚合**：先取得供应商/上游官方字段语义，再按定义决定 SUM/FIRST/MAX 等；MaxIO 的小数另选择明确舍入方案 | 有潜在语义损失，仍可保持旧 Schema（有损） | 需上游确认聚合/取整规则及下游消费者对精度的要求 |
| **C. 完整保留多值与小数**：数组元素明细表 / 动态数组列 + MaxIO real 或其它数值类型 | Schema、semantic model、report、Policy / 回滚可能需要迁移 | 更符合保真目标，但变更范围扩大、需测成本/兼容 |

**没有任何方案被默认批准。** 不部署原 Mashup Candidate。下次不再无意义地重复相同聚合统计；优先明确业务重要性与外部语义定义，然后决定是否以及如何开展 P1 Mashup 修复。

## 项目决定边界

- Streaming 32/32 与 Aggregation 2/2 Key 的 POC 回归仍 Passed，正式 fork KQL 源码保持不变。
- **Mashup 当前 Candidate 仍 Blocked**，原表/Function/Policy 未改；17/17 静态 Schema Check 不等于三字段正确。
- 下一阶段的唯一决策输入：① 当前业务/Report 是否需要这三个 Mashup 字段；② 上游 Gateway 提供何种多值定义；③ 对 MaxIO 是否要求浮点保真。若无法获得官方解释，应该先做书面范围决策，不擅自定义数组语义。

[项目状态](../../STATUS.md) · [GW-001/GW-005](../../tracking/ISSUES.md) · [技术决策](../../tracking/DECISIONS.md)。
