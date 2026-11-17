# Parser candidate read-only dry-runs

**Captured:** 2026-11-17, approximately 07:40 UTC / 15:40 Asia/Singapore. **Source:** User-provided KQL Queryset screenshots; values transcribed, not copied from a downloadable CSV. **Environment:** POC only. **No live DDL executed**.

| Parser candidate | Raw rows in dry-run | Non-null test metrics |
| --- | ---: | --- |
| QueryExecutionAggregationReport | 77 | Average/Max/Min QueryExecutionDuration: **77 each**; Average/Max/Min DataProcessingDuration: **56 each** |
| QueryStreamingReport | 113 | SpoolingDiskReadingDuration: **104**; StreamingDuration: **104** |
| MashupEvaluationReport | 91 | AverageCommit, AverageWorkingSet, MaxCommit, MaxWorkingSet, TotalProcessorTime: **91 each** |

**Proved:** query syntax executes, Raw JSON fields recover expected non-null data. **Not proved:** target column order and types, accurate numeric values, proper unit conversion (especially TotalProcessorTime milliseconds to Kusto timespan), DataSource transformation, live Update Policy compatibility, old null backfill, file-level delivery assurance.

**Next evidence:** 3 candidate `getschema` outputs, target schema comparison, a few raw/target numeric comparisons, then if approved POC-only controlled DDL + new-ingestion verification.

[Candidate KQLs](../../kql/poc-parser-fix-20261117/README.md) · [Issue GW-001](../../ISSUES.md).
