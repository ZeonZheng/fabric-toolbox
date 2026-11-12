# Gateway Monitoring｜架构、日志生命周期与后续技术决策

> **核心决策：当前先保留已验证的 Fabric 端 + Heartbeat / NodeInfo，暂停原始 Uploader 对活动 Gateway 日志的持续运行。**  
> 后续重点不是重做 Report，而是验证一个**不会影响 Gateway 原始日志**的采集方案。

[返回总览](../POC_README.md) · [部署指南](DEPLOYMENT_RUNBOOK.md) · [现状与 Backlog](POC_STATUS.md)

## 1. 已验证的架构与边界

- [可编辑的 POC Draw.io 架构图](Gateway-Monitoring-POC.drawio)（绿色：已验证；橙色：禁用/待验证；虚线：候选方案）
- [Fabric Jumpstart 部署介绍](https://jumpstart.fabric.microsoft.com/catalog/fpm-gateway-monitoring/)
- [原 Toolbox 数据流示意图](../../Images/05%20-%20Gateway%20Report%20Data%20Logic.png)

```text
On-premises Gateway VM
  │
  ├─ Get-DataGatewayInfo.ps1 ────> GatewayMonitoringReports.Eventstream
  ├─ Run-GatewayHeartbeat.ps1 ───> GatewayMonitoringHeartbeat.Eventstream
  └─ Run-UploadGatewayLogs.ps1 ──> GatewayMonitoringReports.Eventstream
       （仅对静态日志副本完成测试）
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

**真正投入后台的只有 Heartbeat（30 秒）和 NodeInfo（开机/每周）任务。** UploadLogs 创建过 Task，但基于文件安全考虑已 **Disabled**。

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

**结论：** 不能假设这些 `*Report_*.log` 是已经关闭的静态日志，或 Gateway 每天都会创建新日期文件。若外部 `Move-Item` 把活动文件搬走：

- 可能因文件被锁定而失败；
- 也可能遇到继续写入旧句柄、重建、延迟轮转等行为；
- **当前没有测试证明 Gateway 一定会自动生成新文件，也没有证明日志写入完全不受影响**。

因此，目前禁止把原始 Uploader 的 `Config.json` 直接指向实时源文件并启用 24×7 Task Scheduler。

## 4. 候选技术路线（讨论过，但均未实施）

| 技术路线 | 优点 | 仍需验证 / 不足 |
| --- | --- | --- |
| **原始 Uploader 直接 Move** | 代码最少；静态副本 POC 已成功 | 活动文件移动安全性、锁定、故障恢复，**不建议直接生产** |
| 周期性 `Copy-Item` 整份文件 | 简单 | Gateway 文件持续追加，会反复上传旧记录；**不建议** |
| **Read-only Tail + Checkpoint**（PowerShell/C#） | 源目录只读、准确增量、REST 443 | 需要处理 CSV 跨行、字节偏移、文件身份、轮转、断点、重试、去重 |
| Fluent Bit / Vector / OpenTelemetry Collector | 文件 Tail 和 Checkpoint 更成熟 | 不意味着可以直接解析 Gateway 复杂 CSV 或原样接 Fabric 的 JSON Schema |
| 已有企业级日志平台 | 可复用部署/安全/运维标准 | 先确认组织是否已部署，以及成本与接入方式 |
| Kafka Broker | 提供消息流中转 | **不负责读取 Windows 文件**，引入额外端口/基础设施；目前没必要 |

**优先评估顺序：** 组织现有日志平台 → Fluent Bit/Vector 等成熟 Agent 的实际可用性 → 轻量 C# Worker → 自研 PowerShell Collector。不要在未验证 CSV 和 Fabric 输出前就做技术承诺。

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
| Gateway-UploadLogs | **Disabled**（只做过测试） | 未确定安全的增量采集手段 |

原 Toolbox XML 模板含旧机器 SID、旧绝对路径、`PT72H` 的执行时限，因此当前采用 Task Scheduler GUI 手动创建任务，并取消长任务时间限制。Windows 后台执行不需要打开 PowerShell 窗口；但必须定期监控 **KQL 最后心跳时间，而不是只看 Task State=Running**。

## 6. 安全和生产化 Checklist

- [ ] 原始日志内容/来源权限审查：QueryText、数据源 URL、错误详情可能含敏感信息。
- [ ] 本地 `Config.json` 与 Eventstream SAS/Client Secret 仅存于授权 VM，不随 Git 分发；跨 VM 单独初始化 DPAPI。
- [ ] 网关技术账户和 Scheduler 运行身份按最小权限部署；保护日志、checkpoint、归档 ACL。
- [ ] 断网、Restart、半写入 CSV、源文件轮转、消息重发、去重、容量/成本都完成验收。
- [ ] Eventhouse Retention、KQL Update Policy 和 Model 的多 Node 聚合准确性验证。
- [ ] 定义企业监控告警、值班、失败重试和回滚方案。

**结论：** 当前 POC 的 **Fabric 后端与报表链路已验证**，真正的后续工作是 Gateway 原始日志的**安全持续采集**。本分支不含未经验证的生产 Collector。
