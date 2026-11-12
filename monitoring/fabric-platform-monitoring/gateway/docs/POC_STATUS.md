# Gateway Monitoring POC — Evidence and status

**Recorded:** 12 Nov 2026 (Singapore time). **Scope:** One On-premises Data Gateway cluster and one node on a Windows Server VM. Values below are observed POC outputs, not a tenant-wide benchmark.

## Validation matrix

| Component | Observed evidence | Assessment |
| --- | --- | --- |
| Jumpstart deployment | Gateway Eventstreams, Eventhouse, KQL Database, Semantic Model and Report existed and were used | Functionally deployed; the uploaded Installer Notebook contains errors from separate attempts — not proof of a clean install |
| PowerShell runtime | PowerShell **7.6.9** and Az.Accounts, Az.Storage, DataGateway, MicrosoftPowerBIMgmt available | Verified on test VM |
| Gateway Performance Reports | Eight types of \`*Report_*.log\` observed in service-account-local \`Report\` directory | Verified; no "Additional logging" enabled |
| Local configuration | Gateway ID inferred from CSV, local-machine encrypted secret verified by decrypt-and-compare | Verified locally, not a validation of all application entitlements |
| Gateway Metadata | Service principal added as target Gateway cluster admin; GET /v1/gateways changed from **0** visible to **1** and matching node **1** | Verified |
| Metadata delivery | Reports Eventstream accepted \`GatewayNodeInfo\` (HTTP **201**) and KQL populated cluster/node/version | Verified |
| Heartbeat | HTTP **201**, \`GatewaysHeartbeat\` rows, \`Responding=true\`; scheduled task \`Running\` / result **267009** | Verified |
| NodeInfo scheduled task | Last task result **0**, \`Ready\` after completion | Verified |
| System counters and aggregate query performance | Test copy of two report types ingested; see counts below | Verified for static historical logs |
| Query execution detail | Test copy of QueryStart + QueryExecution report types ingested; see counts below | Verified for static historical logs |
| Power BI | Gateways, System Counters, Jobs and Queries pages rendered, after Kusto parameter and **Edit credentials** | Verified for tested pages |
| Running Jobs / drill-through | Not demonstrated | **Not verified** |
| Production/continuous uploader | Scheduled \`Gateway-UploadLogs\` deliberately **Disabled** | **Not deployed / not verified** |
| Multi-cluster / multiple nodes | Not attempted | **Not verified** |
| Capacity Events / Activity Events / Fabric Inventory | Not attempted in this POC | **Not verified** |

## Observed ingest counts (static-copy tests)

| Report type | Eventstream messages | Parsed KQL rows |
| --- | ---: | ---: |
| QueryExecutionAggregationReport | 2 | 1,306 |
| SystemCounterAggregationReport | 27 | 53,538 |
| QueryExecutionReport | 3 | 2,055 |
| QueryStartReport | 7 | 2,055 |

The first two sum to **29 successful HTTP requests / 54,844 parsed rows**. Raw and parsed KQL counts matched for these tested datasets. This does **not** establish loss-free continuous production collection.

A filtered report view on **19 Oct 2026** showed **3 Jobs**, **5 Queries**, **1 node**, and **12.44 seconds P90 duration**. Filters determine visible counts; they are not full-timeline totals. CPU and memory curves displayed across historical days.

## Example POC runtime facts (not configuration defaults)

- Gateway cluster display name: \`POC Data Gateway\`.
- Node display name: \`Power BI Gateway POC\`.
- Windows gateway service used a dedicated technical account, not the default PBIEgwService profile.
- Report path on test host: \`C:\Users\powrbi\AppData\Local\Microsoft\On-premises data gateway\Report\`; use discovery on other nodes.
- Gateway node/cluster version observed: \`3000.338.2\`.
- Gateway ID is intentionally omitted from reusable templates; obtain it from target Node.
- Gateway metrics files retained a filename containing \`20261019\` while \`LastWriteTime\` and sizes increased on **12 Nov 2026**: confirmed *active append*, not daily rotation.
- Scheduled Heartbeat: approximately 30-second interval, event age **17 seconds** and **7 events within five minutes** shortly after starting the task. Seven was observed, not a steady-state SLA.

## Outstanding work — do not mark complete

1. Choose and test a safe **read-only** collector for continuously appended report CSV files. Original Uploader uses \`Move-Item\`, so it is not approved for active source logs.
2. Test partial/quoted multiline CSV records, file rotations, checkpoint recovery, retries, and downstream duplicate handling.
3. Run an extended unattended availability test; establish telemetry alerts, disk/retention budgets, secure execution identity and secret rotation.
4. Verify Running Jobs, Job Details, multi-node/multi-cluster scale, data completeness and operational ownership.
5. Review client-secret handling, query text/DSN exposure, and data retention before broad rollout.

## Source evidence

This record comes from live PowerShell outputs, Fabric KQL results, and screenshots collected during **11–12 Nov 2026** in the implementation conversation. The screenshots are intentionally **not** committed because query text, data-source URLs and internal identifiers can be sensitive.
