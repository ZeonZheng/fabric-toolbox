# Engineering docs — OUR notes

The current project state is [../STATUS.md](../STATUS.md). These documents contain **deeper** implementation evidence and must not become duplicate status dashboards.

| Document | Primary concern |
| --- | --- |
| [ARCHITECTURE_OPERATIONS.md](ARCHITECTURE_OPERATIONS.md) | Data path, local file lifecycle, collector risks, PROD decisions |
| [DEPLOYMENT_RUNBOOK.md](DEPLOYMENT_RUNBOOK.md) | Setup and operating procedures |
| [POC_STATUS.md](POC_STATUS.md) | Historic test outputs, field-level technical notes |
| [TROUBLESHOOTING.md](TROUBLESHOOTING.md) | Diagnosed symptoms and fixes |
| [UPSTREAM_CHANGES.md](UPSTREAM_CHANGES.md) | What differs from vendor source |
| [PARSER_SCHEMA_AUDIT.md](PARSER_SCHEMA_AUDIT.md) | Confirmed Raw → Parsed defects |
| [SCHEMA_FIELD_MAPPING_AUDIT.md](SCHEMA_FIELD_MAPPING_AUDIT.md) | Full observed key-to-target inventory |
| [PROD_UPLOAD_MANIFEST_DESIGN.md](PROD_UPLOAD_MANIFEST_DESIGN.md) | Prospective PROD lineage design, **not implemented** |
| [Gateway-Monitoring-POC.drawio](Gateway-Monitoring-POC.drawio) | Editable POC architecture (historic diagram; compare with status) |

Raw snapshots and exports: [../evidence/](../evidence/). Executable KQL: [../kql/](../kql/). Modified notebooks: [../notebooks/](../notebooks/).
