# AI / engineer handoff

**Read:** [README](README.md) → [STATUS](STATUS.md) → [ISSUES](ISSUES.md) → [Docs index](docs/INDEX.md). Current as of **2026-11-17 (Asia/Singapore)**.

**Ownership & scope:** `monitoring/fabric-platform-monitoring/gateway/project/` contains OUR artifacts on branch `docs/gateway-monitoring-poc-20261112`. Upstream PowerShell, Scheduler, PBI Report stay in `gateway/`; upstream Eventhouse schema remains in `monitoring/fabric-platform-monitoring/src/Gateway/`. One modified upstream Heartbeat script is documented in [UPSTREAM_CHANGES](docs/UPSTREAM_CHANGES.md). Never silently modify `main` or non-Gateway modules.

**Facts:** Single-node Gateway POC running. Eight report CSV types observed. 15 deployed functions fully backed up; 15/15 match GitHub source. Three parser candidates successfully executed READ-ONLY and read formerly missing values; schema/value checks pending. No Function DDL or PROD deployment. Raw retention nominal 1 day; 40 History + 2 Source at last VM observation; no source/Raw/Parsed full reconciliation or exactly-once guarantee.

**File index:** [technical docs](docs/INDEX.md) · [evidence snapshots](evidence/README.md) · [candidate/dryrun/rollback](kql/poc-parser-fix-20261117/README.md) · [worklog](CHANGELOG.md) · [decisions](DECISIONS.md).

**Safety & workflow:** Do not commit Config secrets, Eventstream SAS, Gateway QueryText, full raw logs, production identifiers or access data. Before changing Eventhouse, compare live Functions to GitHub, validate Kusto Schema and raw values, consider transactional Update Policy ingestion impact, and have rollback ready. **Never run candidate DDL or Jumpstart overwrite automatically.** After each investigation update only canonical [STATUS](STATUS.md), issue transition in [ISSUES](ISSUES.md), and dated [CHANGELOG](CHANGELOG.md); link immutable evidence. If unsure about a source or test, label it unverified.

**Next:** run [three prebuilt schema checks](kql/poc-parser-fix-20261117/schema/README.md), require every `IsMatch=true`, and compare sampled numeric/unit values before POC-only changes. See [Dry-run evidence](evidence/2026-11-17/DRYRUN_RESULTS.md).
