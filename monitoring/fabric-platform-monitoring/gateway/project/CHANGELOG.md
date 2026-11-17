# Project worklog (chronological)

Each entry is dated in **Asia/Singapore**. This is a historical log, not the latest status; use [STATUS.md](STATUS.md).

## 2026-11-17 — POC candidate validation and repository refactor
- Completed read-only KQL dry-runs: Aggregation 77 rows (3 fields 77 non-null, 3 fields 56); Streaming 113 rows (2 fields 104); Mashup 91 rows (5 fields 91). **Not deployed**.
- Captured 15 live Function bodies, 142 deployed-schema rows, enabled Update Policies and sanitized MCP Eventhouse overview. Source and deployed function bodies agreed 15/15.
- Reorganized **our** docs/evidence/KQL/notebooks under `gateway/project/`. Kept upstream code and vendor assets in place. Introduced central project overview, current status, issue register, ADR log, roadmap and AI handoff.
- Next: verify target column order/types and sampled values, then gate POC-only deployment.

## 2026-11-16 — Raw diagnostics
- Raw snapshot: 9 logTypes, 647 JSON messages, 3,693 CSV records; 106 observed logType/field combinations. Snapshot is not full ingestion audit.

## 2026-11-13 — live Gateway report upload POC
- Eight types of Gateway Report processed via Move → Temp → Eventstream → History; HTTP 201 observed; Gateway recreated files. UploadLogs Task configured for private `Config.live-poc.json`. Continuous exactly-once validation pending.
- PROD historical backfill deferred pending event-time staging plan.

## 2026-11-12 and before — functionality
- Fabric artifacts deployed; functional Gateway, Heartbeat, NodeInfo, System Counters and Jobs/Queries views checked. See [POC_STATUS](docs/POC_STATUS.md) for complete observations.

Update this log only for a significant verified milestone, test result, problem or design decision. Link evidence and specify what remained unverified.
