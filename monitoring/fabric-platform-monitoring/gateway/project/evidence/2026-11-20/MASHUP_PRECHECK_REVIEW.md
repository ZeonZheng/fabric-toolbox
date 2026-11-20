# 2026-11-20 Mashup Parser｜部署前只读验证独立技术评审

> **结论：Blocked — 原有 Mashup Candidate 不得执行。** 用户上传 [本地 MCP 报告](gateway-mashup-precheck-20261120-1626-SGT.sanitized.md)，目标 POC，`parse_MashupEvaluationReport` 尚为旧版，未执行任何 DDL。  
> **证据时间：** 报告称 UTC 08:27–08:36（对应 SGT 16:27–16:36），另写 SGT 16:27–16:44，存在 8 分钟不一致；记录原样保留，下一轮需再确认结束时间。  
> **ChatGPT 独立复核：** 对照分支 `candidates/3-parse_MashupEvaluationReport.kql` 和 `3-mashup-additional-metrics.kql`：三项错误转换表达式存在于 Candidate；此前 17/17 Schema Check 验证的是目标**结构**而非数值语义。

## 当前 Raw 实测（59 Envelope、379 Embedded Records）

| Raw Field | 实测 Raw 非空 | 原 Candidate 非空输出 | 转换失败 | 原因 |
| --- | ---: | ---: | ---: | --- |
| `TotalBytes` | 243 | 0 | **243/243** | JSON **数组文本**，212 单元素、31 多元素；`tolong(整个数组字符串)` 不可能得到所需单个 long |
| `TotalRows` | 34 | 0 | **34/34** | JSON **数组文本**，14 单元素、20 多元素；`toreal(整个数组字符串)` 不可用 |
| `MaxIODataBytesPerSecond` | 127 | 0 | **127/127** | 十进制小数文本，`tolong` 失败而 `toreal` 127/127 成功；现有目标表为 `long`，不可直接填 real |
| `AverageIODataBytesPerSecond` | 127 | 127 | 0 | 当前样本转换正常 |
| `AveragePercentProcessorTime` | 135 | 135 | 0 | 当前样本转换正常 |
| `MaxPercentProcessorTime` | 135 | 135 | 0 | 当前样本转换正常 |

**重要：** `ChangedByCast=0` 对 3 个 `CandidateNonNull=0` 的字段**毫无精度验收意义**（没有成功转换值可比较）；不得把这三项视为“未改变数值”。

五项 Memory/Time（AverageCommit、AverageWorkingSet、MaxCommit、MaxWorkingSet、TotalProcessorTime）在本轮 379 条 Raw 中均 379/379 有值、转换损失 0；只是当前样本的读值成功证据。旧 Parsed 总计 2862 行，`TotalBytes` / `TotalRows` / `MaxIO` 均无非空值，说明缺陷在旧版也存在；**本次没有部署造成新回归**，但旧 Candidate 无法修好这些存量缺陷。

## 决策缺口

1. **TotalBytes / TotalRows：数组语义需要先查明。** 单元素 212 / 14 行可取出一个可转数值的元素，但不应据此推断多元素应 `sum()`、`first()`、`max()`，更不能未经证据把数组当单值丢弃剩余值。可进一步检查两个数组的长度对应关系、每元素数据类型与上游 Gateway 日志/Power BI 模板的消费语义。如果确实是“每个源或每个分区”的多项度量，可能需要新明细表而非强行压成单个标量；这属于架构选项，**尚无定案**。
2. **MaxIO：目标列 `long` 与真实小数值冲突。** `toreal` 只解决读值，不解决与目标表类型的兼容性。选项为保留 `long` 并明确 Round/Truncate 规则（精度损失、汇总影响），或评审改成 `real`（表、Semantic Model、Report、Update Policy、Schema/回滚连带影响），也可以新增高精度列。**现在不选择、不改目标表**。
3. **DataSourceKinds/Paths：** 379/379 Raw 无这两个独立 Key，但含 DataSource；按项目原有 ADR-006，**不强行推断、不改目标 Schema**，仍为 GW-005 数据覆盖缺口。
4. **只读验证仍需重做**：修改 Candidate 之后必须重复 Dry-run、getschema/17列兼容、数值/数组与 Policy 检查。只改 5 个已经验证的 Memory/Time 字段，同时暂时把另外 3 个保持已知 null 是一种**待明确审议的缩小范围方案**，不能误称“全字段修复”。

## 状态与变更边界

- **GW-001 / Mashup：Blocked（设计和转换）。** GW-001 的 Streaming / Aggregation 子项已 POC 功能通过，不能回退其已验收状态。
- **GW-005：** 同步记录 Raw 数组无法无损投影到标量、MaxIO 目标数值类型不一致的 Schema/字段保留设计风险。
- **本轮不修改 Candidate DDL、不修改正式 DatabaseSchema.kql、不修改表/Policy、不回滚 Streaming 或 Aggregation、不操作 PROD**。
- 下一条技术验证：由本地 MCP 只读运行 [数组形状/元素诊断](../../kql/poc-parser-fix-20261117/value-checks/MASHUP_SHAPE_README.md)，并寻求原始字段含义/报表用法的证据；三份新 KQL 尚未在用户 POC 引擎验证语法。

**原始报告仅支持数组长度与类型事实，不支持多元素数组如何聚合、也不支持直接改写目标列类型。**
