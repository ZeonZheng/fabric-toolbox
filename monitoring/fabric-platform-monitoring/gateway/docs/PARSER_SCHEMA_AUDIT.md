# Gateway Monitoring — Parser/schema consistency audit (2026-11-17)

> **Status: confirmed POC data-quality defect; remediation NOT yet applied.** POC single node only. GitHub branch `docs/gateway-monitoring-poc-20261112` is the source of truth. Evidence backups: [baselines/2026-11-17/](baselines/2026-11-17/README.md).

## Executive summary

- Uploader/Eventstream/Raw path is working: 2026-11-16 02:59 UTC query showed **9 `logType` values, 647 JSON messages, 3,693 CSV records**. Eight are CSV Report types; `GatewayNodeInfo` is generated independently. These counts are *point-in-time*, not cumulative or completeness proof.
- Field inventory from Eventhouse Raw: **106 type/field entries** across 9 types (user's `export (3).csv`), every field within a given type appeared in every sampled record; non-empty ratios differ legitimately by field.
- Checked Parsed Tables at **2026-11-17 ~06:24 UTC**:

| Parsed table | Rows | Non-null checked metric | Metric |
| --- | ---: | ---: | --- |
| QueryExecutionReport | 4,595 | 4,595 | `QueryExecutionDuration_ms_` |
| QueryExecutionAggregationReport | 2,935 | **0** | `AverageQueryExecutionDuration_ms_` |
| QueryStreamingReport | 2,346 | **0** | `StreamingDuration_ms_` |
| MashupEvaluationReport | 1,892 | **0** | `AverageCommit byte` |

- Three corresponding Raw field checks at **2026-11-17 ~06:31 UTC**, over the Raw retention window:

| `logType` | Raw field | Raw rows | Non-empty | Example value |
| --- | --- | ---: | ---: | --- |
| QueryExecutionAggregationReport | `AverageQueryExecutionDuration(ms)` | 73 | 73 | `266536.00` |
| QueryStreamingReport | `StreamingDuration(ms)` | 90 | 81 | `38` |
| MashupEvaluationReport | `AverageCommit(byte)` | 77 | 77 | `261075800.00` |

- User-exported deployed bodies for the **three Parser Functions** match the corresponding functions in the fork's `DatabaseSchema.kql` source snapshot. Source/deployment name mismatches below explain the null behavior; the remaining live functions have **not** been exported/verified.
- Eventhouse `.show table * policy update` (14 entries) showed enabled update policies for 10 parsed targets with source `GatewayReports-Raw`, transactional true. Filtered `.show ingestion failures` returned zero results for reported 24h window; this does **not** verify payload field quality.

## Field mapping mismatch — confirmed

| Target field | Source Raw JSON key | Deployed `bag_unpack()` requests | Result |
| --- | --- | --- | --- |
| QueryExecutionAggregationReport.`AverageQueryExecutionDuration_ms_` | `AverageQueryExecutionDuration(ms)` | `AverageQueryExecutionDuration_ms_` | all null in checked table |
| QueryStreamingReport.`StreamingDuration_ms_` | `StreamingDuration(ms)` | `StreamingDuration_ms_` | all null in checked table |
| MashupEvaluationReport.`AverageCommit byte` | `AverageCommit(byte)` | `AverageCommit byte` | all null in checked table |

Additional gaps **need verification, not yet signed off**:
- QueryExecutionAggregation: other `Max/Min/Average...Duration(ms)` to `_ms_` mappings.
- QueryStreaming: `SpoolingDiskReadingDuration(ms)`; Raw also has `StreamingDelayDuration(ms)`, `StreamingPacketsDuration(ms)`, `StreamingTrackingId`, which the current target schema does not preserve.
- MashupEvaluation: Raw `DataSource`, `TotalProcessorTime(ms)`, `AverageWorkingSet(byte)`, `MaxCommit(byte)`, `MaxWorkingSet(byte)`; target has different field names / `timespan` type, and `DataSourceKinds/Paths` target fields require design clarification.
- QueryExecutionReport: Raw has `DataReadingDuration(ms)` and `DataSerializationDuration(ms)` not present in current target schema.
- QueryStartReport: uploader can replace unusually large `QueryText` with sentinel `QueryText too large!`; separately verify parser's Base64 decoding / expansion semantics.
- Confirm normalized expected schema vs "preserve every CSV field": these are **different acceptance criteria**.

## Evidence source, VM state, and retention

- **2026-11-17 VM snapshot:** report directory contains **42 files (40 History merged + 2 Source)**, 8 Report types. History is a merged archive, not original per-file provenance or proof of ingestion.
- `GatewayReports-Raw` retention in repository defaults to **1 day**; parsed tables in repository generally 3650 days. Query in current environment before Raw expiry; changing Functions does not automatically rewrite old parsed rows.
- `DatabaseSchema.source.kql` is a snapshot from GitHub **source**, **not** an actual entire deployed Eventhouse schema dump. User-provided exported live Functions cover 3 parsers only. Credentials, IDs and query payloads deliberately excluded.

## Safe rollout sequence — no live writes performed yet

1. **Read-only inventory:** export all deployed `.show functions` Bodies, table schemas, Update Policy, and retention policy (review sensitive metadata). Complete 8-type/106-field Raw-to-Target mapping matrix. Record any parser version drift vs source.
2. **Design patch in GitHub first:** keep existing target schema for targeted bug fix. Use explicit **raw keys** `['... (ms)']` or `record["... (ms)"]`, cast values with `toreal()/tolong()`, alias to target names. For Mashup unit conversion, preserve target semantic type (e.g. ms numeric vs timespan), don't blindly cast to `timespan`. Capture expected/actual schema order.
3. **Non-mutating validation:** test on recent Raw rows with a standalone `let` / query function, compare before-and-after non-null counts and sample values for **all relevant metrics**, and verify no added nulls.
4. **POC apply only:** deploy vetted `.create-or-alter function` on the POC Eventhouse; verify Update Policy still enabled and new incoming events populate target values. Do not modify upstream source or PROD silently.
5. **Historical remediation separately:** assess Raw retention availability, backfill/reingest path, deduplication, and report/model compatibility before modifying old rows.
6. **Regression:** check all 8 Report types, row count, field accuracy, parsing failures, performance overhead, daily continuity, retry/dedup behavior; only then consider PROD promotion.

### Read-only KQL probe (Raw)

```kusto
['GatewayReports-Raw']
| where logType in ("QueryExecutionAggregationReport","QueryStreamingReport","MashupEvaluationReport")
| mv-expand record=log
| extend RawValue = case(
    logType == "QueryExecutionAggregationReport", todouble(record["AverageQueryExecutionDuration(ms)"]),
    logType == "QueryStreamingReport", todouble(record["StreamingDuration(ms)"]),
    logType == "MashupEvaluationReport", todouble(record["AverageCommit(byte)"]),
    real(null))
| summarize Rows=count(), NonNull=countif(isnotnull(RawValue)), Example=take_anyif(RawValue,isnotnull(RawValue)) by logType
```

## Rollback and deployment guardrails

- Save exact previous live function body before any alteration; `create-or-alter` rollback must use a valid management command wrapper around that body.
- Live update-policy functions may be referenced during ingestion; test in POC with a controlled source period. Existing rows will not change after a function edit.
- Jumpstart reinstall/update can overwrite custom Fabric objects. Keep patch + validation tests version-controlled under `gateway/` before PROD.
- Never commit original `GatewayReports-Raw` events or query texts to GitHub; retain any necessary raw payload evidence in an access-controlled location.

**Acceptance criterion:** verified Raw values are present at the correctly named target fields for newly ingested events, within a controlled window; broader source-to-target mapping and exactly-once completeness remain pending.
