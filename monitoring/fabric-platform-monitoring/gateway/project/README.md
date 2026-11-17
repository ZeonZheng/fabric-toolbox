# Gateway Monitoring — POC & Production Readiness

**As of:** 2026-11-17 (Asia/Singapore) · **Phase:** Parser correctness validation · **Environment:** single Gateway node POC · **PROD:** Not approved.

The Fabric VM → Eventstream → Eventhouse → Semantic Model → Power BI chain is working in POC. Continuous collection is under observation, and full source-to-Eventhouse record reconciliation is **not completed**. A confirmed Raw → Parsed schema mismatch caused three sample performance metrics to remain null in target tables. Three **read-only KQL candidate fixes executed successfully**; their schema/type/value compatibility must be checked before live Function DDL.

## Project cockpit

| Purpose | Single authoritative file |
| --- | --- |
| **Current state and immediate next action** | [STATUS.md](STATUS.md) |
| Roadmap, milestones, release gates | [ROADMAP.md](ROADMAP.md) |
| Issues, risks, priorities | [ISSUES.md](ISSUES.md) |
| Accepted / provisional decisions | [DECISIONS.md](DECISIONS.md) |
| Timestamped change history | [CHANGELOG.md](CHANGELOG.md) |
| AI / engineer context and safety guardrails | [AI_CONTEXT.md](AI_CONTEXT.md) |
| Detailed implementation notes | [docs/INDEX.md](docs/INDEX.md) |
| Captured KQL outputs and baselines | [evidence/README.md](evidence/README.md) |
| Dry-run / candidate / rollback KQL | [kql/poc-parser-fix-20261117/README.md](kql/poc-parser-fix-20261117/README.md) |
| Adapted installation notebooks | [notebooks/](notebooks/) |

## Boundary

**Vendor assets remain outside `project/`:** [PowerShellScript](../PowerShellScript/), [TaskSchedulers](../TaskSchedulers/), [PBI Report](../PBI%20Report/), [Fabric deployable definitions](../../src/Gateway/). One deliberately modified upstream script is `Run-GatewayHeartbeat.ps1`; see [UPSTREAM_CHANGES](docs/UPSTREAM_CHANGES.md).

GitHub is the source of truth for our engineering decisions and source patches. Current POC runtime Config, secret material, Gateway log content and QueryText are **never** committed. Jumpstart reinstall/update can overwrite Eventhouse customizations.

**Current evidence:** Aggregation 77 rows (6 non-null metrics: 77/77/77 and 56/56/56); Streaming 113 rows (2 metrics: 104 each); Mashup 91 rows (5 metrics: 91 each). These are dry-run successes **only**, not evidence of deployed repair.

[Return to Gateway entry](../README.md)
