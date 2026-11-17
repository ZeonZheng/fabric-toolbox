# Gateway Monitoring — 2026-11-17 POC baseline

This directory is an **evidence snapshot, not a production-ready patch**. Scope: single Gateway cluster/node POC; all data collected from the user's KQL Queryset and their GitHub fork on 2026-11-17.

| Artifact | Provenance | Coverage / limitation |
| --- | --- | --- |
| `DatabaseSchema.source.kql` | Fork source `src/Gateway/Gateway Monitoring.KQLDatabase/DatabaseSchema.kql` at blob SHA `10e94c1ec71102283f6f93c9ff05da705db77ab8` | **Source baseline only**: this is NOT a full live Eventhouse schema export. |
| `DeployedParserFunctions.export.csv` | Eventhouse `.show functions | where Name in (...) | project Name, Body` user export (5) | **Actual deployed bodies of 3 functions only**: `parse_QueryExecutionAggregationReport`, `parse_QueryStreamingReport`, `parse_MashupEvaluationReport`. Other live functions have not been backed up. |
| `DeployedUpdatePolicies.sanitized.json` | User Eventhouse `.show table * policy update` export (4) | 14 table entries; environment-specific database GUIDs removed from EntityName. No guarantee of identical full original export metadata. |
| `DeployedFunctions.full.export.csv` | Actual `.show functions | project Name, Parameters, Body, Folder` user export (6) | **All 15 live Function Bodies**, exact export; verified 15/15 Bodies match GitHub source by whitespace-normalized comparison. |\n| `DeployedTableSchema.sanitized.csv` | `.show database schema` user export (7) | **142 schema rows**, DatabaseName stripped to avoid saving environment identifier. Includes tables, column names/types/order. |\n| `EventhouseOverview.MCP.sanitized.json` / `.md` | User-supplied MCP overview | Snapshot of 12 tables, 10 update policies, 2 materialized views, counts/retention. Database GUID / cluster / policy owner IDs removed; not a byte-for-byte runtime export. |\n| `RawFieldInventory.export.csv` | User query export (3) from `GatewayReports-Raw` | 106 (logType, field) presence/non-empty stats sampled while Raw retained. **Not actual raw events, and not a complete schema history**. |

**Deployment source of truth:** GitHub fork branch `docs/gateway-monitoring-poc-20261112`. No changes have been applied to the running Eventhouse or its update policies by this backup. 

**Security:** no gateway log payloads, query texts, secrets, connection strings, gateway/workspace IDs or runtime Config JSON are stored. If generating a complete live export later, review content before committing; use restricted storage for sensitive values.

**Updated 2026-11-17:** All 15 deployed Function bodies and 142 schema rows are now captured, as well as a sanitised MCP overview. The exact 3-function export remains as the earlier narrower evidence. This is not a full Eventhouse service backup: security policies, ingestion connections, identities and original Gateway log payloads are not included. See [field mapping audit](../../SCHEMA_FIELD_MAPPING_AUDIT.md).
