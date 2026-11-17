# Gateway Monitoring｜部署与续建 Runbook

> **适用范围：** Windows On-premises Data Gateway（单 Node POC）。本指南将**操作、预期结果、常见错误**放在同一阶段。  
> **已验证：** Metadata / Heartbeat / 历史日志副本 / Report；2026-11-13 真实 POC 日志首次 Move → Eventstream → History，Gateway 重新创建文件继续写入。  
> **进行中：** `Gateway-UploadLogs` 已切换 `Config.live-poc.json` 并 Running；多日连续性、失败恢复和 PROD 安全性未完成验收。

[返回导航](../POC_README.md) · [故障速查](TROUBLESHOOTING.md) · [POC 验证数据](POC_STATUS.md)

## 快速施工进度

| 阶段 | 操作 | 本次 POC |
| --- | --- | --- |
| 1 | Fabric Jumpstart 部署 Gateway 模块 | ✅ |
| 2 | Setup Notebook 生成配置 | ✅；提交版 Notebook 的新增兼容修改待重测 |
| 3 | Gateway VM 准备 PowerShell / 日志路径 | ✅ |
| 4 | 初始化、加密并验证 SPN Secret | ✅ |
| 5 | NodeInfo + Heartbeat 上传、KQL 验证 | ✅ |
| 6 | 静态副本上传 + 8 类真实 POC 日志首次 Move | ✅ 首轮通过 |
| 7 | Semantic Model + Credentials + Report | ✅ |
| 8 | Task Scheduler 后台运行 | ✅ Heartbeat Running / NodeInfo Ready；🟡 UploadLogs Running（观察中） |
| 9 | 长期连续性、失败恢复及 PROD 首次采集 | 🟡 后续验收 |

## Phase 1 — Fabric 端部署

### Step 1. 在目标 Fabric Workspace 安装 Gateway 模块

