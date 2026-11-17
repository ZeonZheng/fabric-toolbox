# Evidence register

Evidence is organized by **capture date**, not by latest status.

- [2026-11-17 baselines](2026-11-17/README.md): source `DatabaseSchema.kql`, live 15 Function Bodies, actual table schema, policy snapshot, original Raw field inventory and sanitized MCP overview.
- [2026-11-17 Dry-run results](2026-11-17/DRYRUN_RESULTS.md): transcription from 3 user-executed KQL screenshots, NOT original CSV output.
- Earlier observations: [POC_STATUS](../docs/POC_STATUS.md).

Snapshots are documentation, not a recoverable Eventhouse database backup. Do not store raw Gateway log payloads, QueryText, service IDs or credentials.
