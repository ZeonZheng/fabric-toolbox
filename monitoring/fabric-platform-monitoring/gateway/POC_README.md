# Gateway Monitoring — Local POC implementation

> **Scope:** On-premises data gateway monitoring only. This is a local implementation record for the fork, **not** a claim that Capacity Events, Activity Events, Inventory, or other Fabric Platform Monitoring modules were deployed or tested.
>
> **Status (12 Nov 2026):** Single-gateway functional POC **validated** through Eventstreams, Eventhouse, semantic model and Power BI report. Scheduled Heartbeat and NodeInfo **validated**. Unattended production report-log collection **not deployed**; the Uploader task intentionally remains **disabled**.

## Start here

| Document | Purpose |
| --- | --- |
| [Deployment runbook](docs/DEPLOYMENT_RUNBOOK.md) | Repeatable install, configuration, verification and Task Scheduler steps |
| [POC evidence and scope](docs/POC_STATUS.md) | Observed results, exact test scope, and what remains unvalidated |
| [Changes from upstream](docs/UPSTREAM_CHANGES.md) | Actual code differences versus Jumpstart v0.2.3 and official fork scripts |
| [Troubleshooting](docs/TROUBLESHOOTING.md) | Failures encountered and the fixes validated during this POC |
| [Architecture and operational decisions](docs/ARCHITECTURE_OPERATIONS.md) | Data flow, source-file lifecycle, security, and production design alternatives |
| [Editable architecture](docs/Gateway-Monitoring-POC.drawio) | Locally authored editable Draw.io diagram |
| [Local POC notebooks](poc-notebooks/) | Sanitized uploaded notebooks; no output cells, secrets, or tenant-specific IDs |

The repository's existing [main Fabric Platform Monitoring README](../README.md), original folder structure, other modules, and packaged Fabric artifacts have **not** been changed.

## Architecture

![Official Fabric Jumpstart architecture (external reference)](https://jumpstart.fabric.microsoft.com/images/diagrams/fpm-gateway-monitoring_light.svg)

*Official illustration from [Fabric Jumpstart — Gateway Monitoring](https://jumpstart.fabric.microsoft.com/catalog/fpm-gateway-monitoring/), linked rather than copied. The local [editable Draw.io diagram](docs/Gateway-Monitoring-POC.drawio) distinguishes verified POC from the unimplemented log-tail design.*

The existing repository also contains [Gateway Report Data Logic](../Images/05%20-%20Gateway%20Report%20Data%20Logic.png).

## What is committed

- The user's two uploaded Fabric notebooks, **sanitized** and documented. The Setup Notebook uses Fabric REST instead of Fabric CLI and includes minimal follow-up portability fixes. See [diff ledger](docs/UPSTREAM_CHANGES.md).
- One upstream PowerShell change in \`PowerShellScript/Run-GatewayHeartbeat.ps1\`: pass \`$msg\`, not undefined \`$body\`, to \`Add-MsgEventHub\`. This was manually validated locally.
- **No** generated \`Config.json\`, SAS / Event Hub connection strings, service-principal client secrets, gateway logs, screenshots containing sensitive data, task credentials, or other runtime state.

## Guardrails

- The original Uploader **moves and archives** matching \`*Report_*.log\` files. A test against **copies** succeeded. It is **not** validated as safe to run continuously against active Gateway report files.
- Shared repository content is not a deployment of a production service, and cloned notebooks must be configured with fresh environment IDs and re-executed.
- The PowerShell service-principal secret uses local-machine DPAPI; the generated Eventstream connection strings may contain independent SAS credentials. Keep runtime configs out of Git.
- This is a community solution accelerator, not an officially supported Microsoft product.

**Official references:** [Jumpstart catalog](https://jumpstart.fabric.microsoft.com/catalog/fpm-gateway-monitoring/), [upstream toolbox](https://github.com/microsoft/fabric-toolbox/tree/main/monitoring/fabric-platform-monitoring), [source package v0.2.3](https://github.com/ecotte/fabric-platform-monitoring/tree/v0.2.3/gateway-monitoring).
