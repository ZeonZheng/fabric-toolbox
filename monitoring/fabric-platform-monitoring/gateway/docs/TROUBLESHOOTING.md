# Gateway Monitoring｜排障手册（已遇到的问题）

> 先按症状定位，再展开处理步骤。**“已验证修复”**指本次 POC 的实际结果；**“提交版待重测”**指后来整理到 Git 的改动，不能混淆。

[返回总览](../POC_README.md) · [部署步骤](DEPLOYMENT_RUNBOOK.md) · [修改与官方差异](UPSTREAM_CHANGES.md)

## 快速定位

| 症状 / 关键词 | 优先检查 | POC 处理结果 |
| --- | --- | --- |
| `NoAvailableProvider` | `sempy.fabric` 导入时无法找到 notebook context | 避开导入；fork 最终改为 notebook runtime context，**待重测** |
| `JSONDecodeError` / Fabric CLI | `fab` 的非 JSON 错误被传给 `json.loads` | Setup Notebook 改走 Fabric REST |
| `Install-Package: No match` | 模块其实已经存在、PowerShell Gallery 不可用 | 已安装则回答 `N` |
| 找不到 `*Report_*.log` | 指错了 Gateway 日志目录 | 使用服务账户的 `LOCALAPPDATA`，不是脚本 `logs` |
| Client Secret 保存后长度只有 1 | PowerShell `Read-Host -MaskInput` 的 `Ctrl+V` 粘贴问题 | 通过窗口菜单 **Edit → Paste** 完整粘贴 |
| `TennatId` / `TenantId` | Notebook 生成字段与 PowerShell 读取字段不同 | POC 配置补上 `TenantId` |
| NodeInfo HTTP 201 但 Cluster 为空 | SPN 没有目标 Gateway Cluster Admin | 补上权限后数据完整 |
| Heartbeat 没有上报 | 原始脚本将 `$body` 当作消息发出 | 改为 `$msg`，实测成功 |
| Task Scheduler 把 `pwsh.exe` 变成 `C:\Program` | “Program/script” 与参数混填 | 将三栏分开，重新保存后运行成功 |
| Uploader 运行很久 / `log count: 0` | 分批上报完成后仍持续轮询 | 29 批 HTTP 201，KQL 验证 54,844 条（首次） |
| Report 仅有 Gateways / Counters | 未上传查询明细、日期不匹配或 Credentials 未配置 | 补两类日志、改日期、Edit credentials |
| 原始 Gateway 报表文件被长期追加 | 文件名旧时间戳 ≠ 物理创建/事件日期 | 2026-11-13 单节点 POC Move 后同名文件重新建立并继续写入；PROD 不可直接推断安全 |
| `Export-ScheduledTask` 找不到 Task | 任务位于子目录 `\FabricGatewayMonitoring\` | 读取 `TaskPath` 并传入 `-TaskPath` 后导出成功 |
| `Set-ScheduledTask`: `The user name or password is incorrect` | 修改时触发任务 Principal 凭据验证 | 保持原 Task，改用 Task Scheduler GUI 编辑 Action 并保存，已成功 |

## 1. Notebook：`NoAvailableProvider` / `JSONDecodeError`

### 问题 A：`import sempy.fabric as fabric` 直接异常

```text
NoAvailableProvider: lack of valid ContextProvider
```

**原因定位：** 当前 Fabric Notebook 的运行上下文不满足 `sempy` 环境检测，而导入过程便触发 `get_notebook_workspace_id()`。

**处理：** 本 fork 的 Setup Notebook 已移除该导入，改为从 Notebook runtime context 取 Workspace ID：

```python
workspace_id = notebookutils.runtime.context["currentWorkspaceId"]
```

**状态：** 这是对上传 Notebook 的追加兼容修正，**Git 中版本尚未重新运行验证**。如环境不提供该键，需按实际 Notebook Runtime API 调整，不要直接宣称已修好全部环境。

### 问题 B：Fabric CLI 运行后 `json.loads` 失败

```text
JSONDecodeError: Expecting value: line 1 column 1 (char 0)
```

原版先执行 `fab api .../topology`，部分调用使用 `silently_continue=True`，再对 CLI 输出直接做 `json.loads`。CLI 失败或返回非 JSON 文本时就出现此错误。

上传的适配 Notebook 使用 **Fabric REST API**：检查 HTTP 响应码、按实际 Eventstream Item 找 ID、验证 CustomEndpoint Source 数量后读取 Connection。无需再依赖 CLI `text` 包装格式。**执行时仍需对应的 Fabric 权限与 Notebook 环境。**

## 2. Gateway VM：日志和依赖

### 模块已安装却要求重新安装

先检查：

```powershell
Get-Module -ListAvailable -Name Az.Accounts,Az.Storage,DataGateway,MicrosoftPowerBIMgmt |
    Select-Object Name, Version, Path
