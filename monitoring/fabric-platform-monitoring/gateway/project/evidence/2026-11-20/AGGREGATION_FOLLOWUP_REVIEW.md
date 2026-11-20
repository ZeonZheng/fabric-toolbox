# Aggregation Parser｜2026-11-20 15:47–15:51 POC Follow-up 独立评审

> **Verdict：POC 字段功能回归 Passed。** 数据来自用户上传的 [本地 MCP 只读报告](gateway-aggregation-followup-20261120-1547-SGT.sanitized.md)，不是 ChatGPT 直接访问 Eventhouse。  
> **Patch UTC：** `2026-11-20T07:24:24.1258875Z`；报告执行时间 `2026-11-20 15:47–15:51`（Asia/Singapore）。

## 实测结论

| 检查 | 数值 | 解释 |
| --- | --- | --- |
| 现场 Function 与 Candidate | **BodyMatchesCandidate=true**；同规则 SHA256 `626cb51072d65bf416480da28ddc8646bf96dc9756554db3436c0f845e2c7f21` | 目标 Function 在 POC 确实生效 |
| Policy | Enabled / Source=`GatewayReports-Raw` / Transactional | 未观察到配置漂移 |
| 新 Raw vs Parsed | 2 Envelope / **7 Raw 记录**，**7 Parsed 记录** | 当次窗口行数一致，不等于全量/Exactly-once 保证 |
| Query Execution 的 Average/Max/Min Duration | Raw 7/7 非空，Parsed 7/7 非空 | 恢复字段有值 |
| Data Processing 的 Average/Max/Min Duration | Raw 1/7 非空，Parsed 1/7 非空 | **源 Raw 的真实空值被保留**；不能要求 7/7 |
| 复合 Key 样本对照 | **2/2** 个 Key 关联命中；三个 QueryExecution 字段各 **2/2** 匹配，三个 DataProcessing 字段各 **1/1** 匹配；**0 mismatches** | 六字段在有值样本上均符合预期；**不是七条记录的逐行对照** |
| Patch 后 Update Policy ingestion failures | **0 observed** | 未观察到相关摄入失败，不能证明无其它静默数据问题 |
| 原 Query 07 的错误声明 | 本轮报告确认先前复制粘贴 Bug 不存在 | 此前已审计真实 GitHub KQL 六组 × Matches/Mismatches 共 12/12 引用正确 |

## 范围与保留风险

1. **POC 功能性修复 Passed**：现有 Candidate 显式读取 Raw 的 `(ms)` Key，六列 Raw 非空/Parsed 非空以及值匹配检查在当前新数据中成立。
2. **样本有限**：只有 2 个唯一组合 Key、7 条内嵌 Raw 记录、约 17 分钟窗口。需要后续 24h 实际运行观察，验证其它组合及异常情况。
3. **Key 去重限制**：Query 07 使用 `arg_max` 对重复的组合 Key 取最新；不能因此宣称七条完整记录都逐行数值对账，更不能证明无重复、无丢失。
4. **历史 Parsed 的旧 null 不会自动修复**。若业务上需要，需要独立设计补录、去重和回放。
5. **长远 PROD**：Uploader 文件/Batch-level audit、重试/幂等、Retention / Backfill / 多节点等仍属于 GW-002～GW-007 未完成事项。
6. **Mashup 尚未修改**：Raw 有 `TotalBytes` 等值，但旧 Parsed 有缺陷，需在任何 Function DDL 之前补充只读验证。

## 源码管理

- 已将**仅这一个已验证的 Aggregation Function** 同步至 fork `monitoring/fabric-platform-monitoring/src/Gateway/Gateway Monitoring.KQLDatabase/DatabaseSchema.kql`。
- 之前验证的 Streaming Function 保持不变；Mashup 等其余 Function、Update Policies、Gateway Uploader 和 PROD 均未改。
- 源码更新只发生在 GitHub 工作分支，不会自动修改 Eventhouse。旧版完整源码 [2026-11-17 baseline](../2026-11-17/DatabaseSchema.source.kql)，回滚脚本 [Aggregation rollback](../../kql/poc-parser-fix-20261117/rollback/1-rollback-parse_QueryExecutionAggregationReport.kql) 均保留。

**下一阶段关口：** 在 POC 部署 Mashup Candidate 之前，通过 [Mashup 额外字段只读验证](../../kql/poc-parser-fix-20261117/value-checks/3-mashup-additional-metrics.kql)，尤其确认 `TotalBytes` 可以从 Raw 正确转换，且其它原有字段没有新精度/丢值问题。
