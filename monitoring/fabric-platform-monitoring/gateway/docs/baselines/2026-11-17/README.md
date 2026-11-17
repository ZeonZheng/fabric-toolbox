# Gateway Monitoring — 2026-11-17 POC baseline

This directory is an **evidence snapshot, not a production-ready patch**. Scope: single Gateway cluster/node POC; all data collected from the user's KQL Queryset and their GitHub fork on 2026-11-17.

| Artifact | Provenance | Coverage / limitation |
| --- | --- | --- |
| `DatabaseSchema.source.kql` | Fork source `src/Gateway/Gateway Monitoring.KQLDatabase/DatabaseSchema.kql` at blob SHA `10e94c1ec71102283f6f93c9ff05da705db77ab8` | **Source baseline only**: this is NOT a full live Eventhouse schema export. |
| `DeployedParserFunctions.export.csv` | Eventhouse `.show functions | where Name in (...) | project Name, Body` user export (5) | **Actual deployed bodies of 3 functions only**: `parse_QueryExecutionAggregationReport`, `parse_QueryStreamingReport`, `parse_MashupEvaluationReport`. Other live functions have not been backed up. |
| `DeployedUpdatePolicies.sanitized.json` | User Eventhouse `.show table * policy update` export (4) | 14 table entries; environment-specific database GUIDs removed from EntityName. No guarantee of identical full original export metadata. |
| `RawFieldInventory.export.csv` | User query export (3) from `GatewayReports-Raw` | 106 (logType, field) presence/non-empty stats sampled while Raw retained. **Not actual raw events, and not a complete schema history**. |

**Deployment source of truth:** GitHub fork branch `docs/gateway-monitoring-poc-20261112`. No changes have been applied to the running Eventhouse or its update policies by this backup. 

**Security:** no gateway log payloads, query texts, secrets, connection strings, gateway/workspace IDs or runtime Config JSON are stored. If generating a complete live export later, review content before committing; use restricted storage for sensitive values.

**Next missing baseline:** export all remaining deployed function bodies (and actual table schemas/retention policies) from Eventhouse, then compare with GitHub source. This snapshot captures the three live functions already supplied by the user, not more.
