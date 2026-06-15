# Rule Types, Data Model, and Examples

**Parent page:** Cerberos Data Assurance Intelligence Layer - Initial Discovery Concept  
**Status:** Private working draft  
**Purpose:** Supporting technical detail for the initial discovery concept

## Rule Categories

This page provides example rule categories, a conceptual data model, and sample rule definitions. The examples are illustrative and would need validation against actual Cerberos data structures, governance requirements, and safe read-only sources.

Rules may run against different safe source patterns:

- Replica DB rules for recent operational checks such as freshness, referential integrity, status validation, and short-window consistency.
- Athena analytical rules for historical, aggregate, partition-aware checks such as reconciliation, baseline comparison, schema drift, trend analysis, and backtesting.

SQL examples are illustrative only. Dialect should be adapted for PostgreSQL, Oracle, SQL Server, Athena/Presto, or the approved source technology as appropriate. These examples are not intended to be copied directly into production rule definitions without source-specific validation against actual table names, column types, and partition columns.

Athena rules must still be approved rule executions. They must go through `SafeQueryExecutor`, use partition constraints and cost controls, and must not be generated or executed directly by an agent. Detailed analytical guidance is captured in **S3, Parquet, Glue Data Catalog and Athena Analytical Assurance Layer**.

## 1. Completeness Checks

Completeness checks detect missing mandatory or conditionally mandatory fields such as document number, nationality, date of birth, journey reference, event timestamp, source event ID, or carrier reference.

Example:

```sql
select
  source_system,
  event_type,
  count(*) as affected_count
from journey_identity_events
where event_timestamp >= :window_start
  and event_timestamp < :window_end
  and (
    document_number is null
    or nationality is null
    or date_of_birth is null
  )
group by source_system, event_type
having count(*) > :minimum_affected_count;
```

## 2. Freshness Checks

Freshness checks detect whether source systems are still sending expected data within expected time windows.

Example:

```sql
select
  source_system,
  max(event_timestamp) as latest_event_timestamp,
  date_diff('minute', max(event_timestamp), current_timestamp) as minutes_since_last_event
from inbound_event_store
where source_system = :source_system
group by source_system
having date_diff('minute', max(event_timestamp), current_timestamp) > :freshness_threshold_minutes;
```

## 3. Duplicate Detection

Duplicate checks identify duplicate journeys, events, external references, identity records, or idempotency failures.

Example:

```sql
select
  external_reference,
  source_system,
  count(*) as duplicate_count
from journey_events
where event_timestamp >= :window_start
  and event_timestamp < :window_end
group by external_reference, source_system
having count(*) > 1;
```

## 4. Referential Integrity

Referential integrity checks detect records that reference missing parent entities, such as journey events without a journey or status records without the relevant business entity.

Example:

```sql
select
  e.event_id,
  e.journey_id,
  e.source_system
from journey_event e
left join journey j on e.journey_id = j.journey_id
where e.event_timestamp >= :window_start
  and e.event_timestamp < :window_end
  and j.journey_id is null;
```

## 5. Status Transition Validation

Status transition checks detect invalid lifecycle transitions, such as `PROCESSED` without `VALIDATED`, `COMPLETED` after `FAILED`, or `ARCHIVED` without required intermediate state.

Example:

```sql
select
  entity_id,
  previous_status,
  current_status,
  transition_timestamp
from entity_status_transitions
where transition_timestamp >= :window_start
  and transition_timestamp < :window_end
  and concat(previous_status, '->', current_status) not in (
    'RECEIVED->VALIDATED',
    'VALIDATED->PROCESSED',
    'PROCESSED->COMPLETED',
    'FAILED->RETRIED',
    'COMPLETED->ARCHIVED'
  );
```

## 6. Reconciliation Checks

Reconciliation checks compare counts across pipeline stages, such as inbound count, processed count, stored count, archived count, and downstream count.

Example:

```sql
select
  source_system,
  inbound_count,
  processed_count,
  stored_count,
  abs(inbound_count - processed_count) as inbound_processed_gap
from dq_pipeline_counts
where window_start = :window_start
  and window_end = :window_end
  and abs(inbound_count - processed_count) > :allowed_gap;
```

At Cerberos scale, reconciliation rules should usually prefer aggregate counts and windowed metrics. Athena over curated Parquet datasets is a strong fit for historical reconciliation, while replica DBs remain useful for recent operational checks.

## 7. Schema Drift and Data Contract Checks

Schema drift and data contract checks detect changes in field names, types, formats, enum values, payload shape, or source contract compliance.

Example:

