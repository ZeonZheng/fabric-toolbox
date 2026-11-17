# Decisions (lightweight ADR index)

**Updated 2026-11-17 (Asia/Singapore).** An accepted direction is not proof of implementation.

| ID | Date | Decision | Status / reason |
| --- | --- | --- | --- |
| ADR-001 | 2026-11-17 | GitHub engineering assets are the source of truth | Accepted: deployed UI changes must be reproducible from reviewed files |
| ADR-002 | 2026-11-17 | Keep vendor upstream paths unchanged; group OUR artifacts under `gateway/project/` | Implemented in this branch, preserves upstream comparison |
| ADR-003 | 2026-11-17 | Read-only Parser tests before any POC `.create-or-alter function`; retain rollback | Accepted; dry-runs done, schema/value checks pending |
| ADR-004 | 2026-11-17 | Include Upload Manifest as PROD design candidate | Accepted for evaluation only; no manifest built |
| ADR-005 | 2026-11-13 | Observe original Move uploader only in bounded POC; PROD uses staging and event-time cutoff | POC-only scope; reliability unproven |
| ADR-006 | 2026-11-17 | Keep target Parsed schema during initial mapping fix, separately evaluate full Raw field retention | Provisional; avoids unreviewed model/report breaking changes |

For technical rationale, see [Architecture](docs/ARCHITECTURE_OPERATIONS.md), [Parser audit](docs/PARSER_SCHEMA_AUDIT.md), [Manifest design](docs/PROD_UPLOAD_MANIFEST_DESIGN.md).
