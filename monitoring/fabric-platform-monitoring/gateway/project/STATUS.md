# Current status / next action

**Updated:** 2026-11-17, Asia/Singapore · **Current phase:** POC data correctness · **Deployment:** No Parser fixes applied · **Production:** Not approved.

| Workstream | Status | Verified evidence / gap |
| --- | --- | --- |
| Gateway → Fabric → report | Functional POC passed | End-to-end visibility of Gateway information, queries and counters |
| Original Move uploader | Running / monitoring | 42 Report files at last VM check (40 History, 2 Source); exactly-once and loss prevention unproven |
| Raw coverage | Partially checked | 8 report types + GatewayNodeInfo in Raw; full file-level reconciliation pending |
| Raw → Parsed functions | **Confirmed defect / candidate dry-run passed** | Raw nonempty values where 3 key Parsed metrics were entirely null |
| Eventhouse deployed baseline | Backed up | 15 Function bodies, 142 schema rows, policy snapshot and sanitized MCP overview |
| Candidate schema/values | **NEXT: not yet verified** | KQL `getschema`, target order/types, numeric/value equality, DataSource and timespan |
| POC Function DDL | Not performed | Only GitHub candidate and rollback KQL files exist |
| PROD operational design | Blocked | Manifest, file identity, retries/dedup, retention, multi-node, security and runbook acceptance |

## One next action

**Run read-only schema/type checks of the 3 Candidate KQLs.** For each file in [kql/dryrun](kql/poc-parser-fix-20261117/dryrun/), replace only the final `| summarize` with `| getschema`; compare columns, order and data types against [deployed table schema](evidence/2026-11-17/DeployedTableSchema.sanitized.csv). Also verify a few exact source/target numeric values, DataSource shape, and Mashup milliseconds → timespan. Save evidence before any DDL. Do not execute the candidate files yet.

**Last completed checkpoint:** user-run read-only dry-runs on 2026-11-17 ~15:40 Asia/Singapore:
- QueryExecutionAggregation: 77 rows; 3 Query Duration fields 77 non-null each; 3 Data Processing Duration fields 56 each.
- QueryStreaming: 113 rows; both duration fields 104 non-null each.
- MashupEvaluation: 91 rows; five tested metrics 91 non-null each.

Do not compare this 1-day Raw retention window directly with 3650-day Parsed full-table counts. [Detailed test record](evidence/2026-11-17/DRYRUN_RESULTS.md) · [GW-001](ISSUES.md).
