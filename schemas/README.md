# Schemas — Network Compliance Audit

JSON Schema definitions for all components, shared data types, and the rules file format.

## Schema Map

```
schemas/
  shared-types.schema.json                       ← Core data types (Rule, RuleResult, DeviceResult, ScanResults, FleetSummary)
  rules-file.schema.json                         ← Rules file format stored in GitLab

  iag-compliance-evaluate.schema.json            ← compliance_evaluate.py  (IAG5)
  iag-compliance-generate-pdf.schema.json        ← compliance_generate_pdf.py  (IAG5)
  iag-compliance-slack-notify.schema.json        ← compliance_slack_notify.py  (IAG5)

  workflow-collect-configs.schema.json           ← Child: Compliance - Collect Configs
  workflow-evaluate-grade.schema.json            ← Child: Compliance - Evaluate and Grade
  workflow-generate-report.schema.json           ← Child: Compliance - Generate Report
  workflow-servicenow-incidents.schema.json      ← Child: Compliance - ServiceNow Incidents
  workflow-remediate-device.schema.json          ← Child: Compliance - Remediate Device (optional)
  workflow-compliance-audit-orchestrator.schema.json  ← Parent: Compliance Audit
```

## Data Flow

```
rules-file.schema.json
        │
        │ loaded at scan start
        ▼
orchestrator.inputSchema
        │
        ├──▶ collect-configs.inputSchema
        │           │
        │           └─▶ out: configs map, unreachable_devices
        │
        ├──▶ evaluate-grade.inputSchema  (receives configs)
        │           │
        │           ├─▶ iag-compliance-evaluate [×N devices]
        │           └─▶ out: scan_results (ScanResults), fleet_summary
        │
        ├──▶ generate-report.inputSchema  (receives scan_results)
        │           │
        │           ├─▶ iag-compliance-generate-pdf
        │           └─▶ out: pdf_base64, csv_string
        │
        ├──▶ servicenow-incidents.inputSchema  (receives scan_results)
        │           └─▶ out: incidents_created, incidents_closed
        │
        ├──▶ [optional] remediate-device.inputSchema  (per non-compliant device)
        │           └─▶ out: remediation_status, backup_id, post_eval_results
        │
        ├──▶ iag-compliance-slack-notify  (receives fleet_summary)
        │
        └──▶ email adapter  (receives pdf_base64 + csv_string)
```

## Shared Types

All component schemas reference `shared-types.schema.json`:

| Type | Used By |
|------|---------|
| `Rule` | rules-file, iag-evaluate (input), evaluate-grade (input) |
| `RuleResult` | iag-evaluate (output), remediate-device (output) |
| `DeviceResult` | iag-evaluate (output), shared within ScanResults |
| `ScanResults` | evaluate-grade (output), generate-report (input), servicenow-incidents (input) |
| `FleetSummary` | evaluate-grade (output), orchestrator (output), iag-slack-notify (input.summary) |
| `SnowIncidentRef` | servicenow-incidents (output), orchestrator (output) |
