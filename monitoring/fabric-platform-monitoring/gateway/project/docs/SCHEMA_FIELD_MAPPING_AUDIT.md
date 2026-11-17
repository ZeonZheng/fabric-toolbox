# Gateway Raw CSV → Eventhouse target schema comparison (2026-11-17)

**Status:** inspected all 15 live Function bodies and 142 database schema rows. **No live Eventhouse modification.** All 15 deployed Function bodies are textually equivalent after whitespace normalization to the GitHub source schema on branch `docs/gateway-monitoring-poc-20261112`. Both source and deployment therefore share the confirmed parser bugs.

Source files: [deployed functions](baselines/2026-11-17/DeployedFunctions.full.export.csv), [deployed schema](baselines/2026-11-17/DeployedTableSchema.sanitized.csv), [Raw keys/presence](baselines/2026-11-17/RawFieldInventory.export.csv), [MCP overview](baselines/2026-11-17/EventhouseOverview.MCP.sanitized.json).

## Coverage overview

| `logType` | Raw keys observed | Parsed table columns | Finding |
| --- | ---: | ---: | --- |
| ConcurrentOperationAggregationReport | 7 | 7 | Names identical; numeric values not independently compared |
| GatewayNodeInfo | 16 | 17 | Target adds envelope `logDate` |
| MashupEvaluationReport | 17 | 17 | **5 transform/rename differences**; `DataSourceKinds/Paths` have no direct raw key; `DataSource` and `RequestId` are not retained |
| OpenConnectionReport | 9 | 9 | Raw `OpenConnectionDuration(ms)` → target `OpenConnectionDuration_ms_`. Existing parser references the **correct raw key** and outputs in target order; no proven bug for that field |
| QueryExecutionAggregationReport | 13 | 13 | **6 duration fields** have `(ms)` Raw names but parser requests `_ms_` names instead |
| QueryExecutionReport | 17 | 15 | Six normalized duration/size target columns; parser reads their correct Raw keys. Raw-only `DataReadingDuration(ms)` and `DataSerializationDuration(ms)` are not preserved in parsed table |
| QueryStartReport | 8 | 18 | Target adds/derives 10 fields from query text and evaluation context; expansion affects cardinality |
| QueryStreamingReport | 12 | 9 | **2 duration fields** request incorrect Raw names; raw-only `StreamingDelayDuration(ms)`, `StreamingPacketsDuration(ms)`, `StreamingTrackingId` not preserved in target |
| SystemCounterAggregationReport | 7 | 7 | Names identical; numeric values not independently compared |
| CopyExecutionReport | 0 in observed Raw | 7 | Target & policy exist but no source samples, so cannot verify |

## Confirmed loss-of-value problems (not just display naming)

| Type | Raw field | Target field | State |
| --- | --- | --- | --- |
| QueryExecutionAggregationReport | AverageQueryExecutionDuration(ms) | AverageQueryExecutionDuration_ms_ | Raw 73/73 nonempty; parsed 0/2935 nonnull |
| QueryStreamingReport | StreamingDuration(ms) | StreamingDuration_ms_ | Raw 81/90 nonempty; parsed 0/2346 nonnull |
| MashupEvaluationReport | AverageCommit(byte) | AverageCommit byte | Raw 77/77 nonempty; parsed 0/1892 nonnull |

**Potentially same defect from source code, pending per-field live value checks:** QueryExecutionAggregation's remaining 5 duration fields; QueryStreaming's `SpoolingDiskReadingDuration(ms)`; MashupEvaluation's `AverageWorkingSet(byte)`, `MaxCommit(byte)`, `MaxWorkingSet(byte)`, `TotalProcessorTime(ms)`.

**Key nuance:** Parsed target column names need not match Raw keys if the parser explicitly unpacks the correct Raw keys and **projects them in the exact target column order/types**. Example: `QueryExecutionReport` is correctly unpacking `['QueryExecutionDuration(ms)']` and the tested target field is 4595/4595 nonnull. Do not mechanically rename all functions or modify target schema based only on names.

## Data completeness and remaining decisions

1. Decide whether PROD acceptance means **dashboard-required normalized schema** or **every original Gateway CSV field retained for arbitrary later analysis**. Current target schemas do not preserve all Raw fields. Raw retention is set to 1 day; a Raw archive/retention extension or added target fields is needed for long-term raw-field preservation.
2. For MashupEvaluation, `DataSourceKinds`/`DataSourcePaths` are not keys in the observed CSV. Do **not** invent them from `DataSource` without examining actual payload formats and deciding the intended semantics. Proposed POC patch intentionally leaves them as empty string if absent.
3. `TotalProcessorTime(ms)` needs conversion into `TotalProcessorTime:timespan` (numeric milliseconds × `1ms`), including null handling and unit test.
4. Do not infer which Raw `QueryStartReport` rows should equal parsed cardinality without inspecting multi-value evaluation context behavior.
5. Data in Raw can expire independently of 3650-day parsed retention. MCP snapshot reports 1-day Raw retention, 7-day Heartbeat; it does not verify whole historical completeness. Physical deletion may be asynchronous.
6. The MCP overview documents 12 physical tables, 15 functions, 10 enabled transactional policies, and 2 healthy materialized views at capture time. Treat it as a point-in-time inventory, not a replacement for actual Function/Schema exports.
7. None of the candidate patches have been executed or accepted by the POC Kusto engine yet.

Next steps: [run the three read-only preview queries first](../kql/poc-parser-fix-20261117/README.md), verify schema/order and numeric values, then—after review—apply POC-only Function alterations and check **new** ingested records. Keep rollback scripts ready.
