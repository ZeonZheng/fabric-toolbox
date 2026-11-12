# Gateway Monitoring — repeatable deployment runbook

**Purpose:** reproduce the single-node POC; make each verification gate observable before progressing. **Not a production-ready active-log collection procedure.** Use [POC status](POC_STATUS.md) and [operations](ARCHITECTURE_OPERATIONS.md).

## 0. Scope, prerequisites, and guardrails

- Only **On-premises data gateway** was tested. Virtual Network Gateway, Capacity Events, Activity Events, Inventory, multi-cluster and end-to-end active log tailing are out of scope.
- Windows VM has Gateway installed; PowerShell **7+** (\`pwsh.exe\`), ability to read the Gateway service account's logs, and HTTPS/443 access to the Fabric Eventstream CustomEndpoint.
- Create/select a Fabric workspace with sufficient capacity and the rights to provision Notebook, Eventstreams, Eventhouse, KQL Database, Semantic Model and Report.
- For \`Get-DataGatewayInfo.ps1\`: add monitoring **service principal as Admin of the target Gateway Cluster**; also configure relevant tenant and workspace permissions per the [Jumpstart instructions](https://jumpstart.fabric.microsoft.com/catalog/fpm-gateway-monitoring/).
- **Never commit** \`config.json\`, SAS connection strings, client secrets, gateway query logs, \`state.json\`, or task credentials. Use fresh environment IDs.
- Review whether the scripts may write into the service-account Gateway log directory. Until approved, run Uploader **only on copied reports**, with isolated config.

## 1. Deploy the Gateway Jumpstart module

In the destination Fabric workspace Notebook:

\`\`\`python
%pip install -q fabric-jumpstart
import fabric_jumpstart as jumpstart
jumpstart.install("fpm-gateway-monitoring")
\`\`\`

If there are existing items, **inspect conflicts first**. The supplied [installer notebook](../poc-notebooks/Jumpstart%20Installer%20-%20Gateway%20Monitoring.ipynb) records the optional \`update_existing=True\` command, **but its uploaded execution ended with errors**; it does not demonstrate a clean reinstall. Updating may overwrite existing Fabric customizations.

Verify both Eventstreams (\`GatewayMonitoringHeartbeat\`, \`GatewayMonitoringReports\`), Eventhouse, Gateway Monitoring KQL Database, Semantic Model, Report and Setup Notebook exist.

## 2. Generate a runtime configuration

Use the local [sanitized Setup Notebook](../poc-notebooks/Gateway%20Monitoring%20Setup.ipynb) as a **fork adaptation** of the deployed Jumpstart Notebook. It uses direct Fabric REST API calls (rather than \`fab\` CLI) for topology and CustomEndpoint connection keys. Before executing, populate **\`TenatId\` and \`AppId\`** in the parameters cell.

- Fork example configuration: **HeartbeatInterval = 30 sec**, **ReportSendInterval = 60 sec**, **ReportRetention = 10 days**.
- The committed version adds the correctly spelled \`TenantId\` alongside upstream \`TennatId\`. These final portability edits must be revalidated in your Fabric workspace.
- The notebook must resolve **one CustomEndpoint** source per Eventstream and write \`builtin/config.json\`.
- Download that file to the Gateway VM **privately**. It contains Event Hub compatible connection strings/keys; it is a credential-bearing artifact.

## 3. Prepare the Gateway VM

Preserve the existing Toolbox script/module structure:

\`\`\`text
D:\Ang\FabricGatewayMonitoring\
  configs\Config.json            # runtime private; NOT in Git
  modules\Utils.psm1
  modules\UploadGatewayLogs.psm1
  Run-GatewayHeartbeat.ps1
  Get-DataGatewayInfo.ps1
  Run-UploadGatewayLogs.ps1
  Setup-UpdateConfiguration.ps1
  logs\
  poc-input\Report\              # isolated test copies, not the source
\`\`\`

Check runtime and modules in **PowerShell 7**:

\`\`\`powershell
$PSVersionTable.PSVersion
Get-Module -ListAvailable -Name Az.Accounts,Az.Storage,DataGateway,MicrosoftPowerBIMgmt |
  Select-Object Name, Version, Path
\`\`\`

The module list may include versions installed for both PowerShell 7 and Windows PowerShell; it does not mean installation failed. In the POC, all four were already available, so installation prompts were answered \`N\`. \`Az.Accounts\` and \`Az.Storage\` are in the original heartbeat/upload \`#requires\` directives even with Lakehouse upload disabled.

Identify the **actual service account**:

\`\`\`powershell
Get-CimInstance Win32_Service |
  Where-Object { $_.Name -eq "PBIEgwService" } |
  Select-Object Name, State, StartName
\`\`\`

Inspect the GatewayCore DLL config in the actual install path for \`ReportFilePath\`. \`%LOCALAPPDATA%\` is resolved **for the Gateway service identity**, not necessarily the interactive user. POC used \`C:\Users\powrbi\AppData\Local\Microsoft\On-premises data gateway\Report\`. Find \`*Report_*.log\` in that folder. **Additional logging is not required** for these performance reports.

## 4. Initialize on the VM and validate the secret

\`\`\`powershell
Set-Location 'D:\Ang\FabricGatewayMonitoring'
.\Setup-UpdateConfiguration.ps1
\`\`\`

- Answer \`N\` to module installation if already present.
- Set the **parent Gateway log directory** (one level above \`Report\`), not \`D:\Ang\FabricGatewayMonitoring\logs\`.
- Confirm Gateway ID is inferred from the report CSV; do not invent an ID.
- Paste the **Client Secret Value** at \`Read-Host -MaskInput\`. On the tested PowerShell host, **Ctrl+V pasted only one character** into MaskInput, whereas Windows console title bar → Edit → Paste produced the full input. Verify using a dummy string before entering the real secret.
- \`ServicePrincipal.SecretText\` is **DPAPI encrypted**, so its stored Base64 differs from the secret. Do not display either the original or decrypted secret. The original local test confirmed \`DecryptionSuccessful / SecretValidLength / SecretMatches\` all true.
- If using a config created by an old Setup Notebook, add \`ServicePrincipal.TenantId\` equal to \`TennatId\`: the metadata script reads **TenantId**, while upstream generator has the misspelled key. The fork notebook writes both.

No Task Scheduler task should run this setup interactively.

## 5. Metadata and heartbeat verification

\`\`\`powershell
.\Get-DataGatewayInfo.ps1
.\Run-GatewayHeartbeat.ps1
\`\`\`

The fork heartbeat script includes the one-line \`$msg\` fix. Heartbeat is an infinite loop; use Ctrl+C after a manual test.

HTTP \`201 Created\` indicates Eventstream acceptance; **verify the KQL result**:

\`\`\`kusto
['GatewayReports-Raw']
| where logType == "GatewayNodeInfo"
| top 5 by ingestion_time() desc
\`\`\`

\`\`\`kusto
GatewayNodeInfo
| where isnotempty(nodeId)
| project nodeId, clusterName, machine, version, logDate
| order by logDate desc
\`\`\`

\`\`\`kusto
GatewaysHeartbeat
| summarize LastSeen=max(ingestion_time()),
            EventsLast5Min=countif(ingestion_time()>ago(5m))
            by GatewayId
| extend AgeSeconds=datetime_diff('second',now(),LastSeen)
\`\`\`

If metadata is present but \`clusterName/machine\` are null, verify the SPN is **Gateway Cluster Admin**. The POC observed zero visible gateways before, one afterwards. HTTP \`201\` alone is not proof of complete metadata.

## 6. Static-copy report-log test ONLY

Copy selected historical reports from the active Gateway \`Report\` directory into \`poc-input\Report\` (use **Copy-Item**, never move originals), and create a private \`configs\Config.poc.json\` whose \`GatewayLogsPath\` points to \`poc-input\`. Keep EventHubs report upload enabled and Lakehouse uploads disabled.

\`\`\`powershell
.\Run-UploadGatewayLogs.ps1 -configFilePath '.\configs\Config.poc.json'
\`\`\`

The Uploader processes files into its own \`Temp\` / \`History\` folders; it loops continuously. **Ctrl+C** after the copied files are processed. Test KQL:

\`\`\`kusto
['GatewayReports-Raw']
| where logType in ("QueryStartReport","QueryExecutionReport",
                    "QueryExecutionAggregationReport","SystemCounterAggregationReport")
| summarize Messages=count(), DataRows=sum(array_length(log)) by logType
\`\`\`

\`\`\`kusto
QueryStartReport | summarize Records=count(), First=min(QueryExecutionStartTimeUTC), Last=max(QueryExecutionStartTimeUTC)
\`\`\`

\`\`\`kusto
QueryExecutionReport | summarize Records=count(), First=min(QueryExecutionEndTimeUTC), Last=max(QueryExecutionEndTimeUTC)
\`\`\`

Compare Raw and parsed counts for the **same** log type and date range; HTTP \`201\` does not prove every downstream transform succeeded.

## 7. Semantic Model / Report setup (do not skip credentials)

1. KQL Database → copy its **Query URI**, not ingestion URI.
2. Gateway Monitoring Semantic Model → Settings → Parameters: set **KustoURL** to the deployed Query URI and **KustoDB** to the deployed database name.
3. **Settings → Data source credentials → Edit credentials**, or edit the corresponding Cloud Connection. Sign in with an identity that has access to the KQL Database.
4. Open Report → Gateways, System Counters, Jobs, Queries. Use a date covering **event timestamps** in the copied historical files (not only ingestion date). Historical QueryStart + QueryExecution files are required for Jobs/Queries to show detail.
5. Job Details and Running Jobs need separate validation; do not mark them complete based on Jobs alone.

## 8. Task Scheduler background operation

Create a \`FabricGatewayMonitoring\` folder, use **Create Task**, not **Create Basic Task**.

**For all tasks:**
- **Program/script:** \`"C:\Program Files\PowerShell\7\pwsh.exe"\` (quoted path).
- **Start in:** \`D:\Ang\FabricGatewayMonitoring\` (no quotes).
- **Arguments:** shown below; **do not concatenate arguments into Program/script**.
- Use an approved execution identity with required file/network permissions, and **Run whether user is logged on or not**. Remove default **72-hour task execution limit** for long-running tasks; "Do not start a new instance"; configure failure restart and enable Task History.
- If Task Scheduler warns it found arguments in Program/script, **do not select Yes** (the POC observed it split Program to \`C:\Program\`). Fix the field separation instead.

| Task | Arguments | Trigger | Expected result |
| --- | --- | --- | --- |
| Gateway-Heartbeat | \`-NoLogo -NoProfile -NonInteractive -File "D:\Ang\FabricGatewayMonitoring\Run-GatewayHeartbeat.ps1"\` | At startup (optional 1 minute delay) | Remains **Running**; \`267009\` = 0x41301 (running) |
| Gateway-NodeInfo | \`-NoLogo -NoProfile -NonInteractive -File "D:\Ang\FabricGatewayMonitoring\Get-DataGatewayInfo.ps1"\` | At startup + weekly | Returns to **Ready**; result **0** |
| Gateway-UploadLogs | \`-NoLogo -NoProfile -NonInteractive -File "D:\Ang\FabricGatewayMonitoring\Run-UploadGatewayLogs.ps1" -configFilePath ".\configs\Config.poc.json"\` | At startup | **Disabled after isolated scheduler test**; not an active-log service |

Do **not** enable \`Gateway-UploadLogs\` with the real log-directory \`Config.json\` as a production workaround. The original script moves active source files (see [operations](ARCHITECTURE_OPERATIONS.md)).

\`\`\`powershell
$taskPath = '\FabricGatewayMonitoring\'
Get-ScheduledTask -TaskPath $taskPath | Select-Object TaskName, State
Get-ScheduledTask -TaskPath $taskPath -TaskName 'Gateway-Heartbeat' |
  Get-ScheduledTaskInfo | Select-Object LastRunTime, LastTaskResult
\`\`\`

## 9. Handover checklist

Confirm KQL receives new Heartbeat and NodeInfo without an open PowerShell window; preserve Uploader disabled; note gateway owner/service account, ACLs, Fabric workspace and KQL Query URI in your **internal** deployment inventory (not Git); register failure alerts and plan incremental read-only report collection before production.