```sql
select
  source_system,
  field_name,
  observed_type,
  expected_type,
  count(*) as occurrence_count
from observed_payload_schema
where observation_timestamp >= :window_start
  and observation_timestamp < :window_end
  and observed_type <> expected_type
group by source_system, field_name, observed_type, expected_type;
```

For S3-backed datasets, schema drift should not be hidden by uncontrolled crawler updates. Glue Crawler can support PoC discovery, but critical curated schemas should be governed through controlled Glue table definitions or an approved metadata workflow.

## 8. Anomaly and Cross-System Consistency Checks

Anomaly checks detect unusual source volumes, missing-rate spikes, abnormal duplicate rates, or sudden drops in events. Cross-system checks compare operational, archive, reporting, and downstream datasets.

Example:

```sql
select
  source_system,
  event_type,
  current_hour_count,
  baseline_hourly_avg,
  baseline_hourly_stddev
from dq_volume_baseline
where window_start = :window_start
  and current_hour_count < baseline_hourly_avg - (3 * baseline_hourly_stddev);
```

Historical baselines and cross-system consistency checks are often better suited to S3/Parquet/Athena than replica DBs, provided queries are partition-aware, bounded, audited, and cost-controlled.

## Conceptual Data Model

### dq_rule

- `id`
- `rule_code`
- `version`
- `name`
- `domain`
- `description`
- `severity`
- `owner_team`
- `query_template`
- `schedule`
- `threshold_config`
- `source_type`
- `pii_level`
- `enabled`
- `status`
- `created_by`
- `approved_by`
- `created_at`
- `updated_at`

### dq_rule_run

- `id`
- `rule_id`
- `rule_version`
- `started_at`
- `finished_at`
- `status`
- `execution_time_ms`
- `source`
- `row_count`
- `error_message`
- `cost_estimate_units`
- `audit_reference`

### dq_finding

- `id`
- `rule_id`
- `rule_version`
- `run_id`
- `severity`
- `confidence_score`
- `status`
- `domain`
- `source_system`
- `event_type`
- `affected_count`
- `finding_summary`
- `evidence_hash`
- `sample_reference_masked`
- `first_detected_at`
- `last_seen_at`
- `assigned_team`
- `incident_reference`

### dq_feedback

- `id`
- `finding_id`
- `user_id`
- `decision`
- `reason_code`
- `comment`
- `corrected_severity`
- `corrected_owner`
- `is_known_exception`
- `created_at`

### dq_feedback_pattern

- `id`
- `pattern_key`
- `rule_id`
- `domain`
- `source_system`
- `event_type`
- `decision_summary`
- `approved_count`
- `rejected_count`
- `exception_count`
- `confidence_score`
- `last_updated_at`

### dq_rule_suggestion

- `id`
- `suggestion_type`
- `source_rule_id`
- `proposed_rule_yaml`
- `reason`
- `confidence_score`
- `supporting_findings`
- `backtest_result`
- `status`
- `reviewed_by`
- `reviewed_at`

### dq_exception

- `id`
- `rule_id`
- `rule_version`
- `domain`
- `source_system`
- `condition_expression`
- `reason`
- `approved_by`
- `valid_from`
- `valid_until`

### dq_agent_analysis

- `id`
- `finding_id`
- `summary`
- `impact`
- `likely_root_cause`
- `recommended_actions`
- `confidence`
- `generated_at`

### dq_metric_snapshot

- `id`
- `metric_name`
- `domain`
- `source_system`
- `value`
- `window_start`
- `window_end`
- `created_at`

### dq_audit_event

- `id`
- `event_type`
- `entity_type`
- `entity_id`
- `rule_id`
- `rule_version`
- `actor`
- `action`
- `detail`
- `correlation_id`
- `created_at`

This conceptual data model is the authoritative table set for the discovery concept. Other pages
(such as **Technology Selection and Architecture Decision Report** and the ER diagram in
**Supporting Technical Detail**) reference subsets of these tables for illustration.

## Approved Rule Example

Rule definitions use a single canonical flat structure. The same structure is used in
**Technology Selection and Architecture Decision Report** and is validated by the JSON Schema in
**Supporting Technical Detail**. Top-level keys are flat (for example `source_type`, `owner_team`,
`query`), rule IDs follow the pattern `CERB-DQ-<number>`, and optional blocks
(`evidence`, `agent_action`, `notification`, `governance`) are explicitly allowed by the schema.

