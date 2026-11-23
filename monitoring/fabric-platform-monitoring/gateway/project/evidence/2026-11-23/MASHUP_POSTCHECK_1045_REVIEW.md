# Mashup POC Postcheck｜2026-11-23 10:47–10:50 SGT 独立评审

> **来源：** 用户提供 `gateway-mashup-postcheck-20261123-1045-SGT.md`，本地 AI 通过 item-scoped Eventhouse Remote MCP 在 POC 执行只读查询。以下保留脱敏结果摘要；**本助手未直接查询 Eventhouse**。

## 事实

- **PatchUtc：** `2026-11-23T02:39:15.4152749Z`，为用户运行 DDL 后记录的时间。
- **Live Function：** `parse_MashupEvaluationReport` Body 与专门 P1 Candidate 逐字节一致、规范化 SHA256 一致；报告中的 SHA256 `7f333c39791810d52531b3340cbbce0ffb998d2925f0f0a1d594edf8e82813`。Live 已不是旧 `bag_unpack`。
- **Update Policy：** Enabled=true，Source=`GatewayReports-Raw`，Query=`parse_MashupEvaluationReport()`，Transactional=true，无报告所见配置漂移。
- **Query 10：** Patch 后 Raw 0 行、Parsed 0 行，五项目标指标全部 0 个非空，LatestUtc=null。
- **Query 11：** RawRows=0 / ParsedRows=0 / DifferingGroups=0 / UnmatchedRows=0，`Result=INCONCLUSIVE_NO_NEW_RAW`。
- **Query 12：** Patch 后没有 `OriginatesFromUpdatePolicy=true` ingestion failure 记录（0 observed）。
- **补充活动查询：** Mashup Raw 当时有 15 Envelope，最近 6h/1h 均为 0；最新 `logDate=2026-11-22T13:35:13Z`，早于 Patch ~13.2h；实际 Postcheck 于 02:47–02:50 UTC 执行。

## 评审结论

**INCONCLUSIVE（无新 Raw）而非 FAIL/PASS。** Live 函数已应用且 Policy 配置如预期，但**无法**从 0 新记录验证五个 Memory/Time 字段真实摄入后是否正确，或 Raw / Parsed 是否一致。

谨慎修正本地报告两项推断：

1. **“0 Policy failures”不等于事务性策略未阻碍源数据摄入。** 窗口内根本没有 Mashup 新 Raw 样本，故未触发成功业务数据路径；是否存在源端未上传的文件需要单独 Collector 检查。
2. **“其它 logType 此前活跃”不能证明 Patch 之后仍有上传。** 如持续无 Mashup，先看其它 Report 的最新活动与 VM Scheduled Task，再判断是源端日志稀疏还是上传不活跃；不自动归因于 Parser 故障。

## 下一步

- **不重复运行 DDL、不因零样本回滚。** Mashup Function 保持已部署；GW-001 Mashup 子项维持 `POSTCHECK_INCONCLUSIVE_NO_NEW_RAW`，P1 不能标记 Passed。
- 使用 [13-mashup-postdeploy-activity-gate.kql](../../kql/poc-parser-fix-20261117/controlled-rollout/13-mashup-postdeploy-activity-gate.kql) **只读检查一次**最近活动：只有 `MashupRawRecords > 0`，再使用**相同 PatchUtc** 运行 [10/11/12](../../kql/poc-parser-fix-20261117/controlled-rollout/MASHUP_RUNBOOK.md) 完成回归。Activity Gate 可同时观察其他 Report 是否还在产生 Raw；`OtherReportEnvelopes>0` 不等于可靠性验收。
- 如果期望立即制造一个事件，可由用户在 POC 受控执行一次已知经该 Gateway 的刷新，并核实本地 Mashup 文件写入、Uploader 调度与 Eventstream；**刷新不保证一定生成 MashupEvaluationReport**。不要伪造 Raw / Parsed 或用旧业务事件做新数据回归。
- 若新 Raw 出现但仍无对应 Parsed / 出现相关 failures，再单独进入故障分析、人工 Rollback Review。
- 同时允许开启 **P2 Collector Reliability（GW-002）** 的准备工作；不要把 P1 Mashup 功能判 Passed，不将 Function 合入 fork 正式 `DatabaseSchema.kql`，也不动 PROD。

关联：[STATUS](../../STATUS.md) · [GW-001](../../tracking/ISSUES.md) · [部署记录](MASHUP_MANUAL_DEPLOYMENT_20261123.md)。
