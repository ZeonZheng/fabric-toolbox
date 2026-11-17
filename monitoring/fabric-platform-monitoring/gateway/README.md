# Gateway Monitoring — navigation

**Upstream vs our work**

| Directory | Ownership | Purpose |
| --- | --- | --- |
| [project/](project/README.md) | **OURS** | POC engineering, status, issue register, evidence, candidate patches |
| [PowerShellScript/](PowerShellScript/) | Microsoft upstream, **one heartbeat bug fix** | PowerShell uploader and scripts |
| [TaskSchedulers/](TaskSchedulers/) | Microsoft upstream | Task XML templates, not actual runtime configuration |
| [PBI Report/](PBI%20Report/) | Microsoft upstream | Original Power BI template |
| [../src/Gateway/](../src/Gateway/) | Microsoft upstream | Eventhouse/Eventstream/semantic definitions |

Source-level delta is documented in [project/docs/UPSTREAM_CHANGES.md](project/docs/UPSTREAM_CHANGES.md). All new project artifacts live under [project/](project/README.md), except this index, the legacy POC_README redirect, and the necessary .gitignore. Do not change `main` or deploy a candidate to Eventhouse without explicit approval and tests.

**Human and AI starting point:** [project/README.md](project/README.md) → [project/STATUS.md](project/STATUS.md) → [project/ISSUES.md](project/ISSUES.md).

Updated 2026-11-17 (Asia/Singapore); branch `docs/gateway-monitoring-poc-20261112`.
