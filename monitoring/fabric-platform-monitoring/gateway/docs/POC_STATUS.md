# Gateway Monitoring｜POC 进展与验证证据

> **记录截至 2026-11-12。** 这是一次 Windows On-premises Data Gateway **单 Cluster、单 Node** POC 的实测结果；不能代表所有 Gateway 节点或 Fabric Platform Monitoring 模块。

[返回总览](../POC_README.md) · [继续部署](DEPLOYMENT_RUNBOOK.md) · [已发现问题](TROUBLESHOOTING.md)

## 一页结论

**已完成：** 从 Gateway VM 到 Eventstream、Eventhouse/KQL、Semantic Model、Power BI Report 的核心功能链路；NodeInfo / Heartbeat 通过 Scheduled Tasks 后台执行。  
**尚未完成：** 真实日志文件的安全持续采集、Running Jobs / Job Details、多个 Cluster/Node、长期稳定性与生产治理。  
**当前操作限制：** `Gateway-UploadLogs` 保持 **Disabled**，只用日志**副本**完成过数据上传验证。

## 1. 验证矩阵

| 功能 | 实测结果 | 状态 |
| --- | --- | --- |
| Jumpstart Fabric Items | 两个 Eventstreams、Eventhouse/KQL、Semantic Model、Report 已创建并可用 | ✅ |
| Notebook | 已在 POC 中完成配置生成；但 fork 里又做了兼容性修正，**提交版尚未重新运行** | 🟡 |
| PowerShell 环境 | VM 上 PowerShell 7.6.9，四个依赖模块可见 | ✅ |
| Gateway Performance Reports | 服务账户目录中存在 **8 类性能报告**，无需 Additional logging | ✅ |
| SPN Secret | DPAPI 解密成功，长度与再次粘贴的 Secret 完全匹配 | ✅ 本机验证 |
| SPN Gateway 权限 | 添加 Gateway Cluster Admin 后可见 Cluster **0 → 1**、匹配 Node **0 → 1** | ✅ |
| Gateway Node Metadata | Eventstream HTTP 201，KQL 中 Cluster/Node/Version 完整 | ✅ |
| Heartbeat | Eventstream HTTP 201，KQL 有 `Responding=true` 的实时记录 | ✅ |
| Heartbeat Scheduled Task | `Running`、`LastTaskResult = 267009`（正在运行） | ✅ |
| NodeInfo Scheduled Task | 执行后 `Ready`、`LastTaskResult = 0`（成功完成） | ✅ |
| Performance Logs 上传 | 四类**复制的历史日志**上传、Raw 和解析表均有数据 | ✅ 静态副本 |
| Gateways / System Counters | Cluster/Node、CPU/Memory 趋势可见 | ✅ |
| Jobs / Queries | 补充 QueryStart 和 QueryExecution 后在历史日期出现数据 | ✅ |
| Running Jobs / Job Details | 未展示端到端验证证据 | ◻️ 未验证 |
| 活动日志的长期采集 | 原始 Uploader 会移动文件；未持续运行真实源目录 | ❌ |
| 多 Node / 多 Cluster | 未测试 | ◻️ |
| Capacity Events / Activity Events / Inventory | **未参与此次 POC** | ◻️ |

## 2. Raw → Parsed：已核对的数据量

以下是历史副本上传后的结果，**不是持续运行的统计**。

| 类型 | Eventstream 消息批次 | Raw 日志记录数 | KQL 解析后记录数 |
| --- | ---: | ---: | ---: |
| QueryExecutionAggregationReport | 2 | 1,306 | 1,306 |
| SystemCounterAggregationReport | 27 | 53,538 | 53,538 |
| QueryExecutionReport | 3 | 2,055 | 2,055 |
| QueryStartReport | 7 | 2,055 | 2,055 |

首次两类聚合日志共发送 **29 批 / 54,844 条**，全部对应到解析表。随后补充查询明细，使 Jobs / Queries 页面可用。

