# Issue & risk register

**Updated:** 2026-11-17 (Asia/Singapore). This register is the canonical backlog index; detailed technical traces live in docs/evidence. Do not close an issue without a dated reproducible result.

| ID | Priority | Status | First noted | Last reviewed | Responsible | Issue / risk | Next action | Evidence |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| **GW-001** | P1 | **Validating** | 2026-11-17 | 2026-11-17 | TBD | Raw JSON keys mismatch 3 current Parser schemas; parsed durations/memory null | Verify candidate schema/types/values; controlled POC-only Function changes | [Audit](docs/SCHEMA_FIELD_MAPPING_AUDIT.md) · [Dry-run](evidence/2026-11-17/DRYRUN_RESULTS.md) |
| **GW-002** | P1 | Open | 2026-11-12 | 2026-11-17 | TBD | `Move-Item` of active files; partial batch retry / duplicates / lost records | File/record/batch reconciliation + fault injection | [Lifecycle](docs/ARCHITECTURE_OPERATIONS.md) |
| **GW-003** | P1 | Open | 2026-11-17 | 2026-11-17 | TBD | Raw retention 1d insufficient for later validation/replay | Review retention/archive and costs before PROD | [MCP inventory](evidence/2026-11-17/EventhouseOverview.MCP.sanitized.md) |
| **GW-004** | P2 | Planned | 2026-11-17 | 2026-11-17 | TBD | Upload Manifest with source file identity, hash, rows, batch states and audit trail | Design durable manifest and exactly-once/idempotency semantics | [Manifest](docs/PROD_UPLOAD_MANIFEST_DESIGN.md) |
| **GW-005** | P2 | Decision needed | 2026-11-17 | 2026-11-17 | TBD | Parsed Tables omit some original Raw CSV fields | Decide dashboard-oriented vs full-fidelity long-term retention | [Mapping](docs/SCHEMA_FIELD_MAPPING_AUDIT.md) |
| **GW-006** | P2 | Open | 2026-11-13 | 2026-11-17 | TBD | Multi-node isolation, initial PROD backfill and event-time cutoff | Staging/cutoff and rollout design | [Operations](docs/ARCHITECTURE_OPERATIONS.md) |
| **GW-007** | P2 | Open | 2026-11-17 | 2026-11-17 | TBD | QueryText, DataSource URLs, permissions and secrets privacy | Security & least-privilege review | [Operations](docs/ARCHITECTURE_OPERATIONS.md) |
| **GW-008** | P3 | Open | 2026-11-17 | 2026-11-17 | TBD | CopyExecutionReport source never observed; two Parser docstrings inaccurate | Source-triggered test and cosmetic clean-up later | [MCP inventory](evidence/2026-11-17/EventhouseOverview.MCP.sanitized.md) |

**Status definitions:** Open = not started/resolved; Validating = candidate or investigation under test; Planned = design decision approved for exploration, not implemented; Closed = evidence and verification documented. Project uses internal issue IDs; GitHub Issues may be introduced later with cross-references, but no duplicated issue status systems.

**Accountability:** `Responsible = TBD` is intentional; no business owner or engineering assignee has been authorized. Set actual owner and target date when agreed, and update `Last reviewed` when evidence is rechecked. Do not invent completion timestamps.