```yaml
id: CERB-DQ-101
version: 1
name: Missing critical identity fields by source system
domain: identity
description: >
  Detects journey identity events where one or more critical identity fields
  are missing after the expected ingestion and validation window.
severity: HIGH
source_type: athena
source_name: analytical_events.journey_identity_events
query_type: aggregate
schedule: hourly_with_15m_delay
owner_team: identity-data-platform
escalation_team: platform-assurance
pii_level: masked_samples_only
classification:
  max_input_classification: pii_identity_related
  max_output_classification: internal_operational_metadata
query: |
  SELECT source_system, event_type,
         COUNT(*) AS total_count,
         SUM(CASE WHEN document_number IS NULL
                    OR nationality IS NULL
                    OR date_of_birth IS NULL
                  THEN 1 ELSE 0 END) AS affected_count,
         100.0 * SUM(CASE WHEN document_number IS NULL
                            OR nationality IS NULL
                            OR date_of_birth IS NULL
                          THEN 1 ELSE 0 END) / NULLIF(COUNT(*), 0) AS missing_rate_percent
  FROM journey_identity_events
  WHERE event_timestamp >= :window_start
    AND event_timestamp <  :window_end
  GROUP BY source_system, event_type
  HAVING SUM(CASE WHEN document_number IS NULL
                    OR nationality IS NULL
                    OR date_of_birth IS NULL
                  THEN 1 ELSE 0 END) > :minimum_affected_count;
threshold:
  expression: affected_count > 100 AND missing_rate_percent > 1.0
result_limits:
  max_rows: 1000
  timeout_seconds: 60
evidence:
  sample_mode: masked
  max_samples: 20
  include_baseline: true
  include_recent_deployments: true
agent_action:
  enabled: true
  allowed_actions:
    - summarise_finding
    - suggest_possible_owner
    - compare_with_previous_findings
  prohibited_actions:
    - run_uncontrolled_query
    - expose_raw_pii
    - remediate
notification:
  severity_routing:
    CRITICAL: teams_and_ticket
    HIGH: teams_and_ticket
    MEDIUM: daily_digest
    LOW: dashboard_only
governance:
  requires_masking: true
  allow_raw_samples: false
  requires_human_approval: true
```

## Candidate Rule Suggested by Learning Engine

```yaml
id: candidate-dq-rule-processing-014
suggestion_type: new_rule
source_rule_id: CERB-DQ-003
name: Processed records missing processed timestamp
domain: Journey Processing
description: >
  Detects records that have reached PROCESSED status but do not have a
  populated processed_timestamp after the standard processing delay.
reason: >
  Multiple confirmed findings indicate downstream reconciliation gaps where
  records are marked as PROCESSED but cannot be reliably ordered or audited
  because processed_timestamp is missing.
confidence_score: 0.82
supporting_findings:
  - dq-finding-2026-05-12-01984
  - dq-finding-2026-05-18-02213
  - dq-finding-2026-05-25-02877
backtest_result:
  tested_windows: 28
  historical_findings: 19
  matched_confirmed_issues: 16
  likely_false_positives: 2
  missed_confirmed_issues: 1
requires_human_approval: true
status: pending_review
```

## Agent Suggestion Review Example

This example shows the suggestion lifecycle described in **Human Review and Learning Loop**: an
agent proposes an owner, severity, and root cause; a reviewer edits it; and the correction is fed
back into routing and confidence learning. The suggestion is advisory only and still requires human
approval before any rule change.

```yaml
suggestion_id: dq-suggestion-2026-06-001
finding_id: dq-finding-2026-06-001
suggestion_type: owner_routing
agent_suggestion:
  suggested_owner: identity-data-platform
  suggested_severity: HIGH
  suggested_root_cause: "Possible transformation issue after v2.14 deployment"
  confidence_before_review: 0.71
reviewer_decision:
  status: edited
  corrected_owner: ingestion-platform-team
  corrected_severity: MEDIUM
  reason_code: wrong_owner
  comment: "Volume drop originates upstream at ingestion, not in identity transforms."
learning_action:
  update_owner_routing_pattern: true
  update_confidence_for_source_event_pattern: true
  confidence_after_review: 0.58
  led_to_rule_refinement: false
requires_human_approval: true
```

The `reason_code` values reuse the structured feedback options defined in
**Human Review and Learning Loop** (for example `wrong_owner`, `wrong_severity`,
`insufficient_evidence`). Tracking confidence before and after review lets the platform measure the
quality of its own suggestions over time, not only the quality of the underlying findings.

## Why This Model Supports Governance

The model keeps findings linked to rule versions, execution runs, sources, evidence references, feedback decisions, and suggestions. This supports auditability, trend analysis, false positive reduction, exception governance, owner routing improvement, and human-approved rule evolution.
