# MashupEvaluationReport｜字段必要性及实现范围决策（待定）

> **2026-11-23**｜仅用于指导 POC 下一步，不是已批准的变更。**当前运行中的 Mashup Function 未修改，原 Candidate 标记 Blocked；本文件不授权 Schema / Update Policy / PROD 变更。**

## 已验证事实

- 2026-11-20 Precheck：TotalBytes Raw 非空 243/243、TotalRows 34/34、MaxIO 127/127，在旧 Candidate 直接转换均为 null；旧 Parsed 同样全 null。
- 2026-11-23 Shape Follow-up：Raw 36 行；TotalBytes 23 行、44 个全部可转 long 元素（17 单/6 多）；TotalRows 12 行、35 个可转 long 元素（4 单/8 多）；两字段同现 6 行，**6/6 长度相等**但无已验证的 index 或聚合含义。MaxIO 14 非空，12 含小数，当前目标 long 不支持精确保留。见 [证据和独立复核](../evidence/2026-11-23/MASHUP_SHAPE_FOLLOWUP_REVIEW.md)。
- **Source Usage（当前 fork 的可读版本化资产）**：`src/Gateway/Gateway Monitoring.SemanticModel/definition/tables/` 的 6 个业务 TMDL、`src/Gateway/Gateway Monitoring.Report/report.json`、`src/Gateway/Gateway Monitoring.KQLDashboard/RealTimeDashboard.json` 中未见 `MashupEvaluationReport` / `TotalBytes` / `TotalRows` / `MaxIODataBytesPerSecond` 的直接引用；`DatabaseSchema.kql` 中存在该表/Policy。报告中的 Mashup 背景图和说明文字≠实际消费数据。**不覆盖** Gateway Monitor.pbit 二进制、已部署的真实 SM、其它报表/动态引用。
- `toreal` 可以将当前 MaxIO 小数字符串转为浮点值，但不保证十进制**数学上完全精确**。当前绝不自动变更 `long` 列。

## 待明确的一个范围问题

**这三个字段是否为当前 Gateway Monitoring 的必要指标（必须在 POC/PROD 报表中可用）？**

| 方案 | 何时适用 | 结果与限制 |
| --- | --- | --- |
| **A. POC 先聚焦实际所需指标** | 在部署的报表、语义模型、监控需求中确实不使用这三个字段 | 可以讨论将这部分完整解析记为独立 Backlog；**不得**声称 Mashup 全字段修好，也不可无声关闭 GW-001。是否需要一个仅修复 Memory/Time 的最小 Function 需另行 Schema/Dry-run/回归评审 |
| **B. 维持标量目标列，但按**证实**的定义投影** | 业务必须有单个值且微软/上游确认多值时合理聚合/取值规则 | 一旦定义成立再设计 `SUM`/`MAX`/首元素等转换；MaxIO 精度要求若允许有损，可明确舍入策略，**不能未经批准自动截断** |
| **C. 保留多元素和高精度** | 要求追踪每个元素、用于未来诊断或数据不可有损 | 可考虑增加数组 dynamic 或单独子表，以及 MaxIO 更适当的数值类型；涉及 Eventhouse Schema、Update Policy、Report/SM、回滚和迁移 |

## 如果要求上游确认，请提供具体问题

在向 Microsoft Gateway 团队/Unified Support 发技术问题前，先确认当前 PROD 监控是否真的需要这些字段。随后用英文集中询问：

1. In `MashupEvaluationReport_*.log`, what does each element of the JSON-encoded `TotalBytes` and `TotalRows` arrays represent? Are the elements associated with partitions, data sources, evaluations, or another grouping dimension?
2. Why can `TotalBytes` and `TotalRows` appear independently? When both arrays are present, is element-by-index alignment guaranteed, or merely coincidental?
3. Is there a supported **single scalar** interpretation for an evaluation (e.g. sum of all elements)? If not, what is the recommended schema for retaining these metrics?
4. Is `MaxIODataBytesPerSecond` defined as a decimal-valued throughput metric, and what precision/units are expected in downstream reporting? How is it handled in Microsoft's official gateway performance reporting?
5. Are there version-specific differences or a public schema for these performance report columns?

仅提供**脱敏形状/字段名**，不向外发送内部 Raw、DataSource、QueryText、GUID、Token。

## 准入与后续

- 首先核实用户**实际部署**的 Gateway Monitor Report/SM 与 PROD 需求，而不是仅凭 GitHub 版本化资产推断。
- 如果字段无消费者，单独确认缩减 POC 修复范围可否接受；如果是必需指标，先取得有依据的业务定义和小数精度决策，再设计 Candidate。
- 新 Candidate 必须重新 **Dry-run、目标 Schema 一致性、字段非空/数值回归、Update Policy 安全与回滚**，按 POC 单函数受控流程执行。**原 Mashup Candidate 仍禁止部署**。

[STATUS](../STATUS.md) · [GW-001/GW-005](../tracking/ISSUES.md) · [Nov23 评审](../evidence/2026-11-23/MASHUP_SHAPE_FOLLOWUP_REVIEW.md)。
