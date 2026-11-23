# POC Mashup Parser｜第三轮单函数受控部署 Runbook

> **当前状态（2026-11-23）：Candidate 已由用户在 POC 手工执行；请勿再次执行 A 节 DDL。** 首轮 02:47–02:50 UTC Postcheck 无新 Mashup Raw/Parsed，`INCONCLUSIVE_NO_NEW_RAW`；Live Body 与 Candidate 一致，Policy Enabled/Transactional，0 observed failures。详情见 [独立评审](../../../evidence/2026-11-23/MASHUP_POSTCHECK_1045_REVIEW.md)。
> **下一步：** 先运行 [13 只读 Activity Gate](13-mashup-postdeploy-activity-gate.kql)，`PatchUtc=2026-11-23T02:39:15.4152749Z`，仅当出现新 Mashup Raw 才重新只读运行 10/11/12。无新 Raw 不得判 Passed，不需要仅因 0 数据回滚。下述 A 节保留为历史部署记录。
> 此操作仅限已确认的 **POC** Fabric Gateway Monitoring 数据库。**用户只手工执行 1 条 Function DDL；本地 AI / MCP 只读检查，不能代替用户写入。**

## 证据和允许范围

- 五个目标 Memory / Time 字段的候选数值检查通过：四项各 33/33，ProcessorTime 36/36；目标 Schema 17/17 匹配。见 [Minimal Precheck](../../../evidence/2026-11-23/MASHUP_MINIMAL_PRECHECK_REVIEW.md)。
- [非目标 12 列兼容性 Gate](../../../evidence/2026-11-23/MASHUP_LEGACY_CANDIDATE_COMPATIBILITY_PASS.md) **36 / 36、0 不匹配、PASS**；此结果来自用户报告的本地 MCP，只涵盖当前 Raw 样本。
- [ADR-009 / GW-009](../../../tracking/DECISIONS.md)：`TotalBytes`、`TotalRows`、`MaxIODataBytesPerSecond` 延期，不在当前 P1 验收范围，继续可能为 null。**不以 null 当作零或已修好。**
- 仅更新 `parse_MashupEvaluationReport()` Function Body。**不变更 Table Schema、Update Policy、Uploader、PROD 或历史数据**。
- 当前 Update Policy 是 Transactional，失败可能阻碍源 Raw 摄入。**准备回滚不等于保证批次不会丢失**。

## A. 用户在 Fabric POC 手工执行

1. 进入 [Fabric](https://app.fabric.microsoft.com/) → **已人工核实的 POC Workspace** → Gateway Monitoring Eventhouse/KQL Database → KQL Queryset。确认当前数据库是 POC，**不要**通过名字相似来猜测；不确定就停止。
2. 在另一个页面打开 [**现有现场旧版回滚 KQL**](../rollback/3-rollback-parse_MashupEvaluationReport.kql)，保留备用，**暂时不要运行**。
3. 手动或由本地 AI 只读核对当前 `parse_MashupEvaluationReport` Function Body、目标 Schema、Update Policy 与这次 Precheck 基线一致；原报告函数 SHA256 为 `71c6b4026e3404eb719c5d3fb73b251e725852250b63fd02fcf90ff5c8f73a5c`（需使用同一种 Body 规范化/Hash 算法比对），Policy 仍 Enabled / Transactional。**有任何漂移，停止。**
4. 打开 [**POC-only P1 Candidate**](../candidates/4-parse_MashupEvaluationReport_P1_minimal_POC.kql)；使用 GitHub 的 *Copy raw contents* 复制全文；确认只有一个 `.create-or-alter function`，函数名仅为 `parse_MashupEvaluationReport`。不要使用带 `BLOCKED` 标记的历史 `candidates/3-...` 文件。
5. 在确认的 **POC KQL Queryset** 中只运行这一条 Candidate DDL **一次**。如出现任何错误，停止，保存错误和执行时间，不要连续 Run 或修改其它 Parser。
6. **成功后立即在新的查询中运行只读时间记录：**
   ```kusto
   print RecordedAfterPatchUTC = now()
   ```
   保存真实 UTC 时间（不是固定示例），以及修改成功/失败、所在 POC 数据库已人工核实的信息。此时间在 DDL **之后**读取，是保守 Postcheck 起点，可能遗漏部署完成和记录时间之间极少量数据。

## B. 本地 AI / Eventhouse Remote MCP 只读 Postcheck

把成功执行的事实和真实 `RecordedAfterPatchUTC` 交给本地 AI。让其用**实际 UTC** 替换下列脚本中的 `2099-01-01T00:00:00Z`（只允许替换这个占位时间），分别以只读方式执行：

1. [10-mashup-postdeploy-summary.kql](10-mashup-postdeploy-summary.kql)：Raw 与 Parsed 的新数据/五项非空计数；
2. [11-mashup-postdeploy-keycheck.kql](11-mashup-postdeploy-keycheck.kql)：同一个新数据窗口内按业务标识及五项指标 **哈希签名多重计数**对账，不输出 GUID；
3. [12-mashup-postdeploy-ingestion-failures.kql](12-mashup-postdeploy-ingestion-failures.kql)：分组统计 Patch 后 Update Policy 失败。
4. 只读核对 Live Function Body 与上述 Candidate 逻辑一致，Policy 仍 Enabled / Transactional；不要自动修改 Live Function。

将三个脚本结果、真实 PatchUTC、Live Body/Policy 核对摘要保存为**脱敏** Markdown，返回本对话审查。不要导出完整 Raw、QueryText、连接标识、凭据或故障详情。

### 验收判定

- **PASS（功能性）**：Patch 后确实存在新 Raw 与 Parsed；五项指标对 Raw 的非空/数值保持一致；KeyCheck `RawRows > 0`、`RawRows == ParsedRows`、`DifferingGroups=0`、`UnmatchedRows=0`；Live Body 与 Candidate 匹配；未观察到新增相关 Policy Failure。此为当前样本的 **POC 功能通过**，不是长期上传 Exactly-once。
- **INCONCLUSIVE**：没有足够的新 Raw 或 Parsed 数据，或跨 Raw/Parsed 时钟窗口存在尚未完成入库的批次；先观察新数据和摄入延迟，**不要提前标记失败或通过**。
- **INVESTIGATE / ROLLBACK REVIEW**：出现与本函数修改关联的 Policy Failure、字段明显错误或持续不匹配，暂停其它变更；评估手工运行旧版回滚 KQL。回滚后同样需要核查摄入恢复与是否需要上游重试/回放。**恢复 Function 定义不能自动补回丢失批次。**

## C. 验收之后

- 只有 Postcheck 通过后，才把 Mashup 已验证 Function Body 同步到 fork 正式 `src/Gateway/Gateway Monitoring.KQLDatabase/DatabaseSchema.kql`，同时更新 [STATUS](../../../STATUS.md)、[ISSUES](../../../tracking/ISSUES.md) 和 [CHANGELOG](../../../tracking/CHANGELOG.md)。
- 本轮结束后进入 **P2 Collector Reliability（GW-002）**：源文件 → Eventstream → Raw → Parsed 对账、日志轮转、部分批次失败/重试、去重、长期运行。
- **没有 PROD 变更授权**，三项延期指标仍在 GW-009。
