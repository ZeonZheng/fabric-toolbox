# Gateway Monitoring｜当前状态

> **更新时间：** 2026-11-20 15:29（Asia/Singapore）｜**当前阶段：** P1 Parser 字段准确性修复｜**POC：** 单节点｜**PROD：** 未批准

| 工作项 | 现状 | 证据 / 限制 |
| --- | --- | --- |
| Gateway → Fabric → Report | **功能 POC 通过** | 八类 Report 在此前窗口都有入库数据，但端到端不丢失/不重复未验收 |
| **Streaming Parser** | **POC 新数据功能回归 Passed** | 已部署，与 Candidate Body 逐字节一致；32/32 新 Parsed 两个 Duration 非空、Raw 对照 32/32、0 mismatch；24h 稳定性/历史回填另议 |
| **Aggregation Parser** | **已在 POC 手工部署，回归 Inconclusive** | 实际 PatchUtc `2026-11-20T07:24:24.1258875Z`；现场 Body 与 Candidate 一致（SHA256 `626cb510…c7f21`）；15:29 Postcheck Raw 0 / Parsed 0，六 Duration 无可比新样本 |
| Aggregation Update Policy | **配置正常，无当前失败证据** | Enabled、Source Raw、Transactional；Patch 后 `0 observed` failures，不能据此认定修复成功 |
| Mashup Parser | **未部署** | Raw/Parsed 字段映射仍有缺陷，`TotalBytes` 等需额外检查 |
| GitHub 源码 | **仅 Streaming 已同步正式 Schema** | Aggregation 的 Candidate/回滚/只读检查均已备齐，但因回归未通过，**正式 `DatabaseSchema.kql` 暂不修改 Aggregation** |
| PROD Collector / Upload Manifest | **未验收** | GW-002～GW-007：文件/批次级对账、重试幂等、Retention、Multi-node 等仍待验证 |

## 2026-11-20 证据

- Streaming **14:48 Passed**：[实测报告](evidence/2026-11-20/STREAMING_FOLLOWUP_REVIEW.md)。
- Aggregation **15:25–15:29 Inconclusive**：[脱敏原报告](evidence/2026-11-20/gateway-aggregation-postcheck-20261120-1525-SGT.sanitized.md) · [独立评审](evidence/2026-11-20/AGGREGATION_POSTCHECK_REVIEW.md)。
- **原报告关于 Query 07“错误引用字段”的说法不符合当前 GitHub 代码。** 已逐项核对 6 个指标的 Matches/Mismatches 共 12 个表达式，均正确。没有据此改动原查询。
- 上次 Aggregation Raw 最新 `logDate=2026-11-20T06:45:45Z`（在本次 Patch 之前）；原报告所谓“其它类型在 Patch 后正常”引用了更早的 Streaming 验证，应重新检测。

## 当前唯一下一步

让本地 AI 使用 Eventhouse Remote MCP，执行 [**Aggregation Follow-up（完整地址及输出契约）**](kql/poc-parser-fix-20261117/controlled-rollout/AGGREGATION_FOLLOWUP_AI_PROMPT.md)。先跑只读 [Activity Gate](kql/poc-parser-fix-20261117/controlled-rollout/09-aggregation-activity-gate.kql)；若有部署后 Aggregation 新样本，再用**同一 Patch UTC**执行 06/07/08，逐字段核对 6 个 Duration 并提供脱敏 Markdown 报告。

**禁止**重复执行 Aggregation Candidate、自动回滚、修改 Mashup、触碰 PROD。确认出现摄入失败或数值错误时再人工评审回滚；无样本时仍保持 Inconclusive。不要以猜测的固定“30–60 分钟”作为验收依据。

[GW-001](tracking/ISSUES.md) · [Changelog](tracking/CHANGELOG.md)。
