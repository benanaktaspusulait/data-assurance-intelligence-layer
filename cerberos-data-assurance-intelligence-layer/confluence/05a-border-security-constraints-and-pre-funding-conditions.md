# Border-Security Constraints and Pre-Funding Conditions

**Parent page:** Cerberos Data Assurance Intelligence Layer - Initial Discovery Concept  
**Status:** Private working draft  
**Purpose:** Capture the OFFICIAL-SENSITIVE border-security constraints, security/governance red flags, and the conditions that must be met before any funding or AI-agent work is approved

## Framing

This concept does **not** propose autonomous 7x24 AI decision-making. AI is optional, advisory, behind a
stub interface, deterministic-first, and every rule change requires human approval. That stance is the
primary risk control and lowers inherent risk.

The residual risk is that **continuous (7x24) advisory analysis** against an approximately 5 billion
event/month border platform creates scale, legal, and sustainability exposure. This page records those
constraints so they are not lost as the concept matures. It treats "7x24 agent operation" as
*continuous advisory analysis*, never autonomous action.

## Red Flags - Must Fix Before PoC Funding

| # | Area | Required control before funding |
| --- | --- | --- |
| 1 | False-positive avalanche | At 5B events/month, a 0.1% anomaly rate is ~5M findings/month. Mandate per-rule daily finding ceilings, auto-suppression of repeat findings, adaptive thresholds, and human-in-the-loop for Critical/High findings only. Digests and cooldowns alone are insufficient. |
| 2 | AI legal basis (UK GDPR / DPA 2018) | Masked/pseudonymised data is still personal data; only genuinely anonymised data is out of scope. Complete a DPIA before any agent processing. Store agent prompts/outputs in a separate, access-controlled audit zone. Human supervisor review within 24h. No raw or masked PII to an agent without case-level authorisation. |
| 3 | Framework accreditation | No agent framework (Koog, LangChain4j, Spring AI) has demonstrated accreditation suitable for OFFICIAL-SENSITIVE. Freeze agent-framework selection for 12 months; ship deterministic feedback only. Require penetration-test results, a vulnerability disclosure programme, and support SLAs before any LLM. |
| 4 | 7x24 human sustainability | Continuous findings without funded reviewers are unworkable. Fund a tiered triage and on-call rota. If 24x7 reviewers are not funded, agent or analytical output must not drive real-time alerts. |
| 5 | Rule-change governance | A mis-scoped rule (for example one keyed on a nationality cohort) is a diplomatic risk. Agent suggestions must never auto-deploy. Require two-person rule (2PR) approval, a mandatory 7-day live shadow window with no alerts, and a prohibition on rules keyed on protected attributes without explicit governance sign-off. |

## Amber Flags - Address During PoC

| # | Area | Action during PoC |
| --- | --- | --- |
| 6 | Real-time vs batch boundary | State plainly that this is an assurance layer, not an inline operational gate. Athena is not a low-latency path; replica DBs have replication lag. Define acceptable lag. Real-time mandatory checks belong in the ingestion path. |
| 7 | Cost and ROI | No figures exist yet. Run a metered PoC with a hard cost cap (for example total spend at or below GBP/USD 5k) and a success metric of manual-investigation hours saved per week. |
| 8 | UK government technology alignment | Confirm a UK-region, OFFICIAL-SENSITIVE-accredited AWS landing zone before relying on Athena/S3. Prefer Spring Boot over Quarkus for accreditation familiarity. Require a separate AI impact assessment. |

## Green Flags - Already Adequate

- Read-only only, mandatory `SafeQueryExecutor`, SELECT-only plus deny-list, approved templates, query/cost/row limits.
- Human approval before any rule change; no autonomous remediation; no agent production-query or write path.
- Five-level data classification with per-rule maximum input/output classification; PII masking and output minimisation.
- Deterministic learning loop that works without an LLM; agent strictly behind an interface or stub.
- Shadow-mode backtesting, rule versioning, rollback, and a Gate 1 to Gate 5 rollout structure.
- Self-monitoring, observability, and audit-versus-observability separation.

## Indicative Cost Order of Magnitude

These are indicative assumptions to be replaced with measured PoC figures, not commitments.

- Athena, top 10 rules/day: with partition pruning on curated Parquet, roughly 0.05-0.5 TB/day scanned, on the order of single-digit to low-tens of dollars per day. Without partition filters this can rise 50-100x, which is why partition filters are a mandatory control.
- S3 curated 90-day zone: at ~5B events/month and ~0.5 KB compressed Parquet per event, roughly 2.5 TB/month, about 7.5 TB at 90 days, on the order of low hundreds of dollars per month.
- PostgreSQL writes: dominated by finding volume, which is why finding caps (Red flag 1) matter. Bounded findings imply a modest managed instance; unbounded findings do not.
- Agent API calls: Critical/High-only summarisation is negligible; summarising all findings is the cost and compliance trap and must be avoided by design.

ROI baseline is currently unknowable because current manual-investigation cost is not captured. Hours
saved per week is therefore the correct PoC success metric.

## Recommended Next Action - Conditional Go (deterministic-only PoC)

No-Go for any AI-agent-in-the-loop funding now. Conditional Go for a narrow, deterministic PoC under
these conditions:

1. AI/agent layer out of scope for the PoC (12-month framework freeze); deterministic feedback and confidence scoring only.
2. DPIA completed and signed before any agent work is scheduled.
3. One domain, one safe read-only source, five or fewer rules, with hard finding caps and auto-suppression from day one.
4. Funded tiered triage with Critical/High on-call SLA; otherwise no real-time alerting.
5. Two-person rule plus a 7-day live shadow for every rule change; protected-attribute rules blocked pending governance.
6. UK-region, OFFICIAL-SENSITIVE-accredited AWS landing zone confirmed; Spring Boot preferred for accreditation.
7. Metered PoC with a hard cost cap and success measured as manual-investigation hours saved per week with zero production impact.

## Mapping to Rollout Gates

| Condition | Gate (see PoC, Roadmap, and Risks) |
| --- | --- |
| Real-time boundary stated; data sources understood | Gate 1 |
| Accredited AWS landing zone; technology alignment | Gate 2 |
| Finding caps, tiered triage/on-call, 2PR + 7-day shadow agreed | Gate 3 |
| Metered PoC value with cost cap demonstrated | Gate 4 |
| DPIA, agent accreditation, retention, and wider controls approved | Gate 5 |
