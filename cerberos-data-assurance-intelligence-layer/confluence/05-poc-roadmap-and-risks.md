# PoC, Roadmap, and Risks

**Parent page:** Cerberos Data Assurance Intelligence Layer - Initial Discovery Concept  
**Status:** Private working draft

## Example Scenario

Scenario: missing nationality spike from CarrierGateway-A.

1. A rule detects a missing nationality rate of 7.2% in the last hour.
2. The normal baseline is below 0.3%.
3. A High severity finding is created.
4. The UI shows affected count, source, time window, trend, baseline, and masked evidence.
5. A user confirms the issue and links it to a recent transformation deployment.
6. Feedback is stored.
7. The learning engine correlates the finding with similar confirmed issues.
8. A pattern is identified around source, event type, and transformation version.
9. A candidate rule refinement is suggested.
10. A human owner reviews and approves the refinement after backtesting.
11. Future detection improves.

## Possible PoC Scope

The PoC should be deliberately small. This page is the canonical PoC scope for the discovery pack.

Scope:

- One data domain.
- One safe read-only data source.
- Five approved rules.
- One findings store.
- One simple review UI.
- Confirm, reject, and reclassify feedback.
- Basic confidence scoring.
- One rule refinement suggestion.
- One candidate new rule suggestion.
- Daily summary report.
- No production writes.
- No automated remediation.
- No uncontrolled AI access.

Candidate PoC rules:

1. Missing mandatory identity field.
2. Source freshness check.
3. Duplicate external reference.
4. Broken journey-person reference.
5. Inbound versus processed reconciliation.

### Worked PoC Example (Concrete)

A concrete instantiation of the scope above, to accelerate discovery:

- **Domain:** Journey Processing.
- **Source:** `CarrierGateway-A` read replica (independent of live traffic).
- **Five rules:**
  1. **Completeness:** `nationality` missing in more than 1% of inbound journey events in the last 15 minutes.
  2. **Freshness:** no events received from `CarrierGateway-A` in the last 5 minutes.
  3. **Referential integrity:** a `journey_event` whose `journey_id` is absent from the `journey` table.
  4. **Reconciliation (replica-DB version):** more than 0.5% difference between events received and events reaching `processed` status in the last hour.
  5. **Anomaly (simple):** hourly volume per source more than 3 standard deviations below the trailing 7-day average.

These rules are simple, meaningful, and produce data-backed feedback immediately. They map onto the
canonical categories in **Rule Types, Data Model, and Examples**.

## Success Criteria

- Known issue detected.
- False positive rate measured.
- User feedback captured structurally.
- Rule confidence calculated.
- At least one useful rule refinement suggested.
- No production impact.
- Output considered actionable by users.

## Measurable Success Metrics

Potential PoC measures:

- Percentage of findings reviewed within the agreed SLA.
- False positive rate measured by rule and source.
- Number of confirmed issues detected.
- Number of repeated manual investigations reduced or shortened.
- Average time to triage a finding.
- Number of useful rule refinements suggested.
- Athena scan cost per analytical rule.
- Zero production-impact incidents caused by the PoC.

## Risks and Mitigations

| Risk | Mitigation |
| --- | --- |
| Perceived as AI overreach | Position as governed platform assurance, not autonomous AI. |
| Security or privacy concerns | Use read-only sources, PII masking, least privilege, audit logging, and review. |
| Production performance impact | Use replicas, analytical datasets, bounded windows, and cost controls. |
| False positives | Capture feedback, track confidence, and backtest refinements. |
| Alert fatigue | Deduplicate, group, use cooldowns, and route lower severity findings to digest. |
| Unclear ownership | Capture corrected owner feedback and maintain rule owner accountability. |
| Lack of user engagement | Start with one domain and a lightweight review process. |
| Sensitive data exposure | Use masked samples, evidence hashes, output minimisation, and RBAC. |
| Organisational resistance | Mature privately and frame as discovery rather than criticism. |
| Scope becoming too large | Keep PoC narrow and defer advanced intelligence. |
| False-positive avalanche at scale | Per-rule daily finding ceilings, auto-suppression, adaptive thresholds, Critical/High-only human-in-the-loop. |
| No funded 7x24 reviewers | Funded tiered triage and on-call rota; without it, no real-time use of agent or analytical output. |
| Agent legal/compliance exposure | DPIA before any agent; treat masked findings as personal data; agent-output audit zone; supervisor review within 24h. |
| Unaccredited AI framework | 12-month freeze on agent-framework selection; deterministic feedback only until accreditation evidence exists. |
| Diplomatically sensitive rule | Two-person approval, 7-day live shadow, and prohibition on protected-attribute rules without governance sign-off. |

The full OFFICIAL-SENSITIVE constraints, red/amber/green assessment, and the conditions that must be met before funding are captured in the child page **Border-Security Constraints and Pre-Funding Conditions**.

## Phased Roadmap

### Phase 0 - Private Concept Maturity

- Document the idea.
- Collect observations.
- Identify examples.
- Avoid premature publication.

### Phase 1 - Discovery

- Understand current DQ issues.
- Map existing checks.
- Identify safe data sources.
- Identify stakeholders.
- Define PoC candidate.

### Phase 2 - Minimal PoC

- One domain.
- Five rules.
- Findings store.
- Simple UI.
- Feedback capture.
- Basic reporting.

### Phase 3 - Learning Loop

- Confidence scoring.
- Feedback pattern analysis.
- Known exception discovery.
- Rule refinement suggestions.

### Phase 4 - Platformisation

- Rule registry.
- Scheduler.
- RBAC.
- Audit.
- Dashboards.
- Notification integration.

### Phase 5 - Advanced Intelligence

- Anomaly detection.
- Deployment correlation.
- Schema drift correlation.
- Owner routing learning.
- Candidate rule generation.
- Agent-assisted summaries.

### Phase 6 - Controlled Remediation Support

- Remediation suggestions only.
- Human approval.
- Runbook integration.
- No autonomous data correction.

## Rollout Gates

Wider rollout should only be considered after explicit gates:

| Gate | Meaning |
| --- | --- |
| Gate 1 | Discovery validated: current capability, safe sources, and target domain are understood. |
| Gate 2 | Safe source approved: read-only replica or analytical source access is approved. |
| Gate 3 | PoC rules approved: rule owners, thresholds, evidence model, and review process are agreed. |
| Gate 4 | PoC value demonstrated: reviewed findings show actionable value without production impact. |
| Gate 5 | Security and governance approval: wider rollout controls, retention, ownership, and audit are approved. |

## Strategic Positioning

This concept should be positioned as a discovery topic, not as a final implementation proposal. The safest framing is platform assurance: approved read-only checks, human-reviewed findings, structured feedback, and governed learning.

Recommended positioning:

- Treat data quality as a platform assurance concern.
- Emphasise Cerberos scale and the border/security context.
- Avoid presenting the idea as autonomous AI.
- Avoid implying that existing teams or controls have failed.
- Start with a small read-only PoC.
- Use evidence from reviewed findings before proposing wider platformisation.

Suggested summary:

> Given Cerberos scale and the border/security context, data quality may need to be treated as a continuously improving assurance capability. The opportunity is not only to detect known issues, but to learn from reviewed findings and surface hidden data quality patterns over time.
