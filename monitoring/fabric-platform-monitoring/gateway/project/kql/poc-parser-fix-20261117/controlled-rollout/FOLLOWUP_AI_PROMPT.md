# 给本地 AI 的后续只读检查（已部署 Streaming，但首轮无新事件）

> **目标：** 确认 Streaming POC Function 的实际 Body 是否为 GitHub Candidate；确定 Gateway 其它类型是否继续送达、Streaming 是否出现新样本，并评估两项 Duration 是否恢复。**不得修改任何 Eventhouse 对象或 Uploader。**

## GitHub 来源

- 仓库：https://github.com/ZeonZheng/fabric-toolbox
- **指定分支：** `docs/gateway-monitoring-poc-20261112`（不得静默切换 main）。
- 本目录：https://github.com/ZeonZheng/fabric-toolbox/tree/docs/gateway-monitoring-poc-20261112/monitoring/fabric-platform-monitoring/gateway/project/kql/poc-parser-fix-20261117/controlled-rollout
- 已执行的 Candidate：https://github.com/ZeonZheng/fabric-toolbox/blob/docs/gateway-monitoring-poc-20261112/monitoring/fabric-platform-monitoring/gateway/project/kql/poc-parser-fix-20261117/candidates/2-parse_QueryStreamingReport.kql

## 操作方式

1. 仅使用你已连接的 `eventhouse-remote` MCP **只读工具**，核实用户所指的 **POC** Gateway Monitoring KQL Database，输出数据库**名称**，不回传真实数据库 GUID。
2. 获取当前 `.show functions | where Name == "parse_QueryStreamingReport" | project Name, Body`，在本地 AI 环境中与上述 GitHub Candidate 的 Function Body 用同一编码、同一空白规范化规则做文本/Hash 对照。**只输出 `BodyMatchesCandidate: true/false/NotVerified`、比较方式、Hash（若可以）**；不要贴 Body 全文。不应该只比较新/旧 Hash 是否不同。
3. 单独运行本目录两份只读查询：
   - [04-streaming-activity-snapshot.kql](04-streaming-activity-snapshot.kql)：全部 Raw logType 的 Envelope 数、logDate 超过 2026-11-20T06:19:18.0313089Z 的消息数及最新 Envelope 时间；
   - [05-streaming-parsed-field-recovery.kql](05-streaming-parsed-field-recovery.kql)：整个 Parsed `QueryStreamingReport` 的非空指标数，不用业务时间做严格筛选。
4. 如果 Streaming Raw 现在出现 Patch 后 logDate 的样本，再依照 [POSTCHECK_AI_PROMPT.md](POSTCHECK_AI_PROMPT.md) 重跑原来的 01/02/03（采用**同一个 Patch UTC**）。如果没有新 Streaming，且其它 Raw 类型有活动，报告 Streaming 稀疏/未上报；如果全部类型都没有活动，报告需要检查 Uploader/源文件（不要自动重启服务或修改 Task）。若新增 Update Policy ingestion failures，按 `Blocked` 处理并报告，不能自行回滚。
5. 输出一份中文 Markdown `gateway-streaming-followup-YYYYMMDD-HHmm-SGT.md`，包含：目标 POC、实际执行时间（UTC+8）、Candidate Body 对照结果、每类 Raw 活跃度表、Parsed 总数与非空率、新 Streaming Raw/Parsed 及匹配/失败的脱敏结果、`Passed/Blocked/Inconclusive`、未验证项。**附真实查询结果计数**，不贴敏感明文。

## 判定提醒

- 2026-11-20 14:22（UTC+8）首次 Postcheck：Raw/Parsed Patch 后均 **0**，因此 Inconclusive。此后如果仍没有新 Streaming 数据，**不能声称通过**。
- 即使 Parsed 全表首次出现非空值，仍需尽可能核对 Raw 值和 Parsed 对应值；请求 ID 可能重用，Raw 与 Parsed 时间字段语义不同，不能把 KeyJoin 数量差直接作为丢失证据。
- `HTTP 201` 和 `0 ingestion failures` 不证明端到端无丢失/无重复。
- **禁止** DDL、回滚、对 PROD 的任何操作。
