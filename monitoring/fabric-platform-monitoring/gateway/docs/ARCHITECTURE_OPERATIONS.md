# Gateway Monitoring — architecture, log lifecycle, and production decisions

## 1. Verified Fabric data path

The POC validated this flow:

\`\`\`text
Gateway VM
  ├─ Get-DataGatewayInfo.ps1 (manual + scheduled)
  ├─ Run-GatewayHeartbeat.ps1 (continuous, ~30 s)
  └─ Run-UploadGatewayLogs.ps1 (manually on static COPIES only)
        ├─ GatewayMonitoringHeartbeat.Eventstream
        └─ GatewayMonitoringReports.Eventstream
           ↓
Fabric Platform Monitoring.Eventhouse / Gateway Monitoring KQL Database
  ├─ GatewaysHeartbeat → GatewayNodeStatusWithInfo
  └─ GatewayReports-Raw → parsed performance tables and query functions
           ↓
Gateway Monitoring Semantic Model (DirectQuery) → Power BI Report
\`\`\`

See the [editable Draw.io diagram](Gateway-Monitoring-POC.drawio), [official Jumpstart diagram](https://jumpstart.fabric.microsoft.com/images/diagrams/fpm-gateway-monitoring_light.svg), and [existing toolbox data-logic image](../../Images/05%20-%20Gateway%20Report%20Data%20Logic.png).

**Verified:** Gateway metadata, heartbeat, System Counters, Jobs/Queries from historical samples, Report credentials and KQL. **Not verified:** Running Jobs / drillthrough / continuous producer / multi-node.

## 2. All Gateway Performance Report types and selection

Original \`UploadGatewayLogs.psm1\` scans **recursively** for \`*Report_*.log\` excluding \`*_f.log\`. These eight types were observed in the POC source directory:

| Report | Meaning / possible analysis |
| --- | --- |
| QueryStartReport | Start metadata, request tracking, data source, potentially query text |
| QueryExecutionReport | Completion, duration, success/error |
| QueryExecutionAggregationReport | Aggregated query processing |
| SystemCounterAggregationReport | CPU, memory, system counters |
| MashupEvaluationReport | Mashup evaluation activity |
| ConcurrentOperationAggregationReport | Concurrent operation metrics |
| OpenConnectionReport | Connection activity |
| QueryStreamingReport | Query streaming |

The Uploader attempts to send **all matching reports** when \`EventHubs.UploadReports\` is true, whether or not the stock Power BI report visualizes each type. Ordinary service verbose logs such as \`GatewayErrors*.log\` are a **separate** optional Lakehouse path when \`Lakehouse.UploadLogs\` is enabled.

## 3. Why direct \`Move-Item\` is not approved yet

Source code: [UploadGatewayLogs.psm1](https://github.com/microsoft/fabric-toolbox/blob/main/monitoring/fabric-platform-monitoring/gateway/PowerShellScript/modules/UploadGatewayLogs.psm1), [Utils.psm1](https://github.com/microsoft/fabric-toolbox/blob/main/monitoring/fabric-platform-monitoring/gateway/PowerShellScript/modules/Utils.psm1).

Current behavior for a matching report:

1. \`Move-Item\` source CSV to its sibling \`Report\Temp\*_x.log\` (does not make a copy).
2. \`Add-LogToEventHub\` reads CSV, batches up to ~2000 rows and reduces payload near 980 KiB, then submits via Event Hubs-compatible HTTP REST.
3. \`Merge-ReportFiles\` appends to \`Report\History\*_f.log\`, then deletes the processed intermediate file.
4. \`Remove-OldReportFiles\` removes matching old local history based on \`ReportRetention\` (default 10 days). Cleanup is triggered during processing; it is **not** a strict wall-clock purge service.

The POC used \`D:\Ang\FabricGatewayMonitoring\poc-input\Report\Temp\` and \`...\History\`, so the real Gateway source was unaffected.

### Evidence of active append

On **12 Nov 2026** filenames still contained \`20261019T110120\` while their sizes grew over repeated five-minute snapshots:

| Report | First → last bytes | Delta |
| --- | --- | ---: |
| SystemCounterAggregationReport | 6,668,473 → 6,670,766 | +2,293 |
| QueryExecutionAggregationReport | 561,365 → 563,900 | +2,535 |
| ConcurrentOperationAggregationReport | 374,348 → 374,568 | +220 |
| QueryStartReport | 4,482,959 → 4,482,959 | 0 during sample |

This **proves the same named files are actively appended**. It does **not** prove what happens if an external process renames/moves an open file. An open-handle \`Move-Item\` may fail or affect subsequent logging; Gateway rotation/recreation after an external move has **not been tested**, so no one should assume a new daily file automatically appears. Gateway-side file size/count settings explain regular rotation but not behavior under an external move.

### Retention caveats

- The gateway config had \`ReportFileCount=10\` and \`ReportFileSizeInBytes=104857600\` (~100 MiB).
- \`ReportRetention=10\` applies to **local uploader History**, not Fabric Eventhouse retention or original Gateway retention.
- An Eventstream HTTP **201** confirms acceptance, not end-to-end exactly-once. Partial batch submission and retry can create duplicate rows; the original uploader is not a full transactional ingestion system.

## 4. Candidate collection designs (none implemented yet)

| Approach | Pros | Limitations / decision |
| --- | --- | --- |
| Original file-level Uploader | Already proven on static copies, lowest code cost | Moves live source files; **not approved** pending lock/rotation/recovery validation |
| Copy full files periodically | Small script | Re-uploads previously seen rows and wastes capacity; **not recommended** |
| PowerShell or C# read-only tail with checkpoint | No source mutation, uses HTTPS 443 direct REST | Must implement CSV quoted multiline framing, byte offsets, file identity, recovery, batch commit, secure auth; POC required |
| Fluent Bit / Vector / OpenTelemetry Filelog | Mature file tail and checkpoints | Verify multi-line quoted CSV records, Windows file sharing, exact Eventstream message envelope, 443 auth, idempotency |
| Azure Monitor Agent / enterprise logs platform | May fit existing company-wide operations | Additional infrastructure, cost and Fabric schema adaptation |
| Kafka broker | Durable message intermediary | Does **not** read Gateway files; adds infra/ports. Not justified solely for this use case |

**Proposed technical gate, not a completed implementation:** read-only shared FileStream on the active reports → CSV logical-record parser (not newline splitting) → durable per-file offset/checkpoint → HTTP REST into existing Reports CustomEndpoint → KQL duplicate strategy. Checkpoint must advance **only after server acknowledgement**, accepting at-least-once behavior. A stable EventId added to the raw dynamic record is insufficient to ensure the stock parsed KQL tables or semantic model deduplicate; changes require separate testing.

Evaluate the existing corporate telemetry/logging platform before building another Windows service. For the tested corporate network, HTTPS/443 is confirmed; a Kafka TCP path should not be presumed available.

## 5. Scheduling and recovery design

- **Heartbeat:** 30 seconds; long-running PowerShell task at startup, no GUI console required, state \`Running\` is expected. One-minute KQL offline threshold means network jitter can create transient "offline" states; monitor false positives.
- **NodeInfo:** at startup plus weekly; exits to Ready and last result 0.
- **Report polling:** suggested **60 seconds**; current scheduler UploadLogs task **disabled**, because production tail/file safety is unresolved.
- The toolbox XML templates contain legacy paths/SIDs and **72h \`ExecutionTimeLimit\`**. They are reference templates, not deploy-as-is production task definitions.
- Monitor Task Scheduler History, last heartbeat in KQL, disk backlog, auth expiry, failed sends, duplicate rates and source-file retention.
- A host reboot/VM failover must not silently reprocess the full file or lose unacknowledged bytes; this still needs an integration test.

## 6. Security and privacy

- \`config.json\` stores Eventstream Event Hub-compatible connection strings, potentially including **SAS credentials**; keep out of repo and limit file ACLs.
- \`SecretText\` is Windows **LocalMachine DPAPI** encrypted, not plain text, but still sensitive and tied to the machine. Avoid copying encrypted blob across VMs; provision each host separately and rotate credentials through approved systems.
- Gateways can log QueryText, connection paths, data source names, IDs, errors and potentially sensitive parameters. Review data classification and retention before uploading every type to a shared Eventhouse.
- Give Scheduled Tasks the least privileges required to read their data and authenticate; separate interactive POC identities from long-term service identities.
- Define Eventhouse/KQL table retention independently from local \`ReportRetention\`; evaluate Fabric capacity and DirectQuery refresh cost with more nodes.

## 7. Acceptance criteria before marking production-ready

- Source reports untouched through long-running collection, including active writes, file rotation, truncation, and gateway restart.
- All eight required CSV types parsed correctly, including quoted/multi-line data and partial UTF-8 writes.
- Restart/network outage/retry/idempotency tests with counted events, bounded disk buffer and observable failures.
- Service account permission review, secret rotation, governance and privacy review.
- Multi-node cluster test; no report double counting; Report-specific visuals checked.
- Task/collector health alerts and rollback procedure documented.

**Status:** these criteria are **open**, and this branch records technical decisions and POC results; it does not introduce an unvalidated collector or claim production readiness.
