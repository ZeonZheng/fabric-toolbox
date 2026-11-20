# Streaming Parser｜2026-11-20 部署后第一次独立复核

> **执行来源：** 用户在 POC Fabric UI 手工部署 Streaming Function，随后用本地 `eventhouse-remote` MCP 只读 Postcheck；[脱敏原始报告](gateway-streaming-postcheck-20261120-1422-SGT.sanitized.md)。  
> **记录时间：** 2026-11-20 14:22（Asia/Singapore）；实际 `RecordedAfterPatchUTC` = **2026-11-20T06:19:18.0313089Z**。  
> **Verdict：Inconclusive（无修复后新 Streaming 事件）**。**不能称为修复成功，也没有故障证据要求立即回滚**。

## 实测与判定

| 检查 | 结果 | 结论 |
| --- | --- | --- |
| Function 存在 | 是；现场 Body hash=`587bba1a7ad9dac82c873b8a2ed4ef5ef6cc15a5b8cd3ef3e62df13b810703ac`；旧 Preflight hash=`d946208e13a5009a2695a8b52f6ce0c8ed1d2e46d8d148cb069b826bc34f805c` | **报告显示变更迹象**。但 Preflight/后测 hash 计算口径可能不同，尚缺实际 Body 与目标 Candidate 的同口径文本比较 |
| Update Policy | Enabled=true，Source=`GatewayReports-Raw`，IsTransactional=true | 配置保持正常 |
| Patch 后 Raw | **0 行**；Raw logDate 最新 2026-11-20T06:11:46Z | 无新的 Streaming 样本，尚不能验证 Parser 行为 |
| Patch 后 Parsed | **0 行**；Parsed 最新摄入 06:11:47Z | 两项 Duration 尚未获得回归证据 |
| KeyJoin | 0 Raw Keys，0 matched，0 mismatch | 无统计意义，不是通过 |
| Post-Patch ingestion failures | **0 observed** | 观察范围内暂无已知事务性失败，不能排除低频日志/上游采集问题 |

## 风险与建议

1. **主要阻碍：样本不足，而非已发现坏数据**。Streaming 在 Patch 前也不连续；修改后约几分钟内无数据，不能证明摄入停止，也不能证明修复成功。
2. **不要因为空样本而立即回滚，也不要继续改 Aggregation / Mashup**。先获取 Patch 后的 Streaming Raw 事件，同时检查其它 logType 的近期记录；若其它类型持续而 Streaming 不出现，更倾向 Streaming 上游事件低频/未产出（需要 VM 证据确认）。
3. **注意时间过滤语义：** `Raw.logDate` 和 `Parsed.StreamingStartTimeUTC` 是不同时间；如果新摄入老业务时间的事件，原 Postcheck 可能遗漏。新查询因此补充 **Parsed 全表非空率**，不依赖业务时间过滤。历史 Preflight Streaming 两项均 100% null，所以新出现的非空值能支持修复生效，但仍需核对实际内容/时序。
4. **Body hash 不同不足以证明确实安装了 GitHub Candidate**。下一轮需基于实际 live Function Body 和 GitHub Candidate 相同规范化规则比较，并输出布尔结果/Hash，不输出全文。
5. Raw 保留期 1 天，不能拖延历史样本的取证；若出现新摄入 failures，需要立即分析回滚条件。此次仍与完整上传/去重、PROD Ready 无关。

**新验证脚本：** [事件活跃度](../../kql/poc-parser-fix-20261117/controlled-rollout/04-streaming-activity-snapshot.kql) 与 [Parsed 全表非空](../../kql/poc-parser-fix-20261117/controlled-rollout/05-streaming-parsed-field-recovery.kql)。[本地 AI 交接](../../kql/poc-parser-fix-20261117/controlled-rollout/FOLLOWUP_AI_PROMPT.md)。
