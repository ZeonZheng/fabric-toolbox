# Gateway Monitoring｜与官方实现的差异记录

> **对照基线：** [Microsoft Fabric Toolbox（Gateway PowerShell）](https://github.com/microsoft/fabric-toolbox/tree/main/monitoring/fabric-platform-monitoring/gateway) 和 [Jumpstart Gateway Setup Notebook v0.2.3](https://github.com/ecotte/fabric-platform-monitoring/blob/v0.2.3/gateway-monitoring/src/fabric-platform-monitoring/setup/Gateway%20Monitoring%20Setup.Notebook/notebook-content.py)。  
> **范围：** 仅当前 fork 的 `monitoring/fabric-platform-monitoring/gateway/`。原有项目 `README.md`、Fabric Report/Semantic Model/KQL、Capacity、Inventory 等均未改动。

[返回总览](../POC_README.md) · [POC 状态](POC_STATUS.md) · [复现步骤](DEPLOYMENT_RUNBOOK.md)

## 先看变更矩阵

| 文件 / 配置 | 官方原始行为 | 本次记录或修复 | 验证状态 |
| --- | --- | --- | --- |
| **Gateway Monitoring Setup.ipynb** | `fab` CLI 查询 Item、Topology、Connection | 上传的本地版本改用 Fabric REST、显式 HTTP/JSON 检查、校验 CustomEndpoint；fork 再做兼容性修复 | 🟡 旧适配路线用于 POC；**最终提交版待重测** |
| **Jumpstart Installer.ipynb** | 通过 `fabric_jumpstart.install(...)` 安装 | 保留本地安装/更新示例，清理历史执行输出，`update_existing=True` 默认注释 | 🟡 仅作为施工记录，不能视为安装成功证明 |
| **Run-GatewayHeartbeat.ps1** | 生成 `$msg`，但发送 `$body` | **只改一行：** `$body` → `$msg` | ✅ HTTP 201 / KQL 验证 |
| **Setup-UpdateConfiguration.ps1** | 交互设置路径、Gateway ID、加密 Secret | **源文件未改动**；只使用正确粘贴方式和运行时配置 | ✅ |
| **Get-DataGatewayInfo.ps1** | 读取 `ServicePrincipal.TenantId`；通过 API 获取 NodeInfo | **源文件未改动**；补 SPN 的 Gateway Cluster Admin | ✅ |
| **Run-UploadGatewayLogs.ps1 / modules/*.psm1** | 匹配 `*Report_*.log`，`Move-Item` 到 Temp，上传后归档 | **Git 源文件未改动**；原 `Config.poc.json` 处理静态副本；新私有 `Config.live-poc.json` 处理真实 POC Report | ✅ 静态副本；✅ 活动源文件首轮（8 类 / HTTP 201）；🟡 连续运行观察 |
| **TaskSchedulers/*.xml** | 仓库附带 Scheduled Task 模板 | **Git XML 未改**；在 VM 修改任务 Action 至 `Config.live-poc.json`，保留原 Task；备份任务 XML 仅存 VM 本地 | ✅ 2026-11-13 Heartbeat Running、NodeInfo Ready、UploadLogs Running |

## 1. Setup Notebook 的核心差异

### A. 原版 `fab` CLI → 直接 Fabric REST

**原版关键路径：**

```python
result = subprocess.run(["fab", "-c", command], capture_output=True, text=True)
heartbeat_conn_id = json.loads(
    run_fab_command(f"api workspaces/{workspace_id}/eventstreams/{heartbeat_id}/topology",
                    capture_output=True, silently_continue=True)
).get("text", {}).get("sources", [])[0].get("id")
```

问题：`fab` 调用失败后，隐藏错误信息或拿到非 JSON 字符串，紧接着 `json.loads` 就会抛 `JSONDecodeError`。此外，直接取 `sources[0]` 无法保证取到的是需要的 CustomEndpoint。

**上传的本地改版：** 使用 Notebook 的 Fabric 访问令牌，调用 Fabric REST 的 workspace items、Eventstream topology 与 source connection；遇到非成功 HTTP 状态直接报错；按实际名称和 `CustomEndpoint` 类型选择需要的 Source，避免默默选错。

### B. fork 再做的“补充修正”（未重测）

- 移除残留的 `import sempy.fabric as fabric`，避免触发曾遇到的 `NoAvailableProvider`；改用：

```python
workspace_id = notebookutils.runtime.context["currentWorkspaceId"]
```

- 原始 Notebook 配置曾使用 `TenatId` 变量 / `TennatId` JSON 字段，但 PowerShell NodeInfo 读取 `TenantId`。fork **保留旧字段并额外增加正确的字段**：

```python
"ServicePrincipal": {
    "TennatId": TenatId,  # 兼容原始生成结果
    "TenantId": TenatId,  # 下游 PowerShell 使用
    "AppId": AppId,
    "SecretText": ""
}
```

- Notebook 示例中的轮询默认值：Heartbeat `1 → 30` 秒；Report polling `5 → 60` 秒。
- 清除全部执行输出、写入占位值；不在 Git 存储目标 Workspace ID、Connection Strings 或 Secret。

**注意：** 最终提交的这两份 Notebook 没有再在真实 Fabric Workspace 中重跑；需要下一次在新目标环境验证，而不能把代码审阅视为部署成功。

## 2. Heartbeat 的唯一已验证源码修复

**官方代码：**

```powershell
$msg = $processMsg | ConvertTo-Json
Add-MsgEventHub -msg $body -connectionType "Heartbeat" -config $config
```

**本地修复：**

```powershell
$msg = $processMsg | ConvertTo-Json
Add-MsgEventHub -msg $msg -connectionType "Heartbeat" -config $config
```

这个修改在实际 Gateway VM 上已经通过 HTTP 201、Eventhouse `GatewaysHeartbeat` 和 Scheduled Task 运行验证。

## 3. 仅发生在部署环境中的修改（不进 Git）

| 修改 | 理由 |
| --- | --- |
| `Config.json` 的 `GatewayLogsPath` | 实际服务账号目录，而不是默认 PBIEgwService |
| `Config.json` 的 `TenantId` | 与旧 Notebook 的 `TennatId` 拼写兼容 |
| `SecretText` 加密值 | 本机 DPAPI 生成，不能随代码版本管理 |
| SPN 加入目标 Gateway Cluster Admin | 否则可以发送 Metadata，但关键字段为空 |
| `KustoURL`、`KustoDB` 和 **Edit credentials** | 预制 Semantic Model 需要连接实际 KQL Database |
| Task Scheduler Action、执行账号、频率 | 2026-11-13 通过 GUI 将 `Gateway-UploadLogs` 配置从 `Config.poc.json` 切换到 `Config.live-poc.json` 并运行；PowerShell `Set-ScheduledTask` 因现有凭据验证失败，未改动 Git 中 TaskScheduler XML |
| `Config.poc.json`、`Config.live-poc.json`、`poc-input`、`Temp`、`History` | 静态副本与真实 POC 采集的**本机私有**运行时配置/数据，**不进入 Git** |

## 4. 与原始仓库的边界

- 没有把未完成的 Tail Collector 当作已经实施的代码；目前在 POC 上实际运行的是**原始 Move Uploader**，尚未证明长期稳定性。
- 没有把经讨论的 Kafka、Fluent Bit、Vector、C# Worker 等方案描述为已经选型或部署。
- 没有改动原始 `README.md`、Capacity Events / Activity Events / Inventory 或其他模块。
- 未来 Jumpstart `update_existing=True` 可能覆盖 Fabric 托管 Items；应先评估差异和备份，再决定是否执行。

**2026-11-13 新发现（仅部署时观察与记录，未改源码）：** 原 Uploader 的 History 合并发生在**上传之后**，按文件名前缀、当天创建、低于 100 MiB 选择本地归档文件；与 Gateway 自身的 100 MiB / 10 个源日志轮转独立。用户计划 POC 连续观察后，以独立 Staging 和事件时间 cutoff 测试 PROD 首次 Backfill，不直接上传所有生产历史日志。详见 [架构运维](ARCHITECTURE_OPERATIONS.md) 与 [状态记录](POC_STATUS.md)。

**重新实施时先读：** [Runbook](DEPLOYMENT_RUNBOOK.md)，遇到错误再看 [Troubleshooting](TROUBLESHOOTING.md)；若要把 fork 变更回合并或向上游贡献，使用本页核对真实 diff。
