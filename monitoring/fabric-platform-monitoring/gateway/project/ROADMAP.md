# Phases and release gates

**Updated:** 2026-11-17 (Asia/Singapore). Dates below are evidence dates, not promised finish dates.

| Phase | Exit criterion | Status |
| --- | --- | --- |
| **P0 — Functional POC** | Core reporting path, 8 CSV report types, NodeInfo/Heartbeat observed | Completed as functionality, not reliability |
| **P1 — Data correctness** | Every required Raw field maps with correct type/value to Parsed; repeatable KQL fix with rollback; report regression | **In progress**: dry-run 3/3 passed; compatibility validation and deployment pending |
| **P2 — Collector reliability** | Proven file/record/batch reconciliation, multi-cycle tests, rotation, retries/duplicates, failure recovery and resource budget | Not completed |
| **P3 — Production architecture** | Upload Manifest specification, retention/backup, secure identity, event-time backfill cutoff, multi-node, observability/costs signed off | Planned |
| **P4 — Limited PROD pilot** | Approved change, acceptance dashboard, rollback and operator handover | Not authorized |

**PROD gate:** Parser issues closed; newly ingested records pass Raw/Parsed regression; no unexplained data loss/duplicates; safe handling of live files proven; Manifest and replay approach approved; privacy of QueryText/credentials reviewed; scope, operational owner and sign-off recorded.

Related: [ISSUES](ISSUES.md), [DECISIONS](DECISIONS.md), [architecture](docs/ARCHITECTURE_OPERATIONS.md).
