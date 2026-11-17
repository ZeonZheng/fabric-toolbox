# PROD design backlog — upload manifest and end-to-end traceability

> **Decision recorded 2026-11-17:** investigate adding an Upload Manifest before PROD. **Not implemented, not approved for deployment**. Current POC still uses original `Move-Item → upload → Merge-ReportFiles` implementation.

## Why

Current EventHub envelope includes `logType`, `log` (CSV rows) and `logDate`, but lacks source-file ID, name, hash, row offsets and batch ID. `HTTP 201` means receiver accepted the request; it does not guarantee downstream Eventhouse parsing, every row ingested once, or long-term auditability. Merged `History/*_f.log` loses original file boundaries.

## Proposed file-level manifest (design, not schema)

| Field | Purpose |
| --- | --- |
| `ManifestId` / `NodeId` | Stable idempotency identity and origin node; don't rely solely on filename |
| `SourceFileName`, `SourceFileIdentity` | Origin and rotation/truncation detection |
| `ReportType`, `OriginalFileSizeBytes` | Coverage, inventory, diagnostic triage |
| `ContentSHA256`, `RecordCount` | Integrity checks for a **stable snapshot** of the file |
| `FirstEventUtc`, `LastEventUtc` | Actual event time range (not filename creation date) |
| `ExpectedBatchCount`, `AcceptedBatchCount` | Track partial sending; batches need persistent IDs |
| `State` | `Discovered → Snapshotted → Sending → Accepted → RawVerified → ParsedVerified`, with `Failed/RetryPending` paths |
| `Attempts`, `LastError`, `UpdatedUtc` | Diagnostics and audit |
| `ParserVersion` / `SchemaVersion` | Distinguish upload completeness from parser compatibility |

## Caveats and acceptance

- Hash/record counts on live-growing files are unstable. Create a **stable, read-only copy/snapshot** or define byte offsets and frozen file identity before computing full-file checksums.
- Hash alone won't prove all records arrived in Eventhouse. Batch-level identity and downstream Raw/Parsed reconciliation are required.
- Retry after uncertain HTTP/ingestion status may cause duplicates. Define at-least-once semantics and explicit idempotency/deduplication strategy; avoid claiming exactly-once.
- The design should also support multi-node isolation, backfill cutoffs, disk/backlog observability, privacy (QueryText and DataSource), and local retention/secure ACLs.
- Manifest storage, Eventhouse schema, uploader implementation, rollout gating and associated costs remain open decisions. No ingestion code change made in this POC milestone.

**Prerequisite:** close [Parser/schema audit](PARSER_SCHEMA_AUDIT.md) and continuous collection verification before approving PROD.
