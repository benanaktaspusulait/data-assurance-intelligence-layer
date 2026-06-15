# Governance, Security, and Scale

**Parent page:** Cerberos Data Assurance Intelligence Layer - Initial Discovery Concept  
**Status:** Private working draft

## Governance Principles

The capability must be governed from the start.

Principles:

- Rules are approved before execution.
- Rules are versioned.
- Findings reference the exact rule version that generated them.
- Rule changes require human approval.
- Backtesting should be performed before deployment where data is available.
- Rule activation is staged: shadow, then canary (narrow slice), then full deployment, with automatic rollback if the false-positive rate exceeds an agreed threshold shortly after activation.
- Rollback should be possible.
- Exceptions are explicit, approved, versioned, and reviewed.
- Rule owners are accountable for rule intent and quality.
- Learning produces suggestions, not automatic changes.
- Audit trails cover rule creation, execution, feedback, suggestions, approval, and deployment.
- Audit records are tamper-evident: each audit entry stores the hash of the previous entry (a hash chain), and a periodic (for example daily) chain digest is stored separately in an immutable, signed location (for example a KMS-signed S3 object). This allows log integrity to be proven mathematically, which is appropriate for a high-audit border-security context.

## Indicative RACI for Discovery

| Activity | Responsible | Accountable | Consulted | Informed |
| --- | --- | --- | --- | --- |
| Rule definition | Data/platform team | Rule owner | Domain SMEs | Delivery |
| Finding review | Domain SME | Data owner | Platform team | Ops |
| Rule approval | Rule owner | Architecture/Governance | Security | Delivery |
| Platform operation | Platform team | Engineering owner | DevOps/SRE | Users |

This RACI is indicative and uses generic role names (rule owner, data owner, domain SME, platform
owner, security/governance reviewer, delivery stakeholder, SRE/DevOps). The actual responsible,
accountable, consulted, and informed roles must be validated during discovery once real teams and
owners are identified.

## Security and Privacy Controls

Required controls:

- Read-only access.
- Replica-first or analytical-source-first execution.
- Approved query templates.
- No production writes.
- No uncontrolled ad-hoc queries.
- No autonomous remediation.
- RBAC and IAM integration.
- Least privilege.
- PII masking.
- Output minimisation.
- Secure storage.
- Audit logging.
- Human approval.
- Retention policy.
- Sensitive field controls.
- Query allow-list.
- Query deny-list for `UPDATE`, `DELETE`, `DROP`, `ALTER`, `TRUNCATE`, and similar operations.
- Encryption at rest and in transit.
- Security and compliance review.

Agent safety:

> Agents analyse controlled findings, structured metadata, masked evidence, and governed context. They do not receive unrestricted raw sensitive data or direct production database access.

## Border-Security Legal and AI Controls

This capability operates in an OFFICIAL-SENSITIVE border-security context. The following controls are
mandatory and are expanded in the child page **Border-Security Constraints and Pre-Funding Conditions**.

- **Masked data is still personal data.** Under UK GDPR and the Data Protection Act 2018, masked or
  pseudonymised findings remain personal data; only genuinely anonymised data is out of scope. A Data
  Protection Impact Assessment (DPIA) must be completed and signed before any agent processing.
- **Agent output is sensitive.** Agent prompts and responses can generate sensitive inferences. They
  must be classified and stored in a separate, access-controlled audit zone, and be reviewable by a
  human supervisor within 24 hours.
- **No PII to agents without authorisation.** Agents receive masked, bounded findings only. Raw or
  masked PII must not be provided to an agent without case-level authorisation.
- **Two-person rule for rule changes.** No rule change (including agent-suggested refinements) may be
  deployed without two-person (2PR) approval, after a mandatory 7-day live shadow window with no alerts.
- **Protected-attribute prohibition.** Rules keyed on protected attributes (for example nationality
  cohorts) must not be created or deployed without explicit governance sign-off, given the diplomatic
  and equality risks. Rule definitions declare this via the governance metadata.
- **Bounded findings.** Per-rule daily finding ceilings and auto-suppression are required so that
  review load stays sustainable at platform scale.

## Data Classification Model

Each rule and finding should declare the maximum data classification it may access and the maximum classification it may output.

| Classification | Description | PoC Handling |
| --- | --- | --- |
| Public / non-sensitive metadata | Public or non-sensitive technical metadata. | May be shown in dashboards if still relevant. |
| Internal operational metadata | Source names, rule IDs, counts, timings, owners, run status. | Suitable for findings and dashboards with normal internal access controls. |
| Sensitive operational data | Operational values or context that may reveal platform behaviour. | Minimise, mask where possible, and restrict by RBAC. |
| PII / identity-related data | Identity, document, person, journey, or other individual-related data. | Do not store raw values in findings; use masking, hashes, or references. |
| Highly restricted investigation-only data | Data requiring exceptional access or case-specific investigation. | Exclude from PoC findings unless explicitly approved by governance. |

