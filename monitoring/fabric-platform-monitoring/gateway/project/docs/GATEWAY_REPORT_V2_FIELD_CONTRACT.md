# Gateway Monitoring V2 — Implementation-ready field contract (Design Review)

> 2026-11-23 | **Status: visual HTML prototype + editable Draw.io wireframes created in current conversation; not yet an operational PBIR/PBIP or published Power BI report.** Artifacts are downloadable from the ChatGPT conversation; not uploaded to GitHub by this commit. All prototype numbers are synthetic.

## Verified asset baseline

- Original: `src/Gateway/Gateway Monitoring.Report/report.json` (PBIR-Legacy): **6 pages / 58 visual containers**.
- Model: `src/Gateway/Gateway Monitoring.SemanticModel/definition/tables/*.tmdl`, **DirectQuery**, dynamic M parameters `DateToSearch`, `DaysBack`, `SelectedGateways`, `SelectedClusters`.
- Source KQL: `GatewayNodeStatusWithInfo()`, `SystemCounters()`, `QueryExecutionUnified()`, `QueryConnections()`; the latter two use `QueryStartReport` and `QueryExecutionReport`, and SystemCounters uses `SystemCounterAggregationReport`.
- **Original Report does not directly consume** three P1-repaired `QueryStreamingReport`, `QueryExecutionAggregationReport`, `MashupEvaluationReport` Parsed tables. A new model extension would be necessary to add them to V2; intentionally not in V2 first pass.

## V2 five-page field/data contract

| V2 page | Visual or KPI | Current model field/measure | Source & implementation note |
| --- | --- | --- | --- |
| 01 Overview | Responding Nodes | `Gateway Node Info[Responding]` + `[Gateway ID]` | KQL `GatewayNodeStatusWithInfo`; new distinct-nodes-with-responding measure needed |
| 01 Overview | Total / failed queries | `Execution Measures[# Queries]`, `[# Queries (errors)]` | Existing DAX; validate under cluster/node/date filters |
| 01 Overview | P90 query duration | `Execution Measures[Perc. 0.9 Query Duration (s)]` | **P90 in seconds**, not average/milliseconds |
| 01 Overview | Gateway CPU trend | `System Counters[Avg. Counter Value]` + `AggregationEndTimeUTC` + `NodeName` | `CounterName=GatewayCpuPercent` and existing core normalization; validate % scale |
| 02 Gateway Health | Node inventory | `Gateway Node Info[Cluster Name]`, `[Node Name]`, `[Version]`, `[Logical Cores]`, `[Responding]` | NodeInfo/Heartbeat; `Memory Gb` is capacity, **not usage** |
| 02 Gateway Health | CPU/Memory | `System Counters[CounterName]`, `[Average]` and measure `[Avg. Counter Value]` | Verify which `CounterName` truly means memory utilization before labeling % |
| 03 Query Performance | Refresh vs DirectQuery | `Execution Measures[# Queries (Refresh)]`, `[# Queries (DirectQuery)]` | Current DAX |
| 03 Query Performance | P90 / Maximum | `Execution Measures[Perc. 0.9 Query Duration (s)]`, `[Max. Query Duration (s)]` | Seconds. Existing `Query Avg. Execution - ... (ms)` names are **incorrect** |
| 03 Query Performance | Slow query table | `Query Execution[QueryExecutionStartTimeUTC]`, `[QueryExecutionDuration_ms_]`, `[NodeName]`, `[OperationType]`, `[Success]` | Per-row duration unit conversion; confirm query response and sorting |
| 04 Failures | Error count / rate | `Execution Measures[# Queries (errors)]`, `[# Queries]` | If accepted, add DAX `Failure Rate = DIVIDE([# Queries (errors)], [# Queries])` |
| 04 Failures | Error detail | `Query Execution[ErrorMessage]`, `[Success]`, `[ServiceName]`, `[NodeName]` | Access controls; mock-up error categories are **not** a current model field |
| 05 Job Detail | Drillthrough context | `Query Execution[RootActivityId]`, `[QueryTrackingId]`, `[RequestId]`, `[CurrentActivityId]`, Start/End Time | Validate Join cardinality and real drillthrough; sensitive IDs and QueryText not exposed by default |

### Known semantic defects and exclusions

1. Original `Execution Measures[Query Avg. Execution - DirectQuery (ms)]` and `[Query Avg. Execution - Refresh (ms)]` call `[Perc. 0.9 Query Duration (s)]`, therefore **P90 seconds**, not Avg ms. V2 must correct label (or add accurate measures).
2. Original `[# Queries (running)] = [# Queries]-[# Queries (success)]-[# Queries (errors)]` is an inferred remainder, not authoritative real-time running; do **not** display as an exact current-active metric.
3. V2 does not depend on GW-009 deferred Mashup fields or Raw-only missing Parsed fields. The prototype's error classification and timestamps are **illustrative only** and require real field validation before implementing.
4. The repo is PBIR-Legacy and `definition.pbir` has placeholder-looking connection metadata; **do not claim current GitHub copy alone yields an openable live-connected V2**.

## Design implementation sequence

1. **Design completed:** 5-page responsive HTML prototype, 5-page editable Draw.io wireframes, screenshot QA; simulated selectors for period, cluster, node. No operational KQL queries or Fabric writes were made.
2. **Next engineering action:** scaffold an independent **Enhanced PBIR / PBIP** report using Power BI Desktop or the Microsoft `@microsoft/powerbi-report-authoring-cli` scaffold command; retain accurate `$schema` versions and bind to the deployed model. Do not hand-author root project shell from guessed schema.
3. Add first page **Overview** using verified model fields and minimal new DAX, then validate under Live DirectQuery and dynamic M parameters; expand page by page.
4. Acceptance: 5 page navigation, KPI mapping and units, filter propagation, drillthrough, original/new numeric parity for a named time window, no sensitive data exposure, acceptable DirectQuery performance, visual accessibility.
5. **Do not** overwrite the original Report, change Uploader/Eventhouse schema/PROD, or present this as already deployed.

Related: [V2 Blueprint](GATEWAY_REPORT_V2_BLUEPRINT_20261123.md) · [PROD Readiness Gap](PROD_READINESS_GAP_20261123.md) · [GW-011](../tracking/ISSUES.md).
