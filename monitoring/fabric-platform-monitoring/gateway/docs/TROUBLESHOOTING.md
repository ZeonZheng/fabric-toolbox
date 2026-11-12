# Gateway Monitoring — issues encountered and known fixes

These are the actual POC findings, with **verified fixes separated from unverified ideas**.

| Symptom / error | Cause / diagnostic | Verified resolution / caveat |
| --- | --- | --- |
| \`NoAvailableProvider: lack of valid ContextProvider\` importing \`sempy.fabric\` in Fabric Notebook | Semantic Link environment-context discovery fails in a given notebook runtime | Fork Notebook removes the import, using \`notebookutils.runtime.context["currentWorkspaceId"]\`. **This latest fork change has not been rerun**; earlier direct REST flow was tested separately. |
| \`JSONDecodeError: Expecting value: line 1 column 1\` when reading Eventstream topology | Original \`run_fab_command(... silently_continue=True)\` can hide Fabric CLI failures and feed stdout non-JSON to \`json.loads\`; the local investigation observed \`fab\` invalid audience | Uploaded Setup Notebook replaces command invocation with direct Fabric REST, explicit HTTP error reporting and CustomEndpoint selection. Check Fabric permissions/HTTP code; do not parse suppressed CLI errors. |
| \`Install-Package: No match ... Az.Accounts\` | Selected \`Y\` even though module already installed; PowerShell repository discovery unavailable | \`Get-Module -ListAvailable\`; use \`N\` when installed. Do not blindly run \`Install-Module -Scope AllUsers\` on managed VM. |
| \`Set the Gateway Id Manually:\` when using \`D:\...\logs\` | Script was pointed at its **own error-log directory**, not the Gateway's live Performance Report folder | Read actual \`ReportFilePath\` and service account; POC used \`C:\Users\powrbi\AppData\Local\Microsoft\On-premises data gateway\` (parent of \`Report\`). |
| Cannot find default PBIEgwService log path | A technical service account owns Gateway process, with \`%LOCALAPPDATA%\` under its Windows user profile | \`Get-CimInstance Win32_Service\` and inspect GatewayCore \`ReportFilePath\`. Do **not** turn on Additional logging merely to get Performance Reports. |
| \`Read-Host -MaskInput\` shows one \`*\` after Ctrl+V | On tested PowerShell Console, Ctrl+V inserted **1 actual character** despite normal prompt paste being fine | Test with \`Testing123456789\`; use title bar → Edit → Paste or another **verified** method; after encrypting, verify decrypted string equality without displaying it. Manual entry not required. |
| \`SecretText\` differs from Entra App Secret | This field stores DPAPI encrypted value via \`ConvertTo-SecureWithMachineKey\` | Expected. Data is machine-bound; do not paste plaintext into JSON or commit ciphertext. The POC verified length/equality, but validity requires real auth. |
| HTTP \`201 Created\` for NodeInfo but clusterName/machine are null | Eventstream accepts the message, but the SPN cannot enumerate target gateway metadata | Add the SPN as **Gateway Cluster Admin**. POC \`GET /v1/gateways\`: 0 → 1 visible cluster, and matched node 1; new KQL data populated. |
| \`TenantId\` missing even with \`TennatId\` populated | Jumpstart original generator misspells the field as \`TennatId\`; metadata script reads \`TenantId\` | POC manually added \`ServicePrincipal.TenantId\`. Sanitized fork Setup Notebook generates **both** fields. |
| \`GatewayMonitoring.log\` missing after successful run | Script logs errors but does not necessarily create a success log | Absence alone is not a failure; verify REST response and KQL rows. |
| Heartbeat script did not send payload correctly | Upstream script generates \`$msg\` but passes \`$body\` | **One-line fix committed**: \`Add-MsgEventHub -msg $msg\`. Verified by HTTP 201 and KQL \`Responding=true\`. |
| Task Scheduler launches \`C:\Program\` instead of \`pwsh.exe\` | Windows auto-split Program/script path at space after confirming argument-warning | Put quoted executable path **only** in Program; options in Arguments; unquoted root folder in Start in. Select **No** on split prompt and correct fields. |
| Task state \`Running\` / result 267009 | Long-running scheduled script | Expected for Heartbeat; \`NodeInfo\` should finish and be Ready/result 0. A Task status alone is not proof of ingestion; verify KQL freshness. |
| \`Gateway-UploadLogs\` takes time, repeatedly prints \`201\`, then \`Gateway Report log count: 0\` | It divides CSV records into batches and loops for new files | POC processed **29 requests / 54,844 parsed rows** for two initial report types. \`count: 0\` after copy processing is expected. |
| Report has Gateways/System Counters, but Jobs/Queries are blank | Only aggregation files were copied, and/or date filter misses historic data | Copy \`QueryStartReport\` and \`QueryExecutionReport\` for **static** tests and select the event date. POC then displayed 3 Jobs/5 Queries on 19 Oct 2026. |
| Semantic Model report visuals blank or inaccessible | Template may use developer \`KustoURL\`; credentials not configured | Set \`KustoURL\`, \`KustoDB\` and **Edit credentials / Cloud connection** to access target KQL Database. |
| Installer reports conflicting Fabric items or failed publish | Re-running install against existing workspace or failing item publish | Treat uploaded outputs as **failed attempts**. Inspect existing items and use \`update_existing=True\` only after understanding overwrite consequences. No automatic reinstall recommended. |

## Safe local checks

\`\`\`powershell
$root = 'D:\Ang\FabricGatewayMonitoring'
Get-Module -ListAvailable -Name Az.Accounts,Az.Storage,DataGateway,MicrosoftPowerBIMgmt |
  Select-Object Name, Version, Path
Get-ChildItem "$root\modules" -Filter '*.psm1'
Get-CimInstance Win32_Service | Where-Object Name -eq 'PBIEgwService' |
  Select-Object Name,State,StartName
Select-String "$root\Run-GatewayHeartbeat.ps1" -Pattern 'Add-MsgEventHub'
\`\`\`

Do not print full \`Config.json\`, bearer tokens, connection strings, QueryText, or secret input in shared issue comments. Pasting secret at a bare \`PS>\` prompt may also record it in command history.
