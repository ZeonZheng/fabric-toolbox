# MashupEvaluationReport｜字段必要性及实现范围决策（已选方案 A）

> **2026-11-23**｜已确认三个问题指标对当前 POC 与计划中的 PROD 监控并非必需，**采用方案 A：延期为独立 GW-009 Backlog**。这是业务范围决定，**不是技术修复或上线批准**。当前 Mashup Function 未修改、原 Candidate 仍 Blocked；本文件不授权 Schema / Update Policy / PROD 变更。

## 已验证事实

- 2026-11-20 Precheck：TotalBytes Raw 非空 243/243、TotalRows 34/34、MaxIO 127/127，在旧 Candidate 直接转换均为 null；旧 Parsed 同样全 null。
- 2026-11-23 Shape Follow-up：Raw 36 行；TotalBytes 23 行、44 个全部可转 long 元素（17 单/6 多）；TotalRows 12 行、35 个可转 long 元素（4 单/8 多）；两字段同现 6 行，**6/6 长度相等**但无已验证的 index 或聚合含义。MaxIO 14 非空，12 含小数，当前目标 long 不支持精确保留。见 [证据和独立复核](../evidence/2026-11-23/MASHUP_SHAPE_FOLLOWUP_REVIEW.md)。
- **Source Usage（当前 fork 的可读版本化资产）**：`src/Gateway/Gateway Monitoring.SemanticModel/definition/tables/` 的 6 个业务 TMDL、`src/Gateway/Gateway Monitoring.Report/report.json`、`src/Gateway/Gateway Monitoring.KQLDashboard/RealTimeDashboard.json` 中未见 `MashupEvaluationReport` / `TotalBytes` / `TotalRows` / `MaxIODataBytesPerSecond` 的直接引用；`DatabaseSchema.kql` 中存在该表/Policy。报告中的 Mashup 背景图和说明文字≠实际消费数据。**不覆盖** Gateway Monitor.pbit 二进制、已部署的真实 SM、其它报表/动态引用。
- `toreal` 可以将当前 MaxIO 小数字符串转为浮点值，但不保证十进制**数学上完全精确**。当前绝不自动变更 `long` 列。

## 已确认的范围决定

`TotalBytes`、`TotalRows` 和 `MaxIODataBytesPerSecond` **不属于当前必要验收字段**。三字段保留已知 null、数组语义与精度缺陷的证据，独立以 [GW-009](../tracking/ISSUES.md) 追踪；不阻塞当前 P1 必需指标验收，也不以 null 代 0。未来出现新需求或发现直接消费者时重新评估。

**下列 B/C 仅作为未来可选设计；A 已作为当前范围决定采纳。**

| 方案 | 何时适用 | 结果与限制 |
| --- | --- | --- |
| **A. 当前采纳：仅验收必要指标** | 用户已确认这三个字段不属于当前目标监控范围 | 三字段修复单独转 GW-009，不声称 Mashup 全字段修复；GW-001 必需字段仍按事实验收。若其他必需字段有问题，再设计最小 Candidate 并执行 Schema/Dry-run/回归 |
| **B. 维持标量目标列，但按**证实**的定义投影** | 业务必须有单个值且微软/上游确认多值时合理聚合/取值规则 | 一旦定义成立再设计 `SUM`/`MAX`/首元素等转换；MaxIO 精度要求若允许有损，可明确舍入策略，**不能未经批准自动截断** |
| **C. 保留多元素和高精度** | 要求追踪每个元素、用于未来诊断或数据不可有损 | 可考虑增加数组 dynamic 或单独子表，以及 MaxIO 更适当的数值类型；涉及 Eventhouse Schema、Update Policy、Report/SM、回滚和迁移 |

## 将来重新启用 GW-009 时的上游核实问题

**当前不需要联系 Microsoft。** 未来只有在明确出现实际业务消费/监控需求后，才考虑向 Microsoft Gateway 团队/Unified Support 用英文集中询问：

1. In `MashupEvaluationReport_*.log`, what does each element of the JSON-encoded `TotalBytes` and `TotalRows` arrays represent? Are the elements associated with partitions, data sources, evaluations, or another grouping dimension?
2. Why can `TotalBytes` and `TotalRows` appear independently? When both arrays are present, is element-by-index alignment guaranteed, or merely coincidental?
3. Is there a supported **single scalar** interpretation for an evaluation (e.g. sum of all elements)? If not, what is the recommended schema for retaining these metrics?
4. Is `MaxIODataBytesPerSecond` defined as a decimal-valued throughput metric, and what precision/units are expected in downstream reporting? How is it handled in Microsoft's official gateway performance reporting?
5. Are there version-specific differences or a public schema for these performance report columns?

仅提供**脱敏形状/字段名**，不向外发送内部 Raw、DataSource、QueryText、GUID、Token。

## 准入与后续

- 当前已接受三字段退出 P1 必需指标范围，独立追踪 GW-009；仅为工程验收范围决定，不能宣称已部署报表完全不存在该依赖。PROD 进入验收前仍应核对实际消费者与范围。
- 核对剩余**必需的** Mashup 指标在现有 Parsed 新数据中的准确性；仅发现需要修复的缺口时，讨论最小 Function Candidate。
- 如需新 Candidate，必须重新 **Dry-run、目标 Schema 一致性、必需字段非空/数值回归、Update Policy 安全与回滚**，按 POC 单函数受控流程执行。**原 Mashup Candidate 仍禁止部署**。

[STATUS](../STATUS.md) · [GW-001/GW-005](../tracking/ISSUES.md) · [Nov23 评审](../evidence/2026-11-23/MASHUP_SHAPE_FOLLOWUP_REVIEW.md)。
