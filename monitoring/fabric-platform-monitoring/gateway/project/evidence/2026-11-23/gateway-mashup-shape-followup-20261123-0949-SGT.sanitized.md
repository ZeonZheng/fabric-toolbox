> **证据来源：** 用户提供的本地 `eventhouse-remote` MCP 只读 Mashup Shape Follow-up（2026-11-23 09:50–09:55 SGT），GitHub 归档副本。未发现 GUID、完整 URL、原始数组值、凭据等敏感内容。ChatGPT 未直接连接用户 Eventhouse。见 [独立复核](MASHUP_SHAPE_FOLLOWUP_REVIEW.md)。

# Gateway Mashup 数组结构与数值语义只读诊断报告（Shape Followup）

> 依据 `MASHUP_SHAPE_FOLLOWUP_AI_PROMPT.md`（分支 `docs/gateway-monitoring-poc-20261112`）执行，全程只读；未修改 Candidate、Function、目标表 Schema、Update Policy、Uploader，无任何 PROD 操作。

## A. POC、查询时间、工具与三份 KQL 状态

| 项目 | 值 |
| --- | --- |
| 目标数据库 | **Gateway Monitoring**（POC，`eventhouse-remote` item-scoped，`.show database` 核实 ReadWrite；未对 PROD 发出任何调用） |
| MCP 工具 | `eventhouse-remote`（ExecuteKQLQueryTool，只读） |
| 仓库 / 分支 | `ZeonZheng/fabric-toolbox` / `docs/gateway-monitoring-poc-20261112`（非 main） |
| 分支 HEAD Commit | `04ea36aafa074145c93a5d2aa7962bb746345b71`（2026-11-20T08:57:36Z，经 `gh api` 核实） |
| 实际查询时间 | UTC 2026-11-23 01:50–01:55（UTC+8 2026-11-23 09:50–09:55） |
| 查询 4（array-shape） | ❌ 原文执行失败 → ✅ 本地最小修正后成功（见下） |
| 查询 5（length-distribution） | ✅ 原文执行成功 |
| 查询 6（element-validation） | ❌ 原文执行失败 → ✅ 本地最小修正后成功（见下） |
| 数据窗口说明 | Raw 表有 1 天保留策略，本轮窗口为近 24h 的 **36 条**内嵌记录；各项占比与 2026-11-20 Precheck（379 条）方向一致 |

**查询 4/6 语法修正记录（未改动 GitHub 文件）：**
- 报错：`Semantic error: bin(): function expects 2 argument(s)`（两份查询相同）。
- 原因：Kusto 中单参数 `floor(x)` 解析为 `bin()`，需要两个参数。
- 修改：`floor(Number)` → `floor(Number, 1.0)`（查询 4 一处、查询 6 两处），**其余逻辑逐字未动**。修正仅影响小数判定实现方式，不改变统计语义。

## B. 两字段数组单/多元素总数、长度分布与共同出现（查询 4 + 5）

**总体形状（36 条记录）：**

| 指标 | 值 |
| --- | --- |
| TotalBytes 有值 | 23/36（单元素 17，多元素 6，解析失败 0） |
| TotalRows 有值 | 12/36（单元素 4，多元素 8，解析失败 0） |
| 两数组同时出现 | 6 |
| 同时出现且长度相等 | **6/6（100%）** |
| MaxIO 有值 | 14/36 |

**长度组合分布（查询 5，TotalBytesLength × TotalRowsLength → 记录数）：**

| TotalBytes 长度 | TotalRows 长度 | 记录数 | 形态解读 |
| --- | --- | --- | --- |
| 1 | 0 | 17 | 仅 TotalBytes 单元素 |
| 0 | 0 | 7 | 两字段均缺 |
| 0 | 1 | 4 | 仅 TotalRows 单元素 |
| 0 | 2 | 2 | 仅 TotalRows 2 元素 |
| 4 | 4 | 2 | 两数组等长（4） |
| 5 | 5 | 2 | 两数组等长（5） |
| 3 | 3 | 1 | 两数组等长（3） |
| 6 | 6 | 1 | 两数组等长（6） |