```

本次模块都在，因此 `Setup-UpdateConfiguration.ps1` 的安装提示统一回答 `N`。企业 VM 无法访问 PowerShell Gallery 时，不要盲目执行 `Install-Module`。

### 找不到 Gateway Performance Report

曾错误配置成脚本目录 `D:\Ang\FabricGatewayMonitoring\logs`，导致提示手动输入 Gateway ID。

正确操作：

```powershell
Get-CimInstance Win32_Service |
    Where-Object Name -eq "PBIEgwService" |
    Select-Object Name, State, StartName
```

检查安装目录 `GatewayCore.dll.config` 的 `ReportFilePath`，并去服务账户的本地 Profile 下找 `Report\*Report_*.log`。**脚本要求的是 `Report` 的父目录**。POC 有 8 类性能报告；无需开启 Additional logging。

## 3. Secret：为什么只有一个星号？

本次实测：将虚拟字符串 `Testing123456789` 用 `Ctrl+V` 粘贴到 `Read-Host -MaskInput`，结果：

```text
ExpectedLength = 16
ActualLength   = 1
Matches        = False
```

**有效解决：** PowerShell 窗口标题栏菜单 → **Edit → Paste**，此时出现完整掩码长度。原始 Setup 脚本**不需要修改**。

`Config.json` 中的 `ServicePrincipal.SecretText` 是 **DPAPI（LocalMachine）加密密文**，不是明文；解密长度与再次输入比较一致即可，不要打印明文。**注意误把 Secret 粘贴在 `PS>` 普通命令行上会形成潜在命令历史暴露风险。**

## 4. Metadata：HTTP 201，但字段是 `null`

**现象：** `GatewayReports-Raw` 有 `GatewayNodeInfo` 消息，但 `clusterName`、`machine`、`version` 为空。

**实际原因：** 监控 SPN 没有被授予**目标 Gateway Cluster Admin**；与 Fabric Workspace 权限不是同一层。

**实测修复：** 授予 Gateway Cluster Admin 后，`GET /v1/gateways` 可见 Cluster 数量从 **0 → 1**、匹配 Node 从 **0 → 1**；再次运行 NodeInfo，KQL 中关键字段正常。

```kusto
GatewayNodeInfo
| where isnotempty(nodeId)
| project nodeId, clusterName, machine, version, logDate
| order by logDate desc
```

原版 Setup Notebook 写 `ServicePrincipal.TennatId`，但 `Get-DataGatewayInfo.ps1` 要读取 `TenantId`。已在 fork 的 Setup Notebook **同时输出两个键**（待重测），旧配置需手动补上 `TenantId`。

## 5. Heartbeat：变量名错误

官方脚本先生成：

```powershell
$msg = $processMsg | ConvertTo-Json
```

却发送：

```powershell
Add-MsgEventHub -msg $body -connectionType "Heartbeat" -config $config
```

**已验证修复：**

```powershell
Add-MsgEventHub -msg $msg -connectionType "Heartbeat" -config $config
```

fork 已提交这个单行改动。手动 HTTP 201、KQL `Responding=true` 和 Task Scheduler 后台采集都验证成功。

## 6. Task Scheduler：路径分割错误

**错误现象：** 点击任务 `Run` 没有运行，Action 打开后变成 `Program/script = C:\Program`。

**原因：** 在 Program/script 中混入了参数，Windows 弹窗建议拆分，选择 `Yes` 后错误截断。

**正确配置：**

| 字段 | 内容 |
| --- | --- |
| Program/script | `"C:\Program Files\PowerShell\7\pwsh.exe"` |
| Add arguments | `-NoLogo -NoProfile -NonInteractive -File "D:\Ang\FabricGatewayMonitoring\Run-GatewayHeartbeat.ps1"` |
| Start in | `D:\Ang\FabricGatewayMonitoring`（不加引号） |

其他 NodeInfo / UploadLogs 按 [Runbook](DEPLOYMENT_RUNBOOK.md) 的对应脚本调整。若提示自动拆分，返回检查三个字段，不选择错误的自动拆分。

```powershell
Get-ScheduledTask -TaskPath "\FabricGatewayMonitoring\" |
    Select-Object TaskName, State
