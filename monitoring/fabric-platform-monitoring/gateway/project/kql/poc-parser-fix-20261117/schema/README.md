# POC Parser — schema compatibility checks

**All queries are READ-ONLY; do not run candidate DDL until these pass.**

For each target, open the schema-check script (links below) in the POC KQL Queryset. It defines a local `Candidate()`, runs `getschema` for Candidate and existing target table, and joins schemas by `ColumnOrdinal`. The output reports field name and type matches; **all `IsMatch` must be true**. A missing column appears with an empty name on one side.

- [Aggregation schema check](1-schema-check-parse_QueryExecutionAggregationReport.kql)
- [Streaming schema check](2-schema-check-parse_QueryStreamingReport.kql)
- [Mashup schema check](3-schema-check-parse_MashupEvaluationReport.kql)

Microsoft Kusto requires update-policy query output schema to match target table name, type and order. Source: https://learn.microsoft.com/en-us/kusto/management/alter-table-update-policy-command?view=microsoft-fabric

**If any output row differs, stop.** Do not alter the function or policy. Report results in [evidence](../../../evidence/2026-11-17/DRYRUN_RESULTS.md).

Beyond schema, still check sample numeric values/units, especially `TotalProcessorTime(ms)` → Kusto `timespan`, and DataSource behavior. The successful non-null dry-runs do not prove those.
