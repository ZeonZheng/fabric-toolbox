# POC Streaming Parser｜你和本地 AI 各做什么（简明操作版）

> **最新验收 2026-11-20 14:48（UTC+8）：Streaming 功能回归 Passed。** 用户已在 POC 手工执行修改；32/32 新 Parsed 记录两项 Duration 与 Raw 完全一致、0 mismatch、0 observed failures。**不要重复执行 Step 1**，此页保留作为实施记录。下一个对象见 [Aggregation Runbook](AGGREGATION_RUNBOOK.md)。
> 
> **职责分工：你只手动执行一次写命令；本地 AI / Eventhouse Remote MCP 只负责只读检查与报告；ChatGPT 评审是否成功。**

## A. 你在 Fabric 网页上执行（按顺序，不要同时修改其他 Function）

### 1. 打开正确的位置并确认是 POC

1. 打开 `app.fabric.microsoft.com`，进入实际的 **POC Workspace**。人工核实 Workspace 的名称/ID，不能仅依赖之前本地 AI 的“单节点/表结构”判断。
2. 打开其中的 **Eventhouse / KQL Database: Gateway Monitoring**，进入连接该数据库的 **KQL Queryset**。若当前 Queryset 右上角显示另一个数据库，先手动切换到 **POC Gateway Monitoring**。**如无法确认 POC 身份，停止，不运行下面写命令。**

### 2. 确认回滚脚本已打开（暂时不执行）

- [旧版 Streaming Function 回滚脚本](https://github.com/ZeonZheng/fabric-toolbox/blob/docs/gateway-monitoring-poc-20261112/monitoring/fabric-platform-monitoring/gateway/project/kql/poc-parser-fix-20261117/rollback/2-rollback-parse_QueryStreamingReport.kql)，在另一个浏览器标签页打开，以防后续异常。
- 2026-11-20 的本地 Preflight 已确认当前 Streaming Function Body 与 2026-11-17 备份 SHA256 相同。正式操作前，如已有人修改过现场 Function，应停止并重新核对；不要覆盖未识别的修改。
- 可选的只读现场检查：在 POC Queryset 运行：

```kusto
.show functions
| where Name == "parse_QueryStreamingReport"
| project Name, Folder, BodySHA256=hash_sha256(Body)
```

### 3. 只执行一次 Parser 修改

1. 打开 GitHub：[**要执行的 Streaming Candidate 全文**](https://github.com/ZeonZheng/fabric-toolbox/blob/docs/gateway-monitoring-poc-20261112/monitoring/fabric-platform-monitoring/gateway/project/kql/poc-parser-fix-20261117/candidates/2-parse_QueryStreamingReport.kql)。
2. 点击 GitHub 的 **Copy raw contents**（或 Raw → 复制全部），确保复制内容的第一条有效语句是 **`.create-or-alter function`**，末尾含 `}`。
3. 返回**已确认 POC 数据库**的 KQL Queryset，新建一个查询页或清空当前编辑器，**只粘贴这一个 Candidate**。不要混合其他 KQL。
4. 点击 **Run 一次**。成功后截图保留结果；**如报错，先停止，把错误交给 ChatGPT，不要连续点击 Run，也不要立即改三个函数**。

> 这是唯一需要你执行的 Eventhouse **写操作**。它只更新 `parse_QueryStreamingReport` 函数定义，不修改 Schema / Policy；由于目标的 Update Policy 是 Transactional，错误解析可能影响后续 Raw 摄入。

### 4. 记录部署时间，然后交给本地 AI

在同一个 **POC Queryset** 的另一个**新查询**中执行下面这条**只读** KQL：

```kusto
print RecordedAfterPatchUTC = now()
```

复制返回的 UTC 时间（例如 `2026-11-20T05:25:00Z` —— **仅示例，不能使用这个固定值**）。

把以下三项交给本地 AI：
- 实际 `RecordedAfterPatchUTC` 值；
- Candidate 的执行结果：成功/失败（若失败就别进入下一步）；
- POC Workspace / Eventhouse 已经人工核实的事实（不用向外输出敏感 Workspace ID）。

**如果你不想自己执行查询验证，做到这里就可以停止。** 只读检查交给 MCP。

## B. 本地 AI + Eventhouse Remote MCP 执行（只读）

**最新 2026-11-20 14:48 更新：** 首轮 14:22 Postcheck 因零样本为 Inconclusive，之后 [Follow-up](../../../evidence/2026-11-20/STREAMING_FOLLOWUP_REVIEW.md) 获得 Raw 32 行、Parsed 32 行、双指标 32/32 完全匹配，因此本轮功能性修复 **Passed**；本页保留历史流程。

使用 [**POSTCHECK_AI_PROMPT.md**](POSTCHECK_AI_PROMPT.md)，替换其中 `<RecordedAfterPatchUTC>` 为上述实际 UTC 时间，再将 Prompt 交给本地 AI。它会执行本目录已有的三份只读查询：

1. [01-streaming-postdeploy-summary.kql](01-streaming-postdeploy-summary.kql) — Raw / Parsed 新数据及两个 Duration 的非空数量；
2. [02-streaming-postdeploy-keycheck.kql](02-streaming-postdeploy-keycheck.kql) — Key 样本关联及数值匹配；
3. [03-streaming-ingestion-failures.kql](03-streaming-ingestion-failures.kql) — 本次修改后 Update Policy 失败。

**注意：** `RecordedAfterPatchUTC` 是修改**完成之后**记录的时间，可作为一个**保守的验证起点**，可能不包含刚好落在命令完成与时间读取之间的少量事件。Raw 的 `logDate` 与 Parsed 的业务事件时间不同，因此这三份查询应作为**诊断性验证**，不得将不同层的累计行数视作必须相等；若没有新事件，输出 `Inconclusive`。

本地 AI 最终**只需给你一个脱敏 Markdown 报告**，你上传到 ChatGPT 即可。不要让 MCP 执行任何 DDL、Rollback 或 Update Policy 变更。

## C. 如果执行失败怎么办？

- **Candidate Run 报错：** 停止；保存错误文本与时间，不要重复执行。
- **修改后出现新摄入失败、字段仍全 null 或数值异常：** 停止继续修改其他函数，把报告给 ChatGPT 核对；如确认需要回滚，由你在已经确认的 POC Queryset 手动运行 [回滚脚本](https://github.com/ZeonZheng/fabric-toolbox/blob/docs/gateway-monitoring-poc-20261112/monitoring/fabric-platform-monitoring/gateway/project/kql/poc-parser-fix-20261117/rollback/2-rollback-parse_QueryStreamingReport.kql)，并再次检查最新入库及故障。
- **无新数据：** 只标记 `Inconclusive`，不能把空结果判断为“修复成功”或“失败”。
- 回滚只恢复函数定义，**不自动补回**先前失败的摄入批次或历史 null。

## D. 本轮明确不做

**不动 PROD、不动 Uploader、不变更 Update Policies、不修复历史 null、不修改 Aggregation/Mashup、不扩充 Streaming 三个 Raw-only 字段。**

相关：[2026-11-20 Preflight Review](../../../evidence/2026-11-20/PRE_DEPLOYMENT_REVIEW.md) · [当前唯一状态](../../../STATUS.md)。
