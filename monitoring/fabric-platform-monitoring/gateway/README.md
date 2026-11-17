# Gateway Monitoring｜目录说明

这里是 Microsoft Fabric Platform Monitoring 的 Gateway 子项目。**请明确区分上游项目资产与我们的工作记录。**

| 目录 | 来源 | 内容 |
| --- | --- | --- |
| **[project/](project/README.md)** | **我们的 POC / PROD 规划** | [当前状态](project/STATUS.md)、[问题追踪](project/tracking/ISSUES.md)、技术文档、实测证据、KQL 测试和适配 Notebook |
| [PowerShellScript/](PowerShellScript/) | 微软原始脚本 + 1 项已验证的 Heartbeat 修复 | Uploader、Heartbeat、Gateway NodeInfo |
| [TaskSchedulers/](TaskSchedulers/) | 微软上游 | Windows Scheduled Task 模板，不代表 VM 当前运行配置 |
| [PBI Report/](PBI%20Report/) | 微软上游 | Gateway Monitoring Report 模板 |
| [../src/Gateway/](../src/Gateway/) | 微软上游 | Eventhouse / Eventstream / Semantic Model 等定义 |

唯一确认的 PowerShell Fork 变更：Heartbeat 发送变量 `$body` 修复为 `$msg`。详情：[上游差异](project/docs/UPSTREAM_CHANGES.md)。

**人和 AI 都从这里开始：** [项目总览](project/README.md) → [当前状态](project/STATUS.md) → [问题清单](project/tracking/ISSUES.md)。

工作分支：`docs/gateway-monitoring-poc-20261112`。**不能因为本地 POC 成功就称为 Production Ready。**

最后更新：2026-11-17（Asia/Singapore）。
