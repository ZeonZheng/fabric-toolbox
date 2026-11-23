# Mashup 最小修复 Precheck｜2026-11-23 独立评审

> **来源：** 用户提供的本地 AI 只读报告 `gateway-mashup-minimal-precheck-20261123.md`；本次并未直接查询 Live Eventhouse，也未导入完整 MCP 原始查询输出。报告数据均属于**本地 AI 报告的陈述**。**状态：Conditional GO for POC manual deployment review；部署前尚需一个只读兼容性 Gate。**

## 报告支持的结果

- 现场 `parse_MashupEvaluationReport` 仍为旧 `bag_unpack`，报告给出的 Body SHA256 为 `71c6b4026e3404eb719c5d3fb73b251e725852250b63fd02fcf90ff5c8f73a5c`；目标表 17 列，Update Policy Enabled、Transactional；未进行 DDL。
- Raw 最近可用窗口 15 Envelope / 36 内嵌记录：`AverageCommit(byte)`、`AverageWorkingSet(byte)`、`MaxCommit(byte)`、`MaxWorkingSet(byte)` 依次均为 33 Raw NonEmpty / 33 Candidate NonNull，ConversionLoss=0、ChangedByCast=0（报告阈值 0.0001）。
- `TotalProcessorTime(ms)`：36 Raw NonEmpty / 36 Candidate NonNull，ConversionLoss=0、ChangedByCast=0；往返毫秒无超阈值偏差。
- 本地 AI 声明 Candidate 与目标 Schema 列名、列序、类型 17/17 匹配。当前 GitHub 原 Candidate 和这份 v2 **函数体表达式相同**，只是范围决定、文档注释不同：`TotalBytes`、`TotalRows`、`MaxIODataBytesPerSecond` 仍作为 GW-009 Deferred，**不视为已修复**。

## 独立评审的限制与修正

1. **还缺 12 个非目标字段的行为回归。** Candidate 将旧版 `bag_unpack` 整体替换为 `mv-expand + project`。17/17 Schema 匹配不等于这些字段的值、空值与行数一致。新增 [只读兼容性 Gate](../../kql/poc-parser-fix-20261117/value-checks/7-mashup-legacy-candidate-compatibility.kql)：从同一 Raw 样本比较旧 / 新逻辑的**12 个非目标列**，按值签名和出现次数对账；只输出聚合计数，不输出 GUID/Raw。
2. 报告“3 条记录五字段同时为空”的表述与 `TotalProcessorTime` 36/36 非空**矛盾**。应为**另外四个 Memory 字段各 3 条缺值**，没有证据证明五字段同一行同时为空。
3. “无损 / 精确”措辞应收敛为“在当前样本与指定阈值下未观察到差异”，不能推广至全部未来数值。
4. “回滚后无数据损坏”不能保证：当 Transactional Update Policy 失败时，**源 Raw 摄入也可能失败**；恢复旧 Function **不会自动补齐失败批次**，必须另行核对上游重试/回放链路。
5. 报告提供的是样本/Schema/Function 结果摘要，**没有附实际 Kusto 查询原始结果或全 17 字段逐值对照**。故此时不能称为最终部署验收。
6. 由文件中 Candidate 与历史 Candidate 的函数体相同可知，**这是验收范围改变，不是一个新的、更安全的解析算法**；保留原 Candidate 历史 Blocked 记录，审查完成后再建立标识明确的新 POC-only 部署文件。

## 下一步（唯一 Gate）

- 让本地 AI / Eventhouse Remote MCP **只读执行** [7-mashup-legacy-candidate-compatibility.kql](../../kql/poc-parser-fix-20261117/value-checks/7-mashup-legacy-candidate-compatibility.kql)。
- **PASS** 需要 `LegacyRows > 0`、`LegacyRows == CandidateRows`、`DifferingSignatureGroups == 0`、`UnmatchedRows == 0`；0 Raw 为 Inconclusive、查询失败为 No-Go；不输出签名或敏感字段。
- 若 Gate PASS，重新确认 POC 数据库与 Live Function 未漂移，并准备可审核的 v2 Candidate、现场旧版 Rollback、准确部署时间、Postcheck（新 Raw / Parsed 数据五项数值、Policy failures）；由用户决定是否在 POC **手工**执行 DDL。**未经独立评审，不执行。**
- 若 Gate FAIL，先查看是否是非目标字段差异、空值或行数变化，调整 Candidate，再重复只读检查；不要部署现有 v2。
- P2 Collector Reliability 与 PROD 部署仍是独立验收关口，未批准。

相关：[STATUS](../../STATUS.md) · [GW-001/GW-009](../../tracking/ISSUES.md) · [ADR-009](../../tracking/DECISIONS.md)。
