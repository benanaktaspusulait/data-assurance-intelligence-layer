# Athena Analytical DQ Examples and Sub-Architectures

**Parent page:** Cerberos Data Assurance Intelligence Layer - Initial Discovery Concept  
**Status:** Private working draft  
**Purpose:** Worked Athena query examples, SafeQueryExecutor integration, reconciliation, backtesting, schema drift, and agent-layer relationship
**Companion page:** See **S3, Parquet, Glue Data Catalog and Athena Analytical Assurance Layer** for the concept, zones, partitioning, governance, cost controls, and PoC scope.

## 1. Example Athena Data Quality Queries

The examples below are generic and do not assume exact Cerberos table names.

Example SQL is Athena/Presto-style only. Final SQL should be adapted to actual table definitions, partition columns, data types, and approved query templates. These examples are not intended to be copied directly into production rule definitions without source-specific validation.

### 1.1 Missing Field Rate by Source

```sql
select
  source_system,
  count(*) as total_count,
  sum(case when nationality is null then 1 else 0 end) as missing_nationality_count,
  sum(case when document_number is null then 1 else 0 end) as missing_document_count,
  sum(case when date_of_birth is null then 1 else 0 end) as missing_dob_count,
  100.0 * sum(case when nationality is null then 1 else 0 end) / count(*) as missing_nationality_rate
from curated_identity_events
where year = :year
  and month = :month
  and day = :day
  and hour between :start_hour and :end_hour
group by source_system;
```

### 1.2 Source Volume Baseline

```sql
with current_hour as (
  select source_system, count(*) as current_count
  from curated_events
  where year = :year
    and month = :month
    and day = :day
    and hour = :hour
  group by source_system
),
baseline as (
  select source_system, avg(hourly_count) as avg_7_day_count
  from assurance_hourly_source_counts
  where metric_date between :baseline_start and :baseline_end
  group by source_system
)
select
  c.source_system,
  c.current_count,
  b.avg_7_day_count,
  c.current_count - b.avg_7_day_count as delta
from current_hour c
join baseline b on c.source_system = b.source_system
where c.current_count < b.avg_7_day_count * 0.5;
```

### 1.3 Inbound vs Processed Reconciliation

```sql
select
  i.source_system,
  i.event_type,
  i.window_start,
  i.inbound_count,
  p.processed_count,
  i.inbound_count - p.processed_count as count_gap,
  100.0 * abs(i.inbound_count - p.processed_count) / greatest(i.inbound_count, 1) as gap_percent
from assurance_inbound_counts i
join assurance_processed_counts p
  on i.source_system = p.source_system
 and i.event_type = p.event_type
 and i.window_start = p.window_start
where abs(i.inbound_count - p.processed_count) > :allowed_gap;
```

### 1.4 Schema Drift Detection

```sql
select
  o.source_system,
  o.event_type,
  o.field_name,
  e.expected_type,
  o.observed_type,
  o.observed_at
from observed_schema_snapshot o
left join expected_schema_registry e
  on o.source_system = e.source_system
 and o.event_type = e.event_type
 and o.field_name = e.field_name
where e.field_name is null
   or o.observed_type <> e.expected_type;
```

### 1.5 Duplicate External Reference

```sql
select
  source_system,
  event_type,
  external_reference,
  count(*) as duplicate_count
from curated_events
where year = :year
  and month = :month
  and day = :day
group by source_system, event_type, external_reference
having count(*) > 1;
```

### 1.6 Backtesting Candidate Rule

```sql
select
  source_system,
  event_type,
  date_trunc('hour', event_timestamp) as window_start,
  count(*) as affected_count
from curated_identity_events
where event_date between :backtest_start and :backtest_end
  and nationality is null
group by source_system, event_type, date_trunc('hour', event_timestamp)
having count(*) > :candidate_threshold;
```

This should create shadow results only, not live findings or alerts.

## 2. Integration with SafeQueryExecutor

