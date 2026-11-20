# 证据索引（Evidence Register）

> 证据按**实际采集日期**归档，不能用这里的旧快照判断当前运行状态。当前进展见 [STATUS](../STATUS.md)，问题追踪见 [ISSUES](../tracking/ISSUES.md)。

| 日期 | 内容 | 用途 |
| --- | --- | --- |
| [2026-11-17 基线](2026-11-17/README.md) | 源码 Schema、15 个实际 Function Body、142 行 Schema、Policy 与 Raw 字段清单、MCP Overview JSON / Markdown（脱敏） | 回滚前的定义比对、架构与数据质量审计 |
| [2026-11-17 Dry-run + Schema + Value](2026-11-17/DRYRUN_RESULTS.md) | 用户的 3 个 Dry-run、3 份 Schema CSV 及一份 Value CSV | Dry-run 3/3、Schema 39/39、13 数值项无丢失/超阈值差异；DataSource 235/235 内容一致；Mashup Kinds/Paths 在本次 Raw 中无对应 Key |
| [2026-11-20 MCP Preflight](2026-11-20/README.md) | 本地 AI 执行的脱敏结果与独立技术评审，含 Function SHA256 / Policy / Raw-Parsed / failures | 4/4 通过，**只可进入人工 Patch 评审**；额外发现 Mashup TotalBytes、Streaming 缺字段等问题 |
| [历史 POC 测试](../docs/POC_STATUS.md) | 2026-11-12～11-17 多轮测试细节 | 补充历史环境、已知问题和数据流证据 |

- **2026-11-20 Streaming Postcheck：** [首次 Inconclusive → Follow-up Passed](2026-11-20/README.md)。新 Parsed 32/32 双 Duration 恢复，Raw 对照 32/32 匹配；24h/端到端完整性仍未验收。

**安全与准确性：** 这些是导出/记录快照，**不是能够恢复的 Eventhouse 物理数据库备份**。不上传 QueryText、原始 Gateway CSV、SAS、Secrets 或未经脱敏的环境标识。
