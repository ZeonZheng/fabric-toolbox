# Gateway Monitoring｜POC 实施记录

> **更新时间：2026-11-12｜范围：单个 On-premises Data Gateway Cluster / Node**  
> 本目录记录**实际实施、排障和验证结果**，方便下次继续施工。不是对整个 Fabric Platform Monitoring 项目的完成声明。

## 当前结论（先看这里）

| 组件 | 状态 | 依据 |
| --- | --- | --- |
| Fabric Eventstreams → Eventhouse → Semantic Model → Power BI | ✅ 已验证 | Gateway、CPU/Memory、Jobs、Queries 均有可见数据 |
| Gateway NodeInfo | ✅ 已验证 | SPN 获得 Gateway Cluster Admin 后，KQL 元数据完整 |
| Gateway Heartbeat | ✅ 已验证 | Scheduled Task 后台持续运行，KQL 数据更新 |
| Performance Logs 上传 | 🟡 部分验证 | **只验证了复制的历史日志**，未对正在写入的真实日志持续采集 |
| Gateway-UploadLogs Scheduled Task | ⏸️ Disabled | 已验证任务可运行，出于源文件风险未长期启用 |
| Running Jobs / Job Details、多个节点、Capacity Events 等 | ◻️ 未验证 | 不在这次 Gateway POC 的已验收范围 |

**下一步：** 决定并验证真实性能日志的**只读增量采集方案**（文件轮转、CSV 跨行、Checkpoint、重试、去重）。**不要直接把现有 Uploader 指向生产日志目录后启动。**

## 文档导航

| 想做什么？ | 直接阅读 |
| --- | --- |
| **下次从头部署或接着操作** | [DEPLOYMENT_RUNBOOK.md](docs/DEPLOYMENT_RUNBOOK.md) |
| **遇到报错，快速定位** | [TROUBLESHOOTING.md](docs/TROUBLESHOOTING.md) |
| **确认实际完成了什么** | [POC_STATUS.md](docs/POC_STATUS.md) |
| **看我们相对官方版本改了什么** | [UPSTREAM_CHANGES.md](docs/UPSTREAM_CHANGES.md) |
| **理解 Uploader 移动日志的影响与生产方案** | [ARCHITECTURE_OPERATIONS.md](docs/ARCHITECTURE_OPERATIONS.md) |
| **查看可编辑的架构图** | [Gateway-Monitoring-POC.drawio](docs/Gateway-Monitoring-POC.drawio) |
| **看两个改造过的 Notebook** | [poc-notebooks/](poc-notebooks/) |

## 已验证的数据链路

```text
Gateway VM
 ├─ Get-DataGatewayInfo.ps1 ─────┐
 ├─ Run-GatewayHeartbeat.ps1 ────┤ → Fabric Eventstreams
 └─ Run-UploadGatewayLogs.ps1 ───┘   （仅历史日志副本测试）
                                      ↓
                             Eventhouse / KQL Database
                                      ↓
                             Gateway Monitoring Semantic Model
                                      ↓
                             Power BI Gateway Monitoring Report
```

- [可编辑 POC 架构图（Draw.io）](docs/Gateway-Monitoring-POC.drawio) — 区分已验证与未实现部分。
- [原仓库的数据逻辑图](../Images/05%20-%20Gateway%20Report%20Data%20Logic.png)；[Fabric Jumpstart 官方部署流程](https://jumpstart.fabric.microsoft.com/catalog/fpm-gateway-monitoring/)。
- 原项目的 [README.md](../README.md) 保持不变。**Capacity Events、Activity Events、Inventory 等模块未在本次 POC 测试。**

## 我们实际修改了什么？

1. **Gateway Setup Notebook**：保留上传版本的 Fabric REST 实现，移除有问题的 CLI 调用路径；在 fork 中又补充 `TenantId` 兼容、环境 Context 和 30/60 秒默认配置。**这几项最后修改尚待重测。**
2. **Jumpstart Installer Notebook**：保留安装及可选更新示例；移除历史执行输出，默认注释掉存在覆盖风险的 `update_existing=True`。
3. **Heartbeat PowerShell**：修复 `Add-MsgEventHub -msg $body` → `-msg $msg`，这个修复已经实测有效。
4. **原始 Setup / NodeInfo / Uploader 脚本**：未在 Git 中修改。实际部署时的配置、凭据和 Scheduled Tasks 是运行时操作，详见[变更清单](docs/UPSTREAM_CHANGES.md)。

## 安全与使用注意

- **仓库里没有** `Config.json`、Event Hub SAS Key、Client Secret、执行结果中的敏感信息和原始 Gateway 日志。
- `SecretText` 在 Windows 本机使用 DPAPI 加密，**不能将某台 VM 的密文直接复制到其他 VM 使用**。
- Uploader 的 `Move-Item` 会移动被采集文件；已观察到 Gateway 会长期向同名性能日志追加，因此**当前未批准对活动文件直接移动**。
- 这是社区 Solution Accelerator，**不等于生产级日志收集系统已通过可靠性测试**。

**本分支仅记录并改动 `monitoring/fabric-platform-monitoring/gateway/`。** 可先阅读部署指南，再根据状态表继续施工。
