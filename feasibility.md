# Feasibility Assessment: Network Compliance Audit

**Decision: FEASIBLE**
**Platform:** https://platform-6-aidev.se.itential.io
**Assessed:** 2026-04-07

---

## Environment Summary

IAP 6.2.0 cloud instance with 16 adapters and 19 applications, all RUNNING. The platform has full device management (ConfigurationManager), command execution (MOP + IAG), workflow orchestration (WorkflowEngine), ServiceNow and Email adapters online, and two IAG instances for running scripts. 33 devices are registered across 8 device groups, including a "Cisco Devices" group that maps directly to the audit scope. 100 existing workflows are available, with several Slack notification and ServiceNow patterns available for reuse.

---

## Capabilities Resolution

| Spec Capability | Status | Resolution |
|----------------|--------|------------|
| Pull running config via SSH (IOS/NX-OS) | ✓ | ConfigurationManager + MOP Command Templates + IAG (`selab-compute-iag`) |
| Evaluate config against pattern-based rules | ✓ | JST (JSON Schema Transformation) for data shaping + workflow logic for regex evaluation; IAG Python scripts for complex rule matching |
| Score and aggregate results | ✓ | WorkflowEngine + JST transformations for per-device and fleet-level scoring |
| Generate PDF reports | ✓ | IAG5 Python script (reportlab/fpdf2) generates PDF from structured scan results via `selab-compute-iag` |
| Generate CSV reports | ✓ | JST + TemplateBuilder produce CSV; Email adapter delivers it |
| Track compliance posture over time | ✓ | Results stored as JSON in IAP's MongoDB-backed state store per scan run; trend data queryable across runs |
| Apply config changes for remediation | ✓ | MOP + `Push Configuration to Device` workflow (reuse) + IAG |
| Schedule recurring scans | ✓ | OperationsManager triggers (RUNNING) support cron-based scheduling |

---

## Integrations Resolution

| Spec Integration | Status | Resolution |
|-----------------|--------|------------|
| ServiceNow — open/close incidents | ✓ | `ServiceNow` adapter (v2.9.5, RUNNING, ONLINE). `ServiceNow Create CR and Wait for Approval` workflow available for reuse as a pattern for incident creation. |
| Slack — scan summary notifications | ✓ | IAG5 Python script posts to Slack via incoming webhook URL. Two existing `Send Notification - Slack` workflows confirm this pattern is already in use on the platform. |
| Email — report delivery with PDF/CSV | ✓ | `email` adapter (v4.7.12, RUNNING, ONLINE) |

---

## Design Decisions (Engineer-Approved)

| Item | Decision |
|------|----------|
| PDF generation | IAG5 Python script — engineer confirmed |
| Slack notifications | IAG5 Python script via incoming webhook — engineer confirmed |
| Config rule evaluation | IAG5 Python script (regex/pattern matching) — engineer confirmed |
| Any missing capability | IAG5 Python scripts are the default implementation path |
| Device scoping | "Cisco Devices" device group — no OS-level filtering available in CMDB |

---

## Reuse Opportunities

| Existing Asset | Reuse In |
|---------------|----------|
| `Send Notification - Slack` (×2 namespaces) | Notify phase — adapt webhook call for compliance scan summary |
| `ServiceNow Create CR and Wait for Approval` | Incident creation for non-compliant devices (adapt: use Incident not CR) |
| `Update Change Request - ServiceNow` | Close/update ServiceNow incident when device becomes compliant |
| `Push Configuration to Device` | Remediation phase — reuse as-is for applying corrective config |
| `Command Template Runner` | Config collection phase — reuse MOP runner pattern for pulling running configs |
| `@4c935d4a3624d59a36110f17: NHC - Single Device` | Per-device iteration pattern — reuse loop structure for per-device compliance evaluation |

---

## Device Inventory

- **33 devices** registered in ConfigurationManager
- **"Cisco Devices" group** present — use as default audit scope
- Device OS metadata is null in CMDB; scope will be driven by group membership
- IAG instances (`selab-compute-iag`, `selab-iag-4.4`) available for SSH-based config collection

---

## Feasibility Decision

**FEASIBLE**

All required capabilities are achievable on this platform. IAG5 Python scripts (`selab-compute-iag`, RUNNING/ONLINE) are the approved implementation path for PDF generation, Slack notifications, config rule evaluation, and any other capability not covered by a native adapter. No blockers. Design can proceed.