**结构性观察（仅陈述，不作语义断言）：**
1. 两数组**同时出现时长度恒相等**（6/6），提示可能存在按索引的元素级关联（如按数据源逐项对应），但**长度相等本身不能证明索引语义**。
2. 两数组**经常独立出现**：仅 TotalBytes 17 条、仅 TotalRows 6 条、均缺 7 条——若两者是同一分解维度的配对指标，独立出现需要上游解释。
3. TotalRows 多元素占比（8/12 = 67%）显著高于 TotalBytes（6/23 = 26%）。

## C. 数组元素可转换性（查询 6，修正后）

| FieldName | 元素总数 | Numeric | LongConvertible | Fractional | Negative | BeyondDouble(>2^53) |
| --- | --- | --- | --- | --- | --- | --- |
| TotalBytes | 44 | 44 | **44** | 0 | 0 | 0 |
| TotalRows | 35 | 35 | **35** | 0 | 0 | 0 |

**结论：** 两数组全部元素均为**整数**，100% 可安全转 long，无小数、无负数、无 double 精度溢出风险。元素级转换**不是**阻塞点；阻塞点仅在"取哪个/怎么取"的语义未定。

## D. MaxIODataBytesPerSecond 小数与整数分布（查询 4）

| 指标 | 值 |
| --- | --- |
| 有值 | 14/36 |
| 可转 double | 14/14（无非数值异常） |
| **含小数** | **12/14（86%）** |
| 负数 | 0 |
| 非有限值（Inf/NaN） | 0 |

**现有 `long` Schema 的精度影响（不输出实际值）：** 86% 的当前样本含小数部分。若维持 `tolong`：这些值要么转换失败为 null（本轮 Precheck 实测 127/127 全失败），要么任何"先取整再转"的方案都会**系统性丢失小数精度**（吞吐量类指标取整误差在聚合分析中会持续累积）。`toreal` 可无损承载全部 14/14 样本。**是否改列类型（long→real）属 Schema 决策，本轮不代决。**

## E. 语义依据检索结果：`SemanticMeaning = Unverified`

| 检索范围 | 方式 | 结果 |
| --- | --- | --- |
| 本地工作区 | 全文检索 `TotalBytes/TotalRows/MashupEvaluationReport` | 仅命中两份 Preflight 报告（`gateway-preflight-20261118-1513-SGT.md`、`gateway-preflight-20261120-1036-SGT.md`）；二者仅记录 Raw 键**存在且类型为 `["string"]`**（314/314 命中）及 Parsed null 统计，**无数组业务定义或展开规则** |
| 本地工作区 | 目录遍历 | 仅 2 份报告 + `.vscode`，**无** Collector/Uploader 源码、Gateway 日志定义、Report/Semantic Model 文件 |
| GitHub 仓库 | Code Search `repo:ZeonZheng/fabric-toolbox` + `TotalBytes` / `MashupEvaluationReport` | 0 条业务定义命中（仓库内容为 KQL 与文档） |

**结论：** 在已授权访问范围内**未找到** TotalBytes/TotalRows 的业务定义、数组展开规则或报表消费关系 → **`SemanticMeaning = Unverified`**。结构证据（等长 6/6、元素全整数）仅支持" redesign 可行性"，**不构成**选择求和/取首元素/按索引关联/逐元素展开的依据。

## F. Verdict：`Ready for candidate redesign discussion`

**判定依据：** 三份诊断全部成功（2 份经记录在案的最小语法修正）；数据层面**未发现新的损坏或异常**（元素 100% 整数可转、无负值/溢出/非有限值、解析失败 0）；但 Candidate 语义问题未解决，**仍不得部署**，进入"候选重设计讨论"阶段。

