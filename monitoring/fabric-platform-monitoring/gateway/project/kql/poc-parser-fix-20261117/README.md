# POC Parser remediation — candidates only (2026-11-17)

**No patch in this directory has been applied to Eventhouse.** This is a candidate set for the user's single-node Gateway Monitoring POC. **Not for PROD and not yet syntax- or ingestion-validated.**

## Exact scope

- **No Uploader / Task / Eventstream / Update Policy / Table changes.**
- **Only three existing stored Functions** are candidates for later `.create-or-alter function`.
- Existing target schema, **column order and types retained**; no new target columns. Function code reads the original CSV JSON key names explicitly, converts to expected KQL types, and projects values in target order.
- `MashupEvaluationReport.DataSourceKinds` and `DataSourcePaths` are deliberately not synthesized from `DataSource` because their semantics are not established. This is not yet a full-field preservation solution.

## Files

| Folder | Purpose |
| --- | --- |
| `schema/` | **READ-ONLY** KQL `getschema` comparison of Candidate vs target type/order/name; run after successful dry-run |\n| `dryrun/` | **READ-ONLY** KQL using `let Candidate = () {...}` and `summarize`; safe first execution |
| `candidates/` | **MUTATING** `.create-or-alter function` commands, one per file; **do not run yet** |
| `rollback/` | Restore exact 3 exported deployed Function bodies through `.create-or-alter function` wrappers. Use only after verifying current live state |

Full live Function baseline: [DeployedFunctions.full.export.csv](../../evidence/2026-11-17/DeployedFunctions.full.export.csv).
Target schema baseline: [DeployedTableSchema.sanitized.csv](../../evidence/2026-11-17/DeployedTableSchema.sanitized.csv).
Mapping audit: [SCHEMA_FIELD_MAPPING_AUDIT.md](../../docs/SCHEMA_FIELD_MAPPING_AUDIT.md).

## Recommended execution procedure

1. **Make a non-sensitive local snapshot** of the three live Function bodies (`.show functions`) and note the time. Ensure the Queryset is connected to the **POC** KQL database, not PROD.
2. Run the three `dryrun/` files in KQL Queryset **one by one**. The query result should show `Rows > 0` and relevant `Present_...` counts > 0 for metrics whose Raw values are present. `StreamingDuration` can legitimately be empty for some raw rows.
3. **Check schema:** Execute the [prebuilt `schema/` queries](schema/README.md). They compare Candidate and target `ColumnName`, `ColumnType`, and `ColumnOrdinal`; if any `IsMatch=false`, **STOP**.
4. Validate actual values and units: compare the source `record["<CSV key>"]` with the candidate output for a small set of rows using `RequestId`/`QueryTrackingId` or reliable event timestamp; validate DataSource structure, numeric precision and `TotalProcessorTime` milliseconds-to-timespan.
5. Only after these pass, apply **one** `candidates/` command at a time in POC. Watch ingestion/errors immediately: `.show ingestion failures`, Eventstream destination metrics, Raw count and parsed count. Roll back with matching `rollback/` command if problems appear.
6. Validate **newly ingested** events after the change; existing parsed null values are not recomputed automatically. Plan any history backfill separately with duplication controls.
7. Update project's test results and then decide whether to patch `DatabaseSchema.kql` in the fork and eventually PROD. Do not re-run Jumpstart `update_existing=True` without reviewing overwrites.

## Known risks and incomplete acceptance

- Existing update policies are **transactional**: a bad function change may cause the source ingestion to fail. The update policy may not validate all references when a function body changes. Microsoft docs: https://learn.microsoft.com/en-us/kusto/management/update-policy
- Kusto's project/expressions and output types must be verified **on the actual POC engine**. No remote KQL code execution has been done in this commit.
- The original DataSource normalization expression is preserved as written in the deployed functions, but this replacement changes its input expression from unpacked `DataSource` to `record.DataSource`: compare sample values in dry-run.
- Raw may contain more fields than the Parsed Table's existing columns. This patch addresses selected broken mappings only—not full-field retention or uploader delivery completeness.
- Applying changes only through Eventhouse UI will create Git/live drift. Once tested, carry approved source changes in Git and document deployment/rollback commit SHA.


## 2026-11-17 first dry-run outcomes (user-executed)

- QueryExecutionAggregation: 77 rows; 3 query-duration outputs 77 nonnull each, 3 processing-duration outputs 56 nonnull each.
- QueryStreaming: 113 rows; both duration outputs 104 nonnull each.
- MashupEvaluation: 91 rows; five selected metrics 91 nonnull each.
- **These are read-only function candidate results only; no Eventhouse DDL has been applied.** Next: check candidate `| getschema`, target column order/types, DataSource preservation and TotalProcessorTime units before applying anything.

[Current project status](../../STATUS.md) · [dated evidence](../../evidence/2026-11-17/DRYRUN_RESULTS.md).