Athena must not be queried directly from UI, agent, or arbitrary service code.

All Athena access should go through:

- Approved rule definition.
- `SafeQueryExecutor`.
- Athena connector.
- Query parameter validation.
- Partition constraint validation.
- Timeout.
- Scan cost limit.
- Result row limit.
- Audit logging.
- Result masking.

Interface example:

```java
public interface AthenaAssuranceConnector {
    QueryResult executeAthenaRule(ApprovedRule rule, QueryParameters params);
    QueryCostEstimate estimateCost(ApprovedRule rule, QueryParameters params);
    ConnectorHealth health();
}
```

The executor should reject queries that do not constrain partitions, exceed cost limits, request disallowed columns, or attempt unsupported operations.

## 3. Reconciliation Architecture

For Cerberos scale, reconciliation is one of the most important analytical assurance use cases.

Examples:

- Inbound events vs processed records.
- Processed records vs operational DB records.
- Operational DB records vs S3 archive.
- S3 archive vs reporting dataset.
- Source-level count vs platform-level count.
- Kafka consumed offsets vs stored event counts.

```text
 Source
   |
   v
 Inbound
   |
   v
 Processing
   |
   v
 Operational DB
   |
   v
 S3 Archive / Curated Data
   |
   v
 Reporting Dataset

 Data Assurance Reconciliation:
   inbound_count
   processed_count
   operational_count
   archive_count
   reporting_count
   kafka_offset_count
```

Reconciliation should use aggregate counts and windowed metrics, not row-by-row comparison unless necessary.

Reconciliation findings should include:

- Source system.
- Event type.
- Window.
- Expected count.
- Actual count.
- Difference.
- Percentage gap.
- Severity.
- Evidence reference.

## 4. Backtesting and Rule Simulation

Athena and S3 historical partitions support candidate rule backtesting.

Flow:

1. Learning engine suggests a candidate rule.
2. Candidate rule is executed in shadow mode against historical partitions.
3. Results are compared with previously confirmed and rejected findings.
4. Backtest report is produced.
5. Human owner approves or rejects the rule change.

Principles:

- Shadow mode must not create live alerts.
- Shadow findings should be stored separately.
- Backtest results support governance.
- Athena is suitable because it can query historical partitioned datasets.

Example backtest result:

```yaml
tested_windows: 672
historical_findings: 118
matched_confirmed_issues: 42
likely_false_positives: 9
missed_confirmed_issues: 2
estimated_scan_cost: "TBD"
recommendation: "Proceed to human review"
```

## 5. Schema Drift Architecture

S3, Glue, and Athena can support schema drift detection by comparing observed schema history with expected schema definitions.

Potential sources:

- Raw payload observations.
- Glue Data Catalog schema changes.
- Expected schema registry.
- Observed schema snapshots.
- Transformation schema versions.

Drift types:

- New field.
- Missing field.
- Changed type.
- Changed enum.
- Changed date format.
- Changed nested structure.
- Unexpected nullability.

Flow:

```text
Raw / curated data
  -> observed schema snapshot
  -> compare with expected schema
  -> drift finding
  -> review UI
  -> feedback
  -> rule or contract update
```

Glue Crawler alone should not silently normalise drift. Drift should be detected, reviewed, and governed.

## 6. Relationship with Agent / Copilot Layer

Agent/Copilot should not directly query Athena.

Correct model:

```text
Athena rule execution
  -> controlled result
  -> finding summary
  -> agent analysis
  -> human review
```

Agent can help with:

- Summarising Athena query results.
- Explaining reconciliation gaps.
- Suggesting likely root causes.
- Grouping repeated findings.
- Drafting incident text.
- Suggesting rule refinements.
- Analysing feedback patterns.

Agent must not:

- Create arbitrary Athena SQL.
- Query raw S3 data directly.
- Access raw PII.
- Trigger unapproved backtests.
- Deploy rules automatically.

The agent consumes structured, masked, bounded outputs.