```

`Running / 267009` 对无限循环 Heartbeat **正常**；`Ready / 0` 对执行完毕的 NodeInfo **正常**。任务状态以外，还应在 KQL 验证最近的 Heartbeat 持续更新。

## 7. Uploader：HTTP 201、空文件 / Jobs 无数据

### 上传很久、重复看到 `201 Created`

Uploader 会拆分 CSV 成多批请求；处理完静态文件后持续轮询并打印 `Gateway Report log count: 0`。本次首批聚合日志 **29 批、54,844 条**，Raw 与 Parsed 数量一致。**不能仅凭进程还在运行就认为上传失败。**

### Power BI Jobs / Queries 显示空

先确认已上传：

- `QueryStartReport` 和 `QueryExecutionReport`（本次各 2,055 条）
- Semantic Model 的 `KustoURL` / `KustoDB` 已正确配置
- **Data source credentials → Edit credentials** 已完成
- 日期过滤与**查询实际发生日期**一致（本次选择 2026-10-19 后显示 3 Jobs / 5 Queries）

### 不要把“文件移动”当作增量采集

原 Uploader 使用 `Move-Item`，处理所有符合 `*Report_*.log` 的文件。**2026-11-13 在 POC 单节点成功移动 8 类真实文件、收到 HTTP 201、归档至 History，Gateway 后续同名文件重新创建并继续写入；Uploader Task 已启用 Running（观察阶段）。** 不代表 PROD 安全或 Exactly-once。详见 [运维与架构说明](ARCHITECTURE_OPERATIONS.md)。

### 源文件名仍有旧日期、History 文件却是今天

这是两套机制：Gateway 新建的物理文件可能沿用原文件名 `20261019T110120`；Uploader 把原文件 Move 至 Temp 时追加处理时间 `_HHmmss_x.log`，随后 `Merge-ReportFiles` 按当时 UTC 时间创建/追加本地 History `*_f.log`。History 的文件名**不保留原始会话时间戳**；Log 事件时间应看 CSV 内对应字段。

### 上传成功之后还可能有什么风险？

- 发送端逐批发送 HTTP 201 后才执行本地 CSV Merge。若前几个批次已成功、后续失败，原文件已在 Temp，重传可能导致重复。
- 原始扫描带 `-Recurse`，会包含 Temp 下的 `*_x.log`；失败恢复没有经过 POC 验证。
- 原始 `GatewayLogsPath` 可以指向 `Report` 父目录；本次 POC 的 `Config.live-poc.json` **直接指向 Report**，首轮上传成功。临时 `ReportRetention=3650` 只影响本地 History，不代表 Eventhouse 保留期。
- 连续任务 Running 不等于数据新鲜；应以 Eventhouse Raw/Parsed、Heartbeat 时间戳和 Gateway 业务刷新为准。

### Task Scheduler 文件路径与修改方式

原任务在 `Task Scheduler Library\FabricGatewayMonitoring\Gateway-UploadLogs`，`TaskPath` 为 `\FabricGatewayMonitoring\`。使用 `Export-ScheduledTask -TaskName "Gateway-UploadLogs" -TaskPath "\FabricGatewayMonitoring\"` 才能正确定位并生成备份；仅指定 TaskName 曾报 `The system cannot find the file specified`。后台 Task 修改时 `Set-ScheduledTask` 报 `The user name or password is incorrect`，最终通过 GUI 中的 **Properties → Actions → Edit** 将参数切换到 `Config.live-poc.json` 并成功启动，不需要删除重建任务。
