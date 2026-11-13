# Gateway Monitoring｜架构、日志生命周期与后续技术决策

> **决策更新（2026-11-13）：** 已在 POC 单节点验证真实 Performance Logs Move 后自动重建并继续写入，原始 Uploader Task 已启动连续观察。**仍未经生产可靠性验收。**  
> **下一步：** 验证多轮完整性、断网/部分上传后的恢复与资源开销；若原始 Uploader 不达标，再评估只读 Tail/Checkpoint 或企业日志平台。

[返回总览](../POC_README.md) · [部署指南](DEPLOYMENT_RUNBOOK.md) · [现状与 Backlog](POC_STATUS.md)

## 1. 已验证的架构与边界

- [可编辑的 POC Draw.io 架构图](Gateway-Monitoring-POC.drawio)（**2026-11-12 状态快照**；图中 UploadLogs Disabled 信息已过时，2026-11-13 状态请以下文为准）
- [Fabric Jumpstart 部署介绍](https://jumpstart.fabric.microsoft.com/catalog/fpm-gateway-monitoring/)
- [原 Toolbox 数据流示意图](../../Images/05%20-%20Gateway%20Report%20Data%20Logic.png)

```text
On-premises Gateway VM
  │
  ├─ Get-DataGatewayInfo.ps1 ────> GatewayMonitoringReports.Eventstream
  ├─ Run-GatewayHeartbeat.ps1 ───> GatewayMonitoringHeartbeat.Eventstream
  └─ Run-UploadGatewayLogs.ps1 ──> GatewayMonitoringReports.Eventstream
       （POC 真实源文件已首轮验证，后台持续采集观察中）
                                         │
                                         ▼
                               Fabric Eventhouse / KQL
                                         │
                                         ▼
                              Gateway Semantic Model
                                         │
                                         ▼
                              Power BI Gateway Monitor
```

**2026-11-13 任务状态：** Heartbeat `Running`（30 秒）、NodeInfo `Ready`（At startup + weekly 的一次性任务）、UploadLogs `Running`（60 秒轮询，私有 `Config.live-poc.json`）。这不是持续数据完整性验收的替代证据。

## 2. 为什么原始 Uploader 不适合作为“只读 Tail”？

源码：[UploadGatewayLogs.psm1](https://github.com/microsoft/fabric-toolbox/blob/main/monitoring/fabric-platform-monitoring/gateway/PowerShellScript/modules/UploadGatewayLogs.psm1) · [Utils.psm1](https://github.com/microsoft/fabric-toolbox/blob/main/monitoring/fabric-platform-monitoring/gateway/PowerShellScript/modules/Utils.psm1)

### 2.1 Uploader 扫描哪些文件？

它不是只处理 `QueryStartReport`。原始筛选概念上是：

```powershell
Get-ChildItem -File -Path "$path\*Report_*.log" -Exclude "*_f.log" -Recurse
```

POC 的真实 Gateway `Report` 目录发现以下八类日志：

| 类别 | 主要内容 |
| --- | --- |
| QueryStartReport | 查询开始、跟踪 ID、数据源、可能包含 QueryText |
| QueryExecutionReport | 执行结束、Duration、结果、Error |
| QueryExecutionAggregationReport | 查询聚合指标 |
| SystemCounterAggregationReport | CPU、Memory / System Counters |
| MashupEvaluationReport | Mashup 引擎活动 |
| ConcurrentOperationAggregationReport | 并发运行统计 |
| OpenConnectionReport | 数据连接活动 |
| QueryStreamingReport | 查询数据流统计 |

上传配置打开时，**所有符合文件名条件的报告都可能被处理**；并不等于每类都在 Power BI Report 中有独立页面。普通 Gateway `GatewayErrors*.log` 等服务日志由另一条 Lakehouse UploadLogs 可选流程负责。

### 2.2 它具体把文件放在哪里？

原 Uploader 操作的是文件：

```text
<Gateway Log Root>\Report\*Report_*.log
         │
         │ Move-Item（不是 Copy-Item）
         ▼
<Gateway Log Root>\Report\Temp\*_x.log
         │
         │ Import-Csv → JSON 分批 → Eventstream HTTP POST
         ▼
<Gateway Log Root>\Report\History\*_f.log
         └─ 合并归档、清理 Temp；旧 History 按保留策略清理
```

**POC 历史副本测试的位置：**

```text
D:\Ang\FabricGatewayMonitoring\poc-input\Report\
├── Temp\
└── History\
```

**真实 Gateway Source 路径（示例，仅为 POC VM）：**

```text
C:\Users\powrbi\AppData\Local\Microsoft\On-premises data gateway\Report\
```

原脚本不是在自定义 `D:\Ang\FabricGatewayMonitoring\logs` 目录内归档；后者主要用于脚本报错日志。

### 2.3 Retention 的定义不要混淆

| 配置 | 实际含义 |
| --- | --- |
| Gateway 的 `ReportFileSizeInBytes` | Gateway 自己的性能日志大小轮转设置（POC 约 100 MiB） |
| Gateway 的 `ReportFileCount` | Gateway 自己的每类日志文件数（POC 为 10） |
| Uploader 的 `ReportRetention = 10` | 本地 `History` 归档约 10 天的清理策略；清理触发于处理流程，不等于独立每日任务 |
| Eventhouse Retention | **单独配置**；与前两个值无直接继承关系 |

## 3. 直接证据：Gateway 真的在长期追加写同名文件

2026-11-12 实测：`SystemCounterAggregationReport_RNGVMC0046_20261019T110120.log` 由 **6,668,473 → 6,670,766 bytes**；另有 QueryExecutionAggregation、ConcurrentOperationAggregation 文件增长。同名文件在 10 月 19 日起始，但 11 月 12 日仍更新。

**2026-11-13 结果更新：** 在本台 POC Gateway 上，8 个活动日志被原始 Uploader 成功 Move，Eventstream 返回 HTTP 201；Gateway 之后重新创建同名 Performance Logs 并继续写入。文件名仍含旧的 `20261019T110120`，但物理创建时间为 2026-11-13。**仍不能证明无丢失/重复、跨机器行为一致或故障恢复可靠**。

### Uploader 的上传与本地合并机制（源码核对）

1. 扫描匹配 `*Report_*.log`，排除 `*_f.log`，**不以 LastRun/LastWriteTime 增量过滤 Performance Report**。
2. 对每个源文件先 `Move-Item` 至 `Report/Temp/*_x.log`，从 Temp 读取 CSV 并按消息大小分批 POST 至 Eventstream；响应为 HTTP 201 后继续处理。
3. **上传在 Merge 之前**；`Merge-ReportFiles` 再将 Temp 中的 CSV 行追加至本地 `Report/History/*_f.log`，成功后删除 Temp；History 不是再次上传的源。
4. “同类型”取文件名第一个 `_` 前的前缀（例如 `QueryStartReport` 和 `SystemCounterAggregationReport` 独立）。在 History 中找**当天创建、低于 100 MiB** 的 `$report*_f.log` 文件追加；找不到就在**同一个 History 文件夹**新建一个归档文件，**没有按天建子目录**。大小条件在追加前评估，追加后可能超过阈值。
5. **两套独立大小管理：** Gateway `GatewayCore.dll.config` 设置 `ReportFileCount=10`、`ReportFileSizeInBytes=104857600`（100 MiB），控制原始日志；Uploader 的 History `100mb` 是 `Utils.psm1` 写死的本地合并条件，二者彼此独立。Gateway 文件名旧时间戳可持续沿用，不能作为事件时间。
6. **多节点：** 官方按每个 Gateway VM 独立部署 uploader/本地 Report/Temp/History，正常部署时 History 不跨节点合并。若人为改为共享 History，代码未额外检查节点 ID，不能假定安全。
7. **仍需验证：** 分批上传在部分成功后失败，可能重复；递归扫描包含 Temp，重试中对同一路径 Move 的行为尚未测试；History 归档成功也不证明 Eventhouse Parsed 与源端记录 exactly-once。

## 4. 当前原始 Uploader POC 与其他候选技术路线

| 技术路线 | 优点 | 仍需验证 / 不足 |
| --- | --- | --- |
| **原始 Uploader 直接 Move** | 真实 POC 单节点已验证首轮 Move + 自动重新生成日志，并启动后台持续观察 | 多轮数据完整性、部分失败/重试恢复及 PROD 影响尚需验证；**未批准直接生产推广** |
| 周期性 `Copy-Item` 整份文件 | 简单 | Gateway 文件持续追加，会反复上传旧记录；**不建议** |
| **Read-only Tail + Checkpoint**（PowerShell/C#） | 源目录只读、准确增量、REST 443 | 需要处理 CSV 跨行、字节偏移、文件身份、轮转、断点、重试、去重 |
| Fluent Bit / Vector / OpenTelemetry Collector | 文件 Tail 和 Checkpoint 更成熟 | 不意味着可以直接解析 Gateway 复杂 CSV 或原样接 Fabric 的 JSON Schema |
| 已有企业级日志平台 | 可复用部署/安全/运维标准 | 先确认组织是否已部署，以及成本与接入方式 |
| Kafka Broker | 提供消息流中转 | **不负责读取 Windows 文件**，引入额外端口/基础设施；目前没必要 |

**当前顺序：** 先验证原始 uploader 的 POC 连续运行；如果出现完整性/恢复问题，再评估组织现有日志平台 → Fluent Bit/Vector → C# Worker → PowerShell Tail。候选方案尚未投入。

### Read-only Tail 的设计验收点

1. **FileStream shared read**：只读共享打开，允许 Gateway 持续写入；文件句柄共享模式需要 VM 实测。
2. **CSV 完整逻辑记录**：不能只按换行拆记录（`QueryText` 可能引号包裹、跨行）；处理半写入记录和编码边界。
3. **按文件 Byte Offset Checkpoint**：必须只在完整批次发送收到确认后推进。
4. **轮转/截断检测**：通过稳定 File Identity（不能只用路径和日期）识别文件替换；不能静默丢弃旧 Checkpoint 的未确认记录。
5. **HTTP REST Batch**：保持目前已验证的 `{ logType, logDate, log } ` 消息 Schema、发送大小限制及重试规则。
6. **At-least-once / EventId**：HTTP 201 证明 Eventstream 接收，不保证 Exactly-once。即使给原始 dynamic JSON 添加 `EventId`，当前 KQL Parsed Tables 也**不会自动去重**；需要独立实现并测试。
7. **运维**：独立 Collector 健康、磁盘 backlog、故障恢复、任务与身份、敏感数据与网络控制。

## 5. Task Scheduler 的长期运行要求

| Task | 本次策略 | 原因 |
| --- | --- | --- |
| Gateway-Heartbeat | **Enabled**，30 秒，持续后台运行 | KQL `Responding` 已验证 |
| Gateway-NodeInfo | **Enabled**，开机 + 每周执行一次 | Node 元数据变化频率低 |
| Gateway-UploadLogs | **Enabled / Running（2026-11-13 POC）** | Action 改指向 `Config.live-poc.json`；仍在连续运行观察期 |

原 Toolbox XML 模板含旧机器 SID、旧绝对路径、`PT72H` 的执行时限，因此采用 Task Scheduler GUI 管理任务。2026-11-13 通过 GUI 更新 UploadLogs 的 Action（PowerShell `Set-ScheduledTask` 更新时遇到身份凭据错误），并启用。Windows 后台执行不需要打开 PowerShell 窗口；但必须定期监控 **KQL 最近 Heartbeat 与 Performance Logs、而非只看 Task State=Running**。

## 6. 安全和生产化 Checklist

- [ ] 原始日志内容/来源权限审查：QueryText、数据源 URL、错误详情可能含敏感信息。
- [ ] 本地 `Config.json` 与 Eventstream SAS/Client Secret 仅存于授权 VM，不随 Git 分发；跨 VM 单独初始化 DPAPI。
- [ ] 网关技术账户和 Scheduler 运行身份按最小权限部署；保护日志、checkpoint、归档 ACL。
- [ ] 断网、Restart、半写入 CSV、源文件轮转、消息重发、去重、容量/成本都完成验收。
- [ ] Eventhouse Retention、KQL Update Policy 和 Model 的多 Node 聚合准确性验证。
- [ ] 定义企业监控告警、值班、失败重试和回滚方案。

**结论：** 原始 Uploader 已在 POC 首轮真实日志验证并开始持续运行，Fabric 后端与报表链路已有静态副本证据。当前必须核验连续性、异常恢复、端到端行数和 PROD 首次导入范围（使用独立 Staging，按事件时间而非文件名日期筛选）；不声明 Production Ready。
