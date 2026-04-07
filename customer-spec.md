# Use Case: Network Compliance Audit

## 1. Problem Statement

Network teams must prove their devices meet security baselines and regulatory standards. Today this means an engineer manually pulls configs from hundreds of devices, eyeballs them against a spreadsheet of required settings, writes up exceptions, and produces a report for the auditor. It takes weeks, it's inconsistent, and by the time the report is done, configs have already drifted. Violations discovered late in the audit cycle are expensive to remediate under time pressure.

**Goal:** Continuously scan device configurations against defined compliance standards, grade every device, produce audit-ready reports, and optionally auto-remediate violations — turning compliance from a quarterly fire drill into an always-current posture.

---

## 2. High-Level Flow

```
Define Standards  →  Collect Configs  →  Evaluate  →  Grade  →  Report  →  Remediate
      |                    |                |           |           |            |
   Build or            Pull running     Compare      Score       Generate    Optional:
   import              config from      each         each        audit-      push fixes
   compliance          every device     config       device      ready       with approval
   rules per           in scope         against      (pass /     report      gate, re-scan
   standard                             applicable   partial /   PDF + CSV   to confirm
   (CIS + internal)                     rules        fail)
```

---

## 3. Phases

### Define Standards
Compliance rules are expressed as pattern-based checks (regex match / no-match, value comparison). Two standards are in scope:
- **CIS Benchmark** — industry-standard hardening rules for each device platform
- **Internal Corporate Policy** — custom rules defined by the security team

Rules are grouped by standard and versioned. Changing a rule creates a new version so historical audits remain valid.

### Collect Configs
Pull the running configuration from every device in scope via SSH. Scope is defined by device group or platform. Configs are collected as a point-in-time snapshot at scan start and stored as raw text for audit evidence.

### Evaluate
Compare each device's config against every applicable rule. Rules are scoped by platform (e.g., an IOS BGP rule does not apply to a PAN-OS firewall). Each rule produces: **compliant**, **non-compliant**, or **not-applicable**. Specific config lines that pass or fail each rule are captured.

### Grade
Score each device based on evaluation results:
- **Compliant** — all applicable rules pass
- **Partial** — non-critical violations only
- **Non-Compliant** — one or more critical violations

Aggregate scores by site, platform, and standard for executive summaries.

### Report
Generate reports at two levels:
- **Executive summary** — overall posture score, pass/fail counts, trend over time
- **Device detail** — per-device rule results with config excerpts

Output formats: **PDF** (for auditors) and **CSV** (for tracking/import into GRC tools).
Reports include timestamp, standard version, and full device list for reproducibility.

### Remediate (optional, approval-gated)
For violations with known fixes, remediation is available but **off by default**. When enabled:
1. Backup device config before any change
2. Apply corrective config
3. Re-evaluate the specific rule to verify fix
4. If verification fails, flag device and halt — do not retry

All remediations require **explicit human approval** before execution.

---

## 4. Key Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Standards versioned | Changing a rule creates a new version | Historical audits remain valid against the active standard at scan time |
| Evaluation is per-rule | Granular pass/fail/N-A per rule per device | Auditors need exact control visibility, not just a device-level pass/fail |
| Remediation off by default | Must be explicitly enabled per run | Compliance scans must be safe to run anytime without side effects |
| Approval gate on remediation | Human must approve before any config push | Prevents automated changes from causing unintended outages |
| Point-in-time snapshot | Configs collected at scan start, not live during evaluation | Consistent comparison — no mid-scan drift |
| Reports include raw evidence | Config excerpts and rule match details | Auditors can verify findings independently |
| Trend tracking enabled | Score history retained per standard per device | Supports continuous posture visibility and audit trend reporting |

---

## 5. Scope