可参考 [Fabric Jumpstart 官方页面](https://jumpstart.fabric.microsoft.com/catalog/fpm-gateway-monitoring/)；在该 Workspace 的 Fabric Notebook 中执行（根据租户环境安装依赖）：

```python
%pip install -q fabric-jumpstart
import fabric_jumpstart as jumpstart

jumpstart.install("fpm-gateway-monitoring")
```

**检查以下 Fabric items 是否已创建：**

- `GatewayMonitoringHeartbeat` Eventstream
- `GatewayMonitoringReports` Eventstream
- `Fabric Platform Monitoring` Eventhouse、`Gateway Monitoring` KQL Database
- `Gateway Monitoring` Semantic Model、Power BI Report、Setup Notebook

> **注意：** 用户上传的 Installer Notebook 曾有安装冲突/失败输出。它只是一份**使用与重试记录**，不是证明每次安装都成功。不要把 `update_existing=True` 当作首次部署默认选项：它可能覆盖已经调整过的 Fabric items。

### Step 2. 设置 SPN 权限

按 [Jumpstart 权限要求](https://jumpstart.fabric.microsoft.com/catalog/fpm-gateway-monitoring/) 配置 Entra App、Fabric Tenant Settings、Workspace 权限。

**尤其注意：将监控 SPN 加入目标 On-premises Gateway Cluster 的 Admin。** 这不是 Fabric Workspace 的 Admin/Member 所能替代的权限。本次初始缺少 Gateway Admin，导致 NodeInfo 上传成功但 Cluster/Node 字段为 `null`。

## Phase 2 — 生成和检查 Config.json

### Step 1. 运行适配后的 Setup Notebook

使用 [poc-notebooks/Gateway Monitoring Setup.ipynb](../poc-notebooks/Gateway%20Monitoring%20Setup.ipynb)。

- 在参数区填入当前 **Tenant ID / App (Client) ID**，不要沿用 POC 环境值。
- 使用 Fabric REST API 读取两个 Eventstream 的 CustomEndpoint 配置，生成 `builtin/config.json`。
- 提交版额外加了 `ServicePrincipal.TenantId`，同时保留原始拼写 `TennatId`；参考参数为 **Heartbeat = 30 秒、Report Send = 60 秒**。
- **本次提交后的 Notebook 尚未在 Fabric 中重新执行。** 若失败，优先查看 HTTP 状态、Eventstream Source 类型及 Workspace 上下文。

生成后检查结构，不要公开实际内容：

```json
{
  "GatewayLogsPath": ["<GATEWAY_SERVICE_ACCOUNT_LOG_ROOT>"],
  "HeartbeatInterval": 30,
  "ReportSendInterval": 60,
  "ServicePrincipal": {
    "TenantId": "<TENANT_ID>",
    "TennatId": "<TENANT_ID>",
    "AppId": "<APPLICATION_ID>",
    "SecretText": ""
  },
  "EventHubs": {
    "UploadReports": true,
    "ConnectionStrings": ["<SECRET-BEARING-CONNECTION-OBJECTS>"]
  }
}
```

以上只是**字段示意，不是可直接保存的完整配置**。真正的 JSON 应由 Notebook 生成。Eventstream Connection Strings 含访问密钥，**不要提交到 Git**。

## Phase 3 — Gateway VM 基础环境

### Step 1. 使用 PowerShell 7

在 VM 中安装/确认 PowerShell 7（POC 于 2026-11-13 实测为 7.6.10）。准备脚本时，保持模块目录与脚本同级：

```text
D:\Ang\FabricGatewayMonitoring\
├── configs\Config.json         # 本机私有配置
├── modules\Utils.psm1
├── modules\UploadGatewayLogs.psm1
├── Setup-UpdateConfiguration.ps1
├── Get-DataGatewayInfo.ps1
├── Run-GatewayHeartbeat.ps1
├── Run-UploadGatewayLogs.ps1
├── logs\
└── poc-input\Report\           # 测试用的日志副本目录
```

PowerShell 中检查：

```powershell
$PSVersionTable.PSVersion
Get-Module -ListAvailable -Name Az.Accounts,Az.Storage,DataGateway,MicrosoftPowerBIMgmt |
    Select-Object Name, Version, Path
```

**本次 POC 已安装全部四个模块**，因此初始化脚本询问是否安装时统一选择 `N`；不要在企业 VM 上盲目重复执行 `Install-Module -Scope AllUsers`。

### Step 2. 找到真正的 Gateway 日志路径

```powershell
Get-CimInstance Win32_Service |
    Where-Object Name -eq "PBIEgwService" |
    Select-Object Name, State, StartName
```

POC Gateway **实际安装目录为 `D:\On-premises data gateway`**，读取其下 `Microsoft.PowerBI.DataMovement.Pipeline.GatewayCore.dll.config` 的 `ReportFilePath`。其中 `%LOCALAPPDATA%` 对应 **Gateway Service Account** 的用户目录，不一定是默认的 `C:\Windows\ServiceProfiles\PBIEgwService`。

POC 中实测父目录：

```text
C:\Users\powrbi\AppData\Local\Microsoft\On-premises data gateway
└── Report\
    ├── QueryStartReport_*.log
    ├── QueryExecutionReport_*.log
    └── ...（共八种 Performance Report）
```

**无需开启 Additional logging** 才能得到 Performance Report。原 Setup 可使用 `Report` 的父目录；**本次 `Config.live-poc.json` 已实测直接指向 `...\Report` 也可行**。`D:\Ang\FabricGatewayMonitoring\logs` 是脚本自己的错误日志目录，不是输入源。

## Phase 4 — 初始化 + Secret 验证

### Step 1. 执行原始 Setup 脚本

```powershell
Set-Location "D:\Ang\FabricGatewayMonitoring"
.\Setup-UpdateConfiguration.ps1
```

交互操作：

1. 已安装的模块回答 `N`。
2. 选择修改 Gateway Log Path，填入真实服务账号的 **Gateway 日志父目录**。
3. 在 CSV 能读到 `GatewayObjectId` 时，脚本会自动识别 Gateway ID。
4. `Set the Service Principal Secret Text`：输入 Entra App 的 **Client Secret Value**（非 Secret ID）。

**实测踩坑：** PowerShell 7 Console 中，`Read-Host -MaskInput` 使用 `Ctrl+V` 只接收到 **1 个字符**；在 PowerShell 窗口**标题栏右键 → Edit → Paste** 才成功完整粘贴。不要为了这个问题改写整套初始化脚本。

### Step 2. 只验证加密结果，不输出密钥

```powershell
Import-Module ".\modules\Utils.psm1" -Force
$config = Get-Content ".\configs\Config.json" -Raw | ConvertFrom-Json
$stored = ConvertFrom-SecureWithMachineKey $config.ServicePrincipal.SecretText

[PSCustomObject]@{
    Decrypted   = [bool]$stored
    ReasonableLength = ($stored.Length -gt 10)
    TenantIdPresent  = [bool]$config.ServicePrincipal.TenantId
    GatewayIdPresent = [bool]$config.GatewayId
}
Remove-Variable stored
```

`SecretText` 是本机 DPAPI 密文，**与原始 Client Secret 不一样是正常的**。长度检查只能判断输入是否明显被截断；真正的权限/Secret 有效性仍要靠下一阶段 API 验证。

如果使用旧版 Notebook 生成的 JSON，可能只有拼错的 `TennatId`。旧配置需补齐 `TenantId`（**先备份**）：

```powershell
$config = Get-Content ".\configs\Config.json" -Raw | ConvertFrom-Json
$config.ServicePrincipal | Add-Member -NotePropertyName TenantId -NotePropertyValue $config.ServicePrincipal.TennatId -Force
$config | ConvertTo-Json -Depth 10 | Set-Content ".\configs\Config.json" -Encoding utf8
```

## Phase 5 — 验证 Metadata / Heartbeat

### Step 1. 运行 NodeInfo（一次性）

```powershell
.\Get-DataGatewayInfo.ps1
```

收到 HTTP `201 Created` 后，**还要查 KQL**；不能仅凭 HTTP 201 断定元数据完整：

```kusto
GatewayNodeInfo
| where isnotempty(nodeId)
| project nodeId, clusterName, machine, version, logDate
| order by logDate desc
```

若 `clusterName`、`machine`、`version` 为空，优先检查 **SPN 是否为目标 Gateway Cluster Admin**。本次补齐权限后，`/v1/gateways` 的可见 Cluster 由 0 变 1，Node 匹配由 0 变 1。

### Step 2. Heartbeat（持续执行）

已提交的 `Run-GatewayHeartbeat.ps1` 修复了官方版本 `$body` / `$msg` 变量不一致问题。

```powershell
Select-String ".\Run-GatewayHeartbeat.ps1" -Pattern "Add-MsgEventHub"
.\Run-GatewayHeartbeat.ps1
```

手动测试几轮后按 `Ctrl+C` 停止。KQL：

```kusto
GatewaysHeartbeat
| summarize LastSeen=max(ingestion_time()),
            EventsLast5Min=countif(ingestion_time()>ago(5m))
    by GatewayId
| extend AgeSeconds=datetime_diff("second", now(), LastSeen)
```

`AgeSeconds` 小于约 30–60 秒、Heartbeat 持续出现，说明链路正常。本次后台运行时曾观测到 17 秒延迟、最近 5 分钟 7 条（刚启动，并非稳态 SLA）。

## Phase 6 — **只在历史日志副本上**测试 Uploader

### Step 1. 复制待测文件

> **不要移动原始文件。** 选取 `QueryExecutionAggregationReport`、`SystemCounterAggregationReport` 两类，再补 `QueryStartReport` 与 `QueryExecutionReport`。

```powershell
$root = "D:\Ang\FabricGatewayMonitoring"
$source = "C:\Users\powrbi\AppData\Local\Microsoft\On-premises data gateway\Report"
$target = "$root\poc-input\Report"
New-Item -ItemType Directory -Path $target -Force | Out-Null

$patterns = @(
    "QueryExecutionAggregationReport_*.log",
    "SystemCounterAggregationReport_*.log",
    "QueryStartReport_*.log",
    "QueryExecutionReport_*.log"
)
foreach ($pattern in $patterns) {
    $file = Get-ChildItem $source -Filter $pattern -File |
        Sort-Object LastWriteTime -Descending | Select-Object -First 1
    if (-not $file) { throw "Missing report: $pattern" }
    Copy-Item $file.FullName -Destination $target
}
```

建立独立的**本机私有** `configs\Config.poc.json`：复制现有配置，仅将 `GatewayLogsPath` 指向 `D:\Ang\FabricGatewayMonitoring\poc-input`，并关闭不需要的 Lakehouse 上传。**不能提交这个文件到 Git。**

### Step 2. 调用原始 Uploader，测试后停止

```powershell
.\Run-UploadGatewayLogs.ps1 -configFilePath ".\configs\Config.poc.json"
```

观察 `201 Created`；处理完成后可出现 `Gateway Report log count: 0` 并持续轮询，**不是失败**。按 `Ctrl+C` 退出。

KQL 核对 Raw 与 Parsed：

```kusto
['GatewayReports-Raw']
| where logType in ("QueryStartReport","QueryExecutionReport",
                    "QueryExecutionAggregationReport","SystemCounterAggregationReport")
| summarize Messages=count(), DataRows=sum(array_length(log)) by logType
```

两张表可进一步检查：

```kusto
QueryStartReport
| summarize Records=count(),
            First=min(QueryExecutionStartTimeUTC),
            Last=max(QueryExecutionStartTimeUTC)
```

```kusto
QueryExecutionReport
| summarize Records=count(),
            First=min(QueryExecutionEndTimeUTC),
            Last=max(QueryExecutionEndTimeUTC)
```

本次四种类型的实际记录数见 [POC_STATUS.md](POC_STATUS.md)。这里证明**静态副本可以摄取并解析**，不代表连续采集对活动日志安全。

## Phase 6B — 2026-11-13 真实活动日志 POC（首轮已通过）

这是**单节点 POC** 实测，不允许直接复制到 PROD 作为风险验收：

1. 测试前将真实 `Report` 目录下 **8 类日志**复制至 `D:\Ang\GatewayLogs-Backup-BeforeMove`，另存 `before.csv` 记录文件状态；保留静态副本的原有 `Config.poc.json`。
2. 基于原 POC 配置另存 **私有** `configs/Config.live-poc.json`。关键实测参数为 `GatewayLogsPath = ["C:\Users\powrbi\AppData\Local\Microsoft\On-premises data gateway\Report"]`（**直接指向 Report**），`ReportSendInterval=60`、`ReportRetention=3650`（仅 POC 暂存）、`EventHubs.UploadReports=true`，两项 Lakehouse 上传设为 false。
3. 手动执行 `./Run-UploadGatewayLogs.ps1 -configFilePath "./configs/Config.live-poc.json"`。首轮查到 **8 个**文件，依次执行 **Move to Temp → CSV 分批上传 HTTP 201 → Merge 到 History → 删除 Temp**；日志最终在 `Report/History` 中按类型归档。手动停止持续循环以便查看文件。
4. Gateway 此后重新创建 8 个同名原始 Performance Logs 并继续写入；旧名称里的 `20261019T110120` 不等于新文件的物理创建日期。**HTTP 201 与重新创建文件已验证，不代表真实批次在 Eventhouse 没有重复/遗漏**。

**归档区分：** Gateway 自身轮转配置 `ReportFileCount=10`、`ReportFileSizeInBytes=104857600`（100 MiB）；Uploader 的 `Merge-ReportFiles` 另在同一 `History` 目录下按 Report Type（文件名第一个下划线前的部分）、当天创建、`<100mb` 追加；不满足条件时在 History 根目录创建新 `*_f.log`，不是创建按天子目录。Uploader 不再次上传 History 文件。

## Phase 7 — 让 Power BI Report 正常读数据

这是容易遗漏的一步，**仅修改 Parameters 不够，还要 Edit credentials**：

1. 在 KQL Database 的 Details 中复制 **Query URI**（不是 Ingestion URI）。
2. Semantic Model → Settings → Parameters：`KustoURL` 指向当前 Query URI；`KustoDB` 指向当前 `Gateway Monitoring` Database。
3. Semantic Model → **Data source credentials → Edit credentials**（或 Cloud Connection）以有权限的组织身份登录。
4. 打开 Report。先看 Gateways、System Counters，再看 Jobs、Queries。
5. 报表顶部筛选日期要匹配**日志内部的事件日期**，不一定是上传当天。此次切到 **2026-10-19** 才看到了当日 Jobs/Queries。

## Phase 8 — Task Scheduler 后台任务

在 Task Scheduler Library 下建 `FabricGatewayMonitoring` 文件夹，使用 **Create Task**；选择经批准的执行账号与 `Run whether user is logged on or not`。

**三个 Task 的 Action 共同部分：**

| 字段 | 填写值 |
| --- | --- |
| Program/script | `"C:\Program Files\PowerShell\7\pwsh.exe"`（包含双引号） |
| Start in | `D:\Ang\FabricGatewayMonitoring`（不带引号） |
| Add arguments | 按下表填写，只放参数和脚本路径 |

| Task | Add arguments | Trigger / 预期状态 |
| --- | --- | --- |
| **Gateway-Heartbeat** | `-NoLogo -NoProfile -NonInteractive -File "D:\Ang\FabricGatewayMonitoring\Run-GatewayHeartbeat.ps1"` | At startup；持续 `Running` |
| **Gateway-NodeInfo** | `-NoLogo -NoProfile -NonInteractive -File "D:\Ang\FabricGatewayMonitoring\Get-DataGatewayInfo.ps1"` | At startup + weekly；完成后 `Ready` |
| **Gateway-UploadLogs** | `-NoLogo -NoProfile -NonInteractive -File "D:\Ang\FabricGatewayMonitoring\Run-UploadGatewayLogs.ps1" -configFilePath ".\configs\Config.live-poc.json"` | At startup；2026-11-13 已启用 `Running`（POC 观察） |

**Tasks Settings：** 禁用默认的长任务 72 小时时限；`If the task is already running` 设为 `Do not start a new instance`，并设置失败重试、打开 History。

**实测踩坑：** 首次把可执行程序+参数都放进 Program/script，弹窗选择 `Yes` 后它变成 `C:\Program`，导致无法启动。应当将 `pwsh.exe` 单独放在 Program，参数放在 Add arguments；如果弹窗要求自动拆分，选择 `No` 并手动更正。

检查任务：

```powershell
$taskPath = "\FabricGatewayMonitoring\"
Get-ScheduledTask -TaskPath $taskPath | Select-Object TaskName, State
Get-ScheduledTask -TaskPath $taskPath -TaskName "Gateway-Heartbeat" |
    Get-ScheduledTaskInfo | Select-Object LastRunTime, LastTaskResult
```

2026-11-13 截图确认：`Gateway-Heartbeat` **Running**、`Gateway-NodeInfo` **Ready**（短任务正常完成）、`Gateway-UploadLogs` **Running**。`Running / 267009`（0x41301）表示任务仍在运行；**必须另外检查 KQL freshness/Raw/Parsed，而非只看 Task 状态**。

## Phase 9 — POC 连续观察与 PROD 首次采集计划

### Scheduled Task 切换至活动日志（2026-11-13 已完成）

- 实际 TaskPath：`\FabricGatewayMonitoring\`，Task Name：`Gateway-UploadLogs`。先导出 `Gateway-UploadLogs.backup.xml`（1528 bytes）；`Export-ScheduledTask` 必须带 `-TaskPath` 才能找到目标。
- PowerShell `Set-ScheduledTask` 修改 Action 时提示 `The user name or password is incorrect`，因此通过 **Task Scheduler GUI → Task Properties → Actions → Edit** 修改现有 Action 的 `-configFilePath ".\configs\Config.live-poc.json"`，保留 `pwsh.exe`、原有 WorkingDirectory、Triggers、Principal；无需删除重建任务。
- 已启用三个 Task：Heartbeat Running、NodeInfo Ready、UploadLogs Running。Uploader 自带 `ReportSendInterval=60` 无限循环，不应每分钟让 Task Scheduler 再启动新进程。
- **后续 24–48 小时、继而 3–7 天：** 检查 Gateway 的 Refresh/写日志，History 按类型追加、Eventhouse 最新记录与 Raw/Parsed 行数、CPU/内存/磁盘；还需专项验证部分批次成功后失败、Temp 文件恢复、重复/遗漏与重试。

### PROD 首次导入策略（计划中，未实施）

- PROD 已观察到大量历史 Performance Logs、多个近 100 MiB；**不应直接启动原始 uploader 全量回传历史**。
- 先只读统计并复制候选文件到 **Report 目录之外的独立 Staging**；从约定 cutoff（计划从昨天 00:00）筛选记录。Gateway 原始文件名中的日期可能长期不变，按文件名日期过滤并不可靠；原始 uploader **没有按 CSV 事件时间过滤功能**，需要独立预处理或策略验证。
- 规划首轮 Backfill 与随后在线持续采集的交接，避免重复/遗漏；勿将旧文件移动到 `Report\Archive` 并期待脚本跳过，原脚本递归扫描子目录。
- 生产部署前还需验证中断重试、权限、告警、成本和多节点；只读 Tail/Checkpoint 为备用方案，不代表当前已经部署。

**当前仍非 Production Ready。** 本次 POC 只批准测试节点的原始 Move Uploader 持续观察。
