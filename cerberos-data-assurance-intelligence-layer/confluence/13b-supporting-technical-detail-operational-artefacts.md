# Supporting Technical Detail - Operational Artefacts

**Parent page:** Cerberos Data Assurance Intelligence Layer - Initial Discovery Concept  
**Status:** Private working draft  
**Purpose:** Notification templates, agent prompt/guardrails, integration test scenarios, tooling comparison, and rule complexity/value/quota
**Companion page:** See **Supporting Technical Detail** for the evidence model, data model, schemas, sequences, confidence/adaptive thresholds, and backtesting.

## 1. Notification Templates

### Teams / Slack (High or Critical)

```text
[Cerberos DQ] {severity} finding in {domain}
Rule: {rule_name} (v{rule_version})
Source: {source_system}
Affected: {affected_count} records in {time_window}
Baseline: {baseline_value} | Current: {current_value}
Confidence: {confidence_score}
Review: {review_url}
```

### Email (digest item)

```text
Subject: [Cerberos DQ] Daily assurance digest - {date}

{count_critical} critical, {count_high} high, {count_medium} medium findings.

Top findings:
- {rule_name} | {domain} | {source_system} | {affected_count} | {review_url}
...

Full dashboard: {dashboard_url}
This is an automated assurance summary. No action is taken automatically.
```

### Jira / ServiceNow (ticket body)

```text
Summary: [DQ][{severity}] {rule_name} - {domain}
Description:
  Finding ID: {finding_id}
  Rule: {rule_name} (v{rule_version})
  Source: {source_system}
  Affected count: {affected_count}
  Time window: {time_window}
  Evidence (masked): {evidence_reference}
  Suggested owner: {owner_team}
Labels: cerberos, data-assurance, {domain}
```

All templates use masked references only. Raw PII must never appear in a notification.

## 2. Agent Prompt Template and Guardrails

When the optional `AgentAnalysisService` is enabled, the agent receives only structured, masked context.

System prompt skeleton:

```text
You are a data assurance analysis assistant for the Cerberos platform.
You receive structured, masked data quality finding metadata.
You must:
- Summarise the finding in plain language.
- Suggest likely root-cause hypotheses (clearly labelled as hypotheses).
- Suggest a candidate owner team from the provided list only.
You must NOT:
- Request or infer raw PII.
- Recommend or perform any data modification.
- Make operational or security decisions.
- Output anything other than the requested structured fields.
All output is advisory and subject to human review.
```

Input contract (masked):

```json
{
  "finding_id": "dq-finding-...",
  "rule_name": "Missing critical identity fields",
  "domain": "passenger-identity",
  "source_system": "CarrierGateway-A",
  "severity": "HIGH",
  "affected_count": 847,
  "baseline": "0.3%",
  "current": "7.2%",
  "recent_deployments": ["transform-v2.14"],
  "candidate_owner_teams": ["ingestion-platform-team", "identity-data-platform"]
}
```

Guardrails enforced outside the model:

- Input is built from masked finding metadata only; no raw rows are passed.
- Output is parsed into a typed `AgentAnalysis` object; free text is length-limited.
- Prompt and response are written to the audit log.
- The agent endpoint must be an approved enterprise LLM gateway.

## 3. Integration Test Scenarios (PoC)

| # | Scenario | Expected result |
| --- | --- | --- |
| 1 | Approved rule executes against replica and breaches threshold | Finding created, audit event written, notification routed by severity |
| 2 | Approved rule executes and stays within threshold | Run recorded, no finding, no notification |
| 3 | Rule query containing `UPDATE`/`DELETE` submitted | Rejected by SafeQueryExecutor, no execution, audit event written |
| 4 | Query exceeds timeout or row limit | Execution aborted, run marked failed, no partial finding persisted |
| 5 | Connector to source unavailable | Run marked failed, health check reflects unhealthy, alert raised |
| 6 | Reviewer confirms a finding | Feedback stored, confidence pattern updated on next learning run |
| 7 | Reviewer marks false positive | Confidence score decreases for that rule segment |
| 8 | Learning job detects high false-positive rate | Rule suggestion created with status `pending_review` |
| 9 | Suggested refinement backtested | Backtest report produced; no writes to live findings store |
| 10 | Finding evidence requested | Only masked samples returned; raw PII never exposed |

## 4. Data Quality Tooling Comparison

A focused comparison for the "build vs reuse" decision deferred in the main report.

| Tool | Strengths | Gaps for this concept |
| --- | --- | --- |
| Great Expectations | Rich expectation library, validation, docs generation | No governed human-feedback learning loop or review workflow |
| Deequ (Spark) | Scales on large datasets, constraint suggestions | Spark dependency; no review/feedback model; JVM but batch-oriented |
| Soda (Soda Core/SodaCL) | Readable checks language, scheduling, alerting | Feedback-driven rule evolution and audit governance not native |
| dbt tests | Simple, integrates with transformation layer | Limited to warehouse transforms; no finding review workflow |
| AWS Glue Data Quality | Managed, DQDL rules, AWS-native | Less suited to human-in-the-loop learning and custom governance |

Conclusion: the PoC should validate whether existing tools can be reused for the rule execution
layer, while the custom part stays focused on the governed learning loop (reviewed findings,
structured feedback, confidence, exception discovery, human-approved evolution), which is not native
to these tools. This keeps the build vs reuse decision open and evidence-led rather than assuming a
custom platform up front.

## 5. Rule Complexity, Value, and Finding Quota

These deterministic metrics keep the rule set sustainable as it grows from five rules toward hundreds.

### Complexity Score

Each rule is assigned a complexity score (1-10) derived from its shape:

- Partition filter present (lower complexity if yes).
- Number of joined tables.
- Number of functions/expressions used.

The score is used to prioritise rule review and cost optimisation. High-complexity, low-value rules
are placed on a "rewrite recommended" list. The learning engine can suggest simpler rewrites of
similar rules using a rule-based (non-ML) approach.

### Rule Value Score

```text
value_score = (confirmed_findings * severity_weight) / (execution_cost + review_time)
```

Low value-score rules are surfaced for owner review and possible retirement, so effort and cost stay
focused on rules that produce actionable assurance.

### Monthly Finding Quota

Each rule has a maximum monthly finding quota (for example 10,000). When the quota is reached:

- The rule is automatically set to **inactive**.
- The owner is notified: "this rule is producing too many findings, please review the threshold".

This protects reviewers and storage from a single misbehaving rule, and complements the per-rule
daily ceilings and auto-suppression in **Border-Security Constraints and Pre-Funding Conditions**.
