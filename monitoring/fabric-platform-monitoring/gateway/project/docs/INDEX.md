# 技术文档索引

> 该目录保存**我们的实施说明、研究与排障**，不是微软上游文档。当前阶段和下一步以 [STATUS.md](../STATUS.md) 为准；日期和问题状态统一在 [tracking/](../tracking/ISSUES.md)。

| 需要查找 | 文件 | 用途 |
| --- | --- | --- |
| 架构与 Collector | [ARCHITECTURE_OPERATIONS.md](ARCHITECTURE_OPERATIONS.md) | Move/Temp/History、Uploader 局限、多节点及 PROD 决策 |
| 部署与接续操作 | [DEPLOYMENT_RUNBOOK.md](DEPLOYMENT_RUNBOOK.md) | 环境、安装及运行命令 |
| 发生错误 | [TROUBLESHOOTING.md](TROUBLESHOOTING.md) | 实际遇到的症状、判断及已验证修复 |
| 实际 POC 历史 | [POC_STATUS.md](POC_STATUS.md) | 2026-11-12～11-17 详细测试记录（**不是实时状态页**） |
| Upstream vs Fork | [UPSTREAM_CHANGES.md](UPSTREAM_CHANGES.md) | 哪些是微软源文件、哪些是我们的修改 |
| 解析问题复现 | [PARSER_SCHEMA_AUDIT.md](PARSER_SCHEMA_AUDIT.md) | 已观察到的 Raw / Parsed 值差异、原因与验证证据 |
| 全字段对应关系 | [SCHEMA_FIELD_MAPPING_AUDIT.md](SCHEMA_FIELD_MAPPING_AUDIT.md) | 8 类 CSV / GatewayNodeInfo 的字段覆盖及下游映射缺口 |
| PROD 文件追踪设计 | [PROD_UPLOAD_MANIFEST_DESIGN.md](PROD_UPLOAD_MANIFEST_DESIGN.md) | Upload Manifest 设计草案（未实现） |
| 可编辑架构图 | [Gateway-Monitoring-POC.drawio](Gateway-Monitoring-POC.drawio) | 历史 POC 快照，需对照最新状态 |

运行证据：[evidence/](../evidence/README.md) · KQL：[kql/](../kql/poc-parser-fix-20261117/README.md) · 适配 Notebook：[notebooks/](../notebooks/)。