**数据层面已确认（可作为重设计输入）：**
1. TotalBytes/TotalRows 数组元素 100% 整数、100% long 可转、无精度风险 → 任何"解析数组后取值"的方案在**转换层**无障碍。
2. 单元素场景可无损提取：TotalBytes 17/23（74%）、TotalRows 4/12（33%）。
3. 两数组共同出现时长度恒等（6/6）→ 若上游确认按索引配对，逐元素展开或配对求和在结构上自洽；**但需上游确认，本轮不假定**。
4. MaxIO 86% 含小数 → `long` 列无法无损承载；`toreal` 是数据层面唯一无损路径（Schema 变更需决策）。

**仍需方案选择的问题（本轮不代决）：**

| # | 问题 | 可选路径 | 缺少的证据 |
| --- | --- | --- | --- |
| 1 | TotalBytes/TotalRows 数组语义 | 求和 / 取首元素 / 按索引与另一数组配对 / 逐元素展开成行 | **上游字段定义**（Gateway 日志 schema 或 Collector 代码）；`SemanticMeaning=Unverified` |
| 2 | 两数组独立出现（TotalBytes 单独 17 条、TotalRows 单独 6 条）如何解释 | 同一语义规则覆盖 / 缺省 null | 上游上报逻辑 |
| 3 | MaxIO 目标类型 | `long→real`（需表 Schema 变更）/ 保留 long + 明确取整规则（有精度损失） | Schema 变更评审决策 |
| 4 | 多元素数组的行展开是否可接受（Parsed 行数将 > Raw 记录数） | 展开 / 聚合为标量 | 下游报表消费方式 |

**可以保持原列 / 无需改动的部分：** 五项 Memory/Time 字段（2026-11-20 Precheck 已验证当前数据 0 丢失 0 变化）、AverageIODataBytesPerSecond（real，127/127 成功）、Average/MaxPercentProcessorTime（real，0 丢失）、GatewayObjectId/QueryTrackingId/ConnectionId/EndTimeUTC 等标量字段、DataSourceKinds/DataSourcePaths（已知设计限制，保持空值，不推断）。

**尚缺的证据（下一步建议）：**
1. **最关键**：从 Gateway 产品组/Collector 团队获取 `MashupEvaluationReport` 中 `TotalBytes`、`TotalRows` 的字段定义（为何是数组、数组元素含义、与 `DataSource` 的关系）。
2. 上游确认后修订 Candidate（数组解析 + 语义规则 + MaxIO 类型），重跑 2026-11-20 Precheck 的六项检查 + Dry-run，再进入人工 DDL 评审。
3. 在获得语义确认前，**维持 Blocked 状态的实际约束：不部署现有 Candidate、不改 Schema**。

---

### 附：真实脱敏查询输出汇总（供独立复核）

| # | 查询 | 状态 | 关键输出 |
| --- | --- | --- | --- |
| 1 | 04 array-shape（修正后） | ✅ | RawRows=36；TB 23(17单/6多/0败)；TR 12(4单/8多/0败)；Both=6；SameLen=6/6；MaxIO 14 有值/14 可转/12 小数/0 负/0 非有限 |
| 2 | 05 length-distribution（原文） | ✅ | (1,0)=17；(0,0)=7；(0,1)=4；(0,2)=2；(4,4)=2；(5,5)=2；(3,3)=1；(6,6)=1 |
| 3 | 06 element-validation（修正后） | ✅ | TotalBytes：44/44/44/0/0/0；TotalRows：35/35/35/0/0/0（Elements/Numeric/LongConv/Frac/Neg/>2^53） |
| 4 | `.show database` | ✅ | Gateway Monitoring（POC），ReadWrite |
| 5 | 语义检索 | — | 本地 2 份 Preflight（仅键存在性/类型）；GitHub Code Search 0 命中；`SemanticMeaning=Unverified` |