# Fork changes and upstream comparison

**Scope:** only \`monitoring/fabric-platform-monitoring/gateway/\`. The existing project \`README.md\`, Capacity, Inventory, Activity Events, and Fabric deployment artifacts are unchanged.

## Revisions and provenance

| Artifact | Reference | What changed | Verified? |
| --- | --- | --- | --- |
| \`poc-notebooks/Gateway Monitoring Setup.ipynb\` | [Jumpstart v0.2.3 original source](https://github.com/ecotte/fabric-platform-monitoring/blob/v0.2.3/gateway-monitoring/src/fabric-platform-monitoring/setup/Gateway%20Monitoring%20Setup.Notebook/notebook-content.py) | Uploaded local notebook replaced \`fab\` subprocess commands with \`requests\` against Fabric REST; ID resolution by actual Eventstream display name; selection of unique \`CustomEndpoint\`, HTTP status check, auth host restriction, pagination and explicit JSON errors; writes \`builtin/config.json\` using \`notebookutils.nbResPath\` | REST approach used in POC, but sanitized committed notebook has **not been rerun** |
| Same Setup notebook — fork hardening | Uploaded user notebook | Removes remaining \`sempy.fabric\` import (previously caused \`NoAvailableProvider\`) using \`notebookutils.runtime.context["currentWorkspaceId"]\`; adds correctly spelled \`ServicePrincipal.TenantId\` in addition to legacy \`TennatId\`; changes illustrative poll intervals from **1/5** to **30/60 sec**; comments out unused CLI installation step | **New edits — require revalidation** |
| \`poc-notebooks/Jumpstart Installer - Gateway Monitoring.ipynb\` | [Jumpstart install instructions](https://jumpstart.fabric.microsoft.com/catalog/fpm-gateway-monitoring/) | Preserves the uploaded \`jumpstart.install("fpm-gateway-monitoring")\` and optional \`update_existing=True\` retry command; replaces hardcoded Workspace ID with placeholder and clarifies overwrite risk | Original uploaded notebook included failed/conflicting installer executions; sanitized archive is **not a successful deployment test** |
| \`PowerShellScript/Run-GatewayHeartbeat.ps1\` | [Official toolbox main](https://github.com/microsoft/fabric-toolbox/blob/main/monitoring/fabric-platform-monitoring/gateway/PowerShellScript/Run-GatewayHeartbeat.ps1) | **One-line fix:** \`Add-MsgEventHub -msg $body\` → \`-msg $msg\`, matching \`$msg = $processMsg | ConvertTo-Json\` | **Locally validated:** HTTP 201, KQL heartbeats |
| \`PowerShellScript/Setup-UpdateConfiguration.ps1\` | Official toolbox main | **Not modified in fork.** The local workflow used the stock script, and corrected its input and resulting config. | Used successfully on test VM |
| \`PowerShellScript/Get-DataGatewayInfo.ps1\` | Official toolbox main | **Not modified in fork**; uses \`ServicePrincipal.TenantId\` from config; Gateway SPN admin role was added | GET APIs and Eventhouse metadata verified |
| \`PowerShellScript/Run-UploadGatewayLogs.ps1\`, \`modules/*.psm1\` | Official toolbox main | **Not modified**; used against copied logs via an isolated POC config | Static-copy ingest proven, unattended active-file collection **not proven** |
| Task Scheduler XML | Official toolbox main | **Not modified in Git.** Equivalent tasks created manually with corrected paths, 30/60-second config and appropriate run settings | Heartbeat + NodeInfo validated; UploadLogs disabled |

## Functional delta: uploaded Setup notebook vs Jumpstart

- Original: \`%pip install ms-fabric-cli\`, \`fab get\`, \`fab api\`; CLI textual errors were often suppressed with \`silently_continue=True\`, followed by \`json.loads\` on non-JSON output.
- Uploaded revision: direct Fabric REST with existing notebook user's \`pbi\` token, \`GET /workspaces/{id}/items\`, \`/eventstreams/{id}/topology\`, \`/sources/{sourceId}/connection\`; no \`fab\` CLI runtime requirement in the main setup code.
- Uploaded revision: requires exactly one CustomEndpoint source and unambiguous Eventstream names instead of selecting \`sources[0]\`.
- Uploaded revision: checks HTTP error codes and parses direct JSON response (no CLI \`text\` envelope); does not print Event Hub access keys.
- Fork-only hardening beyond the uploaded revision: runtime-context workspace ID, dual \`TenantId\`/\`TennatId\` keys, 30/60-second example values, and removal of execution outputs.

**Important:** the precise original installer code is distributed by the \`fabric-jumpstart\` package. The supplied Installer Notebook is a usage/retry record, **not** an upstream code patch. We do not attribute deployment success to its captured executions.

## Runtime-only adjustments (not checked into Git)

- Target gateway log location was discovered under the **service account's local profile**, not the hard-coded PBIEgwService default.
- The generated config initially had \`TennatId\` but \`Get-DataGatewayInfo.ps1\` required \`TenantId\`. In the original POC this was fixed with \`Add-Member\` in \`Config.json\`.
- \`SecretText\` was encrypted by \`ConvertTo-SecureWithMachineKey\`; the encrypted ciphertext was verified against the original secret **without printing it**.
- Service principal was added as the **target Gateway Cluster Admin** after API returned zero gateways; result then showed one cluster/node.
- Semantic Model \`KustoURL\` / \`KustoDB\` parameters and **Edit credentials** were corrected through Fabric UI.
- Task Scheduler action split correctly into \`pwsh.exe\`, arguments, and unquoted \`Start in\`; XML templates were not directly imported.
- Runtime \`Config.poc.json\` pointed to copied logs. It was **never committed**, as it includes credentials.

## Known divergences / review points

- The Setup Notebook still contains the upstream \`TenatId\` Python variable and \`TennatId\` JSON property for backward compatibility. Verify downstream requirements before removing either.
- Setup notebook's \`x-ms-fabric-skill\` header was retained from the supplied revision; confirm whether it is needed with current Fabric REST versions.
- Direct REST calls and notebook runtime context must be re-tested on the target workspace/tenant after restoring sanitized placeholders.
- Re-running Jumpstart with \`update_existing=True\` can overwrite Fabric items and custom changes.
- The official README's description of "tail" should not be interpreted as byte-offset incremental reading: current PowerShell Uploader operates at **file level** and calls \`Move-Item\`.