Raw rows should not be stored directly in the findings store. Findings should store aggregate evidence, masked sample references where allowed, hashes, rule/run metadata, and links to governed evidence locations.

## Retention Decisions

Retention should be agreed during discovery for:

- Findings.
- Feedback.
- Masked evidence references.
- Audit events.
- Rule execution metadata.
- Agent prompts and outputs, if enabled. Recommended default: 90-day retention in the controlled agent-output audit zone, access restricted to audit and security teams, with automatic destruction after the retention period (subject to governance confirmation).
- Backtest results.
- Athena query result metadata and query execution IDs.

## S3, Glue and Athena Governance Controls

The analytical assurance layer introduces additional governance requirements for S3, Parquet, Glue Data Catalog and Athena.

Required controls:

- Separate raw and curated S3 zones with different access policies.
- Raw-zone access restricted to tightly controlled investigation and schema drift use cases.
- Curated Parquet datasets preferred for repeatable Athena DQ rules.
- Glue Crawler permitted for PoC discovery where appropriate.
- Critical curated schemas should move toward controlled Glue table definitions via IaC or an approved metadata workflow.
- Athena queries must run through `SafeQueryExecutor`, not directly from UI, agent, notebook, or arbitrary service code.
- Athena workgroups should enforce result location, encryption, and cost controls where available.
- Query execution audit should link findings to Athena query execution IDs.
- No raw PII should appear in Athena query outputs used for notifications or agent prompts.
- Agent/Copilot tooling must never directly query S3 or Athena.

## Scale Considerations

Cerberos produces approximately 5 billion events per month. The platform must avoid scanning all raw data continuously.

Recommended approaches:

- Incremental time-window checks.
- Partition-aware queries.
- Aggregate metrics.
- Precomputed counters.
- Sampling where appropriate.
- Exception-focused queries.
- Source and event-type grouping.
- Athena partition pruning.
- S3 Parquet optimisation.
- Query timeout.
- Concurrency limit.
- Cost control.
- Result row limits.
- Scheduling windows.
- Separation from production workload.
- Per-rule monthly finding quota with automatic deactivation when exceeded (see **Supporting Technical Detail**).

Rules should be designed to work against replicas or analytical datasets, not operational production databases.

Replica DB checks and Athena analytical checks should be treated as complementary. Replica databases are appropriate for recent operational read-only checks; S3/Parquet/Athena is appropriate for historical, aggregate, reconciliation, schema drift, and backtesting checks.

Detailed guidance is captured in **S3, Parquet, Glue Data Catalog and Athena Analytical Assurance Layer**.

## Alerting and Notification

| Severity | Suggested Handling |
| --- | --- |
| Critical | Immediate alert and ticket. |
| High | Alert within defined SLA. |
| Medium | Hourly or daily digest. |
| Low | Dashboard and reporting only. |

Alerting should include deduplication, grouping, cooldown periods, owner-based routing, and integration with Jira, ServiceNow, Teams, or Email.

Repeated findings should update an existing incident where appropriate rather than creating unnecessary duplicates.

## Finding Clustering

Deduplication and grouping are necessary but not sufficient at platform scale. At approximately 5
billion events/month, even a 0.1% anomaly rate implies millions of raw findings. Smart clustering
keeps review load sustainable.

- Findings from the same rule, same source, and same time window are collapsed into a single
  **representative finding** (a cluster), not shown individually.
- The cluster card shows: total affected record count, time range, trend (increasing, decreasing, or
  steady), and first and last occurrence.
- A reviewer decision (confirm, reject, exception) applies to the **whole cluster** at once, not to
  each member finding.
- Clustering complements auto-suppression and the per-rule finding caps described in
  **Border-Security Constraints and Pre-Funding Conditions**.

## Dashboard Views

Executive view:

- Overall data assurance score.
- Active critical findings.
- Findings by domain.
- Findings by source system.
- Findings by owner/team.
- Issue age.
- Confirmed versus rejected findings.
- Learning suggestions pending approval.

Technical view:

- Freshness status.
- Duplicate rate.
- Missing field rate.
- Reconciliation gaps.
- Schema drift events.
- False positive rate.
- Rule confidence score.
- Top recurring issues.
- Rule run history.
- Evidence quality feedback.