**不要混淆：** HTTP `201 Created` 仅证明接收端接受消息；Raw 和解析后的计数一致，是另一层独立验证。它仍不能证明未来持续采集会 Exactly-once。

### 复查 KQL（可直接复制）

```kusto
['GatewayReports-Raw']
| where logType in ("QueryStartReport", "QueryExecutionReport",
                    "QueryExecutionAggregationReport", "SystemCounterAggregationReport")
| summarize Messages=count(), Rows=sum(array_length(log)) by logType
```

```kusto
GatewaysHeartbeat
| summarize LastSeen=max(ingestion_time()),
            EventsLast5Min=countif(ingestion_time()>ago(5m))
    by GatewayId
| extend AgeSeconds=datetime_diff("second", now(), LastSeen)
```

## 3. 报表验收详情

| 页面 | 实测观察 |
| --- | --- |
| Gateways | 显示 POC Gateway Cluster、Node、Gateway 版本、逻辑核心数、内存、OS |
| System Counters | System CPU、Gateway CPU、System Memory、Gateway Memory 的历史趋势正常 |
| Jobs | 将日期选到 **2026-10-19**，显示 **3 Jobs、1 Node、5 Queries、P90 12.44 秒** |
| Queries | 看到 **5 Queries、3 Jobs** 和数据源分类（如 ODBC / SQL / Web） |
| Running Jobs / Job Details | 未提供测试结果，不能标记完成 |

**重要经验：** 日期过滤使用**日志事件时间**，而不是上传到 Eventhouse 的日期。此次如果只看 2026-11-11，会误以为 Jobs / Queries 没有数据。Model 参数配对后还必须执行 **Edit credentials**。

## 4. 本机文件持续写入：直接实测

在 2026-11-12 多次检查，文件名保持 `20261019T110120.log`，但文件大小不断增长。

| Report | 第一次（bytes） | 最后一次（bytes） | 差值 |
| --- | ---: | ---: | ---: |
| SystemCounterAggregation | 6,668,473 | 6,670,766 | **+2,293** |
| QueryExecutionAggregation | 561,365 | 563,900 | **+2,535** |
| ConcurrentOperationAggregation | 374,348 | 374,568 | **+220** |
| QueryStart | 4,482,959 | 4,482,959 | 无变化 |

这证明至少有部分报表正在**以同名文件继续追加**。并没有证明被外部移动后会如何重建，因此不能安全假设 Uploader 可以直接 `Move-Item` 活动文件。

## 5. 保留的环境说明（仅用于解释此次 POC）

- 网关展示名：`POC Data Gateway`；Node：`Power BI Gateway POC`。
- Gateway 版本：`3000.338.2`。
- Gateway 服务以技术账号运行；实际 Report 根目录位于该账号的用户 Profile 下，而不是默认 PBIEgwService 路径。
- 曾在 Heartbeat 刚启动时看到 `AgeSeconds = 17` 和最近 5 分钟 `7` 条记录；这不是连续五分钟稳定运行的基准值。
- 为安全起见，**没有把 Tenant ID、App ID、Gateway ID、Client Secret、Eventstream Key 或原始日志上传到仓库**。

## 6. 下次续建 Backlog

- [ ] 对比企业现有日志采集平台与只读 Tail Collector（Fluent Bit / Vector / C#），选型前做 POC。
- [ ] 验证所有 8 类 CSV（含引号、多行 QueryText、UTF-8、文件截断/轮转）。
- [ ] 验证增量 Offset/Checkpoint、批量发送、重试、恢复、EventId 去重。
- [ ] 在不移动 Gateway 原始日志前提下，完成 UploadLogs 后台长期运行。
- [ ] 增加 Collector 健康、任务失败、磁盘积压、消息延迟的告警。
- [ ] 验证 Running Jobs、Job Details、多 Gateway Nodes/Clusters 和最终数据完整性。
- [ ] 复核网络、Service Account 权限、敏感 QueryText、Eventhouse Retention 与 Fabric 成本。

**当前不应称为“Production Ready”**。本页记录的是已完成的功能性 POC，而不是完整生产验收。