**In scope:**
- Compliance standard definition: CIS Benchmark + Internal Corporate Policy rules
- Config collection via SSH from Cisco IOS/IOS-XE and NX-OS devices
- Rule evaluation with per-rule pass/fail/not-applicable results
- Device grading and fleet-level aggregation
- Audit-ready PDF and CSV report generation
- Trend tracking: compliance score history over time
- Optional remediation with backup, verification, and approval gate
- On-demand and scheduled scans (weekly default)

**Out of scope:**
- Defining what the compliance rules should be (provided by security/compliance team)
- Firmware or OS-level compliance (configuration only)
- Physical security audits, user access reviews, policy exception/waiver management
- Non-network infrastructure (servers, cloud workloads)
- Platforms other than Cisco IOS/IOS-XE and NX-OS (extendable later)
- ServiceNow CMDB sync or asset management (ticketing only)

---

## 6. Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| Rules too broad → false positives | Alert fatigue, audit noise | Scope rules by platform and context; allow not-applicable results |
| Config collection fails on some devices | Incomplete audit | Report unreachable devices explicitly; never mark them compliant |
| Standard changes mid-audit | Inconsistent results | Lock standard version at scan start |
| Auto-remediation introduces config error | Service impact | Backup before fix, verify after, approval gate required, disabled by default |
| Large fleet makes scanning slow | Audit window exceeded | Parallelize config collection; evaluate locally after collection |

---

## 7. Requirements

### Capabilities

| Capability | Required | Notes |
|-----------|----------|-------|
| Pull running config via SSH | Yes | IOS/IOS-XE and NX-OS |
| Evaluate config against pattern-based rules | Yes | Regex match/no-match, value comparison |
| Score and aggregate results across devices | Yes | By device, site, platform, standard |
| Generate PDF and CSV reports | Yes | Audit-ready with evidence |
| Track compliance posture over time | Yes | Score history per standard per device |
| Apply config changes for remediation | Optional | Off by default, approval-gated |
| Schedule recurring scans | Yes | Weekly default, configurable |

### Integrations

| System | Purpose | In Scope |
|--------|---------|----------|
| CMDB / inventory | Device list and platform info | No — device list provided manually or via static inventory |
| GRC platform | Standards and posture tracking | No — standards defined locally, reports exported to file |
| ServiceNow | Open incidents for non-compliant devices; track remediation tasks | Yes |
| Slack | Post scan summary and violation alerts to a designated channel | Yes |
| Email | Send audit report (PDF + CSV) to distribution list on scan completion | Yes |
| Report storage | Archive audit reports | No — reports stored locally on the platform |

---

## 8. Batch Strategy

| Strategy | Behavior | When to Use |
|----------|----------|-------------|
| Full sweep | Scan all in-scope devices in one run | Weekly scheduled scan, on-demand audit |
| Targeted | Scan specific device group or site | Post-change verification, incident response |

Rolling scans (subset per day) are out of scope for initial delivery.

---

## 9. Acceptance Criteria

1. Compliance standards (CIS + Internal) are defined with versioned, platform-aware rules for Cisco IOS/IOS-XE and NX-OS
2. Running configs are collected via SSH from all in-scope devices; unreachable devices are flagged explicitly and never marked compliant
3. Every device is evaluated against every applicable rule with a clear pass / fail / not-applicable result
4. Devices are graded (compliant / partial / non-compliant) and scores are aggregated by site, platform, and standard
5. Audit-ready PDF and CSV reports are generated with executive summary and device-level detail including config excerpts
6. Reports include timestamp, standard version, and full device list for reproducibility
7. Compliance score history is retained and surfaced in trend reporting
8. Remediation (when enabled) applies fixes only after human approval, with config backup and post-fix rule re-evaluation
9. Scans run on-demand and on a weekly schedule without manual intervention
10. A ServiceNow incident is opened for each non-compliant device, assigned to the network team, and closed when the device reaches compliant status
11. A Slack message is posted to a designated channel on scan completion with overall posture score and a count of violations by severity
12. An email with the PDF and CSV report attached is sent to a configured distribution list on scan completion
