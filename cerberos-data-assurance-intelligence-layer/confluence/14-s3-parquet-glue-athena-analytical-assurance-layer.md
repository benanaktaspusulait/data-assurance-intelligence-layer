# S3, Parquet, Glue Data Catalog and Athena Analytical Assurance Layer

**Parent page:** Cerberos Data Assurance Intelligence Layer - Initial Discovery Concept  
**Status:** Private working draft  
**Purpose:** Explain how S3, Parquet, Glue Data Catalog, Glue Crawler, and Athena support analytical assurance checks for Cerberos

## 1. Executive Summary

Cerberos produces approximately 5 billion events per month. At this scale, continuously scanning operational databases for historical or aggregate data quality analysis is not appropriate. The Data Assurance Intelligence Layer should separate near-operational read-only checks from analytical assurance checks.

Two complementary access patterns are required:

- **Replica DB checks:** recent-window, near-operational, read-only checks such as freshness, referential integrity, operational status validation, and recent event consistency.
- **S3/Parquet/Athena analytical checks:** historical, aggregate, partition-aware checks such as reconciliation, schema drift analysis, anomaly baselines, backtesting, trend analysis, and cross-system consistency.

S3, Parquet, Glue Data Catalog, and Athena provide a scalable and cost-aware analytical assurance layer. Curated datasets can be stored in partitioned Parquet format on S3. Glue Data Catalog provides metadata such as databases, tables, columns, and partitions. Athena can then query those curated datasets without placing load on operational systems.

This layer supports reconciliation, schema drift detection, anomaly baselines, backtesting rule suggestions, historical investigation, and data quality trend analysis. It does not replace replica DB checks, and Athena should not be treated as a low-latency online retrieval engine.

## 2. Why This Layer Is Needed

Cerberos-scale assurance requires historical and aggregate analysis. Some quality questions cannot be answered safely or efficiently by repeatedly querying operational databases.

Reasons this layer is needed:

- Cerberos operates at approximately 5 billion events per month.
- Operational databases are not ideal for repeated historical scans.
- Some quality issues are only visible through trend, baseline, or source-volume comparison.
- Reconciliation often requires comparing counts across pipeline stages.
- Backtesting candidate rules requires historical windows.
- Schema drift analysis may require raw or curated payload history.
- Analytical checks should not impact live operational workloads.

Core principle:

> Replica databases are suitable for recent operational checks; S3/Parquet/Athena is suitable for historical, aggregate and analytical assurance checks.

## 3. Proposed Analytical Data Flow

```text
   +--------------------+
   | Source Events /    |
   | Operational Systems|
   +----------+---------+
              |
              v
   +----------+---------+
   | Ingestion /        |
   | Processing Pipeline|
   +----------+---------+
              |
              v
   +----------+---------+        +----------------------+
   | S3 Raw Zone        |------->| Schema Observation / |
   | source-aligned     |        | Drift Snapshots      |
   +----------+---------+        +----------------------+
              |
              v
   +----------+---------+
   | S3 Curated Zone    |
   | Parquet datasets   |
   | partitioned        |
   +----------+---------+
              |
              v
   +----------+---------+
   | Glue Crawler /     |
   | Metadata Workflow  |
   +----------+---------+
              |
              v
   +----------+---------+
   | Glue Data Catalog  |
   | tables, columns,   |
   | partitions         |
   +----------+---------+
              |
              v
   +----------+---------+
   | Athena             |
   | partition-aware    |
   | analytical queries |
   +----------+---------+
              |
              v
   +----------+---------+
   | Data Assurance     |
   | Rule Executor      |
   | reconciliation,    |
   | backtesting, drift |
   +----------+---------+
              |
              v
   +----------+---------+      +-----------------------+
   | Findings Store     |----->| Review UI             |
   +----------+---------+      +----------+------------+
              |                           |
              v                           v
   +----------+---------+      +----------+------------+
   | Learning /         |<-----| Feedback Store        |
   | Backtesting Engine |      +-----------------------+
   +----------+---------+
              |
              v
   +----------+---------+
   | Agent / Copilot    |
   | Summary Layer      |
   | controlled outputs |
   +--------------------+
```

Flow summary:

```text
Source/Event data
  -> S3 raw and curated Parquet
  -> Glue Data Catalog
  -> Athena
  -> DQ rules, backtesting, reconciliation
  -> findings
  -> review UI
  -> feedback
  -> learning loop
```

## 4. Data Lake Zones

Not all zones are required for the PoC. The model helps separate concerns and access patterns.

### 4.1 Raw Zone

The raw zone stores immutable, source-aligned payloads with minimal transformation.

Characteristics:

- Useful for investigation and schema drift analysis.
- Access tightly controlled.
- May contain sensitive data.
- Should not usually be queried directly by general DQ rules.
- Useful as an evidence source when curated data is insufficient.

### 4.2 Curated Zone

The curated zone stores cleaned and standardised datasets suitable for analytical assurance checks.

Characteristics:

- Parquet format.
- Partitioned by date, hour, source, domain, and event type where appropriate.
- Suitable for Athena DQ checks.
- Masking or minimisation applied where possible.
- Preferred source for repeatable analytical rules.

### 4.3 Assurance Metrics Zone

The assurance metrics zone stores precomputed aggregates for cheap and fast checks.

Examples:

- Hourly and daily counts.
- Missing field rates.
- Duplicate rates.
- Freshness snapshots.
- Reconciliation counts.
- Baseline metrics.

This zone reduces repeated expensive scans over raw or curated event data.

### 4.4 Backtesting Zone

The backtesting zone stores historical rule execution outputs, candidate rule simulation results, shadow-mode findings, and backtest reports.

It should not be used for live alerting. Its purpose is governance, evaluation, and rule refinement.

### 4.5 Audit / Evidence Zone

This optional zone stores evidence references rather than unrestricted raw evidence.

Characteristics:

- Masked samples only.
- Evidence hashes.
- Immutable references.
- Retention-controlled.
- Linkable from findings and audit records.

## 5. Parquet Design Principles

Parquet is useful for analytical assurance because it is columnar and efficient for aggregate queries.

Benefits:

- Lower Athena scan cost compared with raw JSON or CSV for many analytical workloads.
- Efficient column pruning.
- Good compression support.
- Better fit for large-scale analytical queries.
- Works well with partition pruning.

File sizing principles:

- Avoid too many tiny files.
- Avoid huge unmanageable files.
- Target reasonably sized Parquet files aligned to query and compaction patterns.
- Compact small files where required.

Schema evolution principles:

- Control schema changes.
- Prefer backward-compatible changes.
- Define a Glue Data Catalog update strategy.
- Avoid uncontrolled crawler-driven surprises in production.

Concrete process: additive, backward-compatible changes (new optional columns) are allowed and
versioned in the Glue table definition; breaking changes (type changes, removals, semantic changes)
require a new table version or a new partition/dataset rather than mutating in place. Schema changes
to critical curated datasets go through the controlled Glue table definition (IaC or approved metadata
workflow), with a compatibility check in CI before promotion. Observed-vs-expected schema drift is
detected and reviewed (see the schema drift rule), not silently absorbed by a crawler.

Raw payloads may still be retained separately for investigation and audit. Parquet should be the preferred analytical format, not necessarily the only retained format.

## 6. Partitioning Strategy

Partitioning should match common assurance query patterns.

Possible partitions:

- `year`
- `month`
- `day`
- `hour`
- `source_system`
- `event_type`
- `domain`

Trade-offs:

- Date and hour partitions support time-window checks.
- `source_system` supports source-specific rules.
- `event_type` supports targeted checks.
- Too many partitions can create metadata overhead.
- Partition strategy should be reviewed against expected query patterns.

Example path:

```text
s3://cerberos-curated/events/domain=identity/source_system=CarrierGateway-A/event_type=JOURNEY_UPDATE/year=2026/month=06/day=15/hour=10/
```

Partitioning should support common DQ queries such as:

- Last hour missing field rate.
- Daily source count.
- Inbound vs processed reconciliation.
- Event type trend.
- Schema drift by source.
- Historical backtesting over 7, 28, or 90 days.

## 7. Glue Crawler and Data Catalog

AWS Glue Crawler can discover schemas from S3 datasets. Glue Data Catalog stores metadata such as databases, tables, columns, and partitions. Athena uses Data Catalog metadata to query S3 datasets.

In this architecture, Glue Data Catalog acts as the metadata registry for analytical assurance datasets.

Important nuance: Glue Crawler is useful for discovery and early development, but production schema management may require more control.

### 7.1 Crawler-driven Metadata

Pros:

- Fast discovery.
- Useful during PoC.
- Less manual setup.

Cons:

- Unexpected schema changes.
- Less controlled for critical systems.
- May hide schema drift if not governed carefully.

### 7.2 Infrastructure-managed Catalog

Pros:

- Explicit schema ownership.
- Versioned table definitions.
- Better governance.
- Safer for production.

Cons:

- More setup.
- Requires a schema management process.

Recommendation:

- PoC: crawler may be acceptable for selected datasets.
- Platform: prefer controlled Glue table definitions via IaC or an approved metadata workflow for critical curated datasets.

## 8. Athena Usage Patterns

Good Athena use cases:

- Aggregate checks.
- Historical analysis.
- Partitioned queries.
- Reconciliation.
- Backtesting.
- Schema drift analytics.
- Baseline generation.
- Daily and weekly assurance reports.

Poor Athena use cases:

- Low-latency per-user online request path.
- Unbounded raw data scans.
- Uncontrolled ad hoc queries.
- Frequent small transactional lookups.
- Production operational decisioning.

Athena queries should be:

- Parameterised.
- Approved.
- Partition-aware.
- Bounded by time window.
- Cost-limited.
- Audited.
- Executed through `SafeQueryExecutor`.
- Never directly generated and executed by an agent.

The worked Athena query examples (missing field rate, source volume baseline, reconciliation, schema drift, duplicates, backtesting), `SafeQueryExecutor` integration, reconciliation architecture, backtesting and rule simulation, schema drift architecture, and the agent/Copilot relationship are captured in the child page **Athena Analytical DQ Examples and Sub-Architectures** to keep this page focused on concept, governance, cost, and scope.

## 9. Security and Governance

Controls specific to S3, Athena, and Glue:

- IAM least privilege.
- Bucket policies.
- KMS encryption.
- Lake Formation if applicable.
- Glue Data Catalog permissions.
- Athena workgroups.
- Query result location controls.
- CloudTrail audit.
- S3 access logging.
- Partition-level access where needed.
- Sensitive zone separation.
- Masked curated datasets.
- No raw PII in notifications or agent prompts.
- Athena workgroup query limits.
- Approved query templates.
- Audit link from finding to Athena query execution ID.

Athena workgroups can help enforce controls such as query result locations, encryption, and cost-related limits.

## 10. Cost and Performance Controls

Athena cost is based on scanned data. Parquet and partitioning are therefore central design controls.

Controls:

- Avoid `SELECT *`.
- Query only required columns.
- Enforce partition filters.
- Limit historical windows.
- Use precomputed assurance metrics.
- Compact small files.
- Cache or snapshot repeated aggregates.
- Monitor query cost by rule.
- Set workgroup limits.
- Fail queries that exceed expected cost threshold.

Rule-level cost metadata:

```yaml
expected_scan_mb: 500
max_scan_mb: 2048
athena_workgroup: dq_assurance
requires_partition_filter: true
```

## 11. PoC Scope for Analytical Layer

For the canonical overall PoC scope, see **PoC, Roadmap, and Risks**. The scope below is only the analytical-layer subset.

Suggested PoC scope:

- One curated Parquet dataset.
- One Glue table.
- One Athena workgroup.
- Two Athena DQ rules.
- One reconciliation query.
- One backtesting example.
- One schema drift example if feasible.
- Findings stored in PostgreSQL.
- Reviewed through the same UI.
- No raw PII.
- No direct agent access to Athena.

Candidate PoC rules:

1. Source volume baseline check.
2. Missing field rate by source.
3. Inbound vs processed reconciliation.
4. Candidate rule backtest over 7 or 28 days.

These analytical rules are not a separate PoC. They are the Athena/S3 expression of the canonical
PoC scope in **PoC, Roadmap, and Risks**: "Missing field rate by source" is the analytical form of
the canonical *missing mandatory identity field* rule, and "Inbound vs processed reconciliation" is
the same canonical reconciliation rule run over curated Parquet. "Source volume baseline" and
"Candidate rule backtest" are analytical-layer additions that exercise historical, partition-aware
queries. The canonical replica-DB rules (freshness, duplicate external reference, broken
journey-person reference) remain defined in **PoC, Roadmap, and Risks**.

Rule count clarification: the PoC remains **five canonical rules**. A rule executed against both a
replica and an Athena dataset is the *same rule with two source variants*, not two rules. "Source
volume baseline" and "Candidate rule backtest" are not additional production rules; the baseline is
the analytical form of the canonical simple-anomaly rule, and the backtest is an evaluation activity,
not a standing rule. So the analytical layer does not increase the five-rule PoC count.

Success criteria:

- Athena queries are bounded and partition-aware.
- Findings are created from analytical checks.
- Scan cost is measured.
- Backtest report can be produced.
- No operational DB impact occurs.
- Results are actionable.

## 12. Technology Decisions / ADR Additions

Full ADR text should live in **Architecture Decision Records and Reference Architecture** to avoid duplicated decisions drifting across pages.

This page contributes the following ADR additions:

- **ADR-009:** Use S3/Parquet for historical analytical assurance datasets.
- **ADR-010:** Use Glue Data Catalog as metadata layer for Athena-accessible datasets.
- **ADR-011:** Use Athena for partitioned analytical DQ checks and backtesting.
- **ADR-012:** Require all Athena access through `SafeQueryExecutor`.
- **ADR-013:** Use controlled Glue table definitions for critical curated datasets.
- **ADR-014:** Keep raw and curated S3 zones separate.

## 13. Risks and Mitigations

| Risk | Mitigation |
| --- | --- |
| Athena cost surprises | Use Parquet, partition filters, workgroup limits, cost estimates, and cost dashboards. |
| Slow queries | Use partition pruning, precomputed metrics, bounded windows, and compacted files. |
| Too many small files | Add compaction process and file sizing standards. |
| Poor partitioning | Design partitions around DQ query patterns and review usage. |
| Uncontrolled schema changes | Use controlled table definitions for critical datasets and review crawler output. |
| Glue Crawler masking schema drift | Store observed schema snapshots and compare against expected schema. |
| Raw PII exposure | Separate raw and curated zones, mask curated datasets, restrict access, and minimise outputs. |
| Agents generating arbitrary SQL | Do not allow direct agent SQL execution; agents consume controlled outputs only. |
| Analytical findings lag operational reality | Use replica DB checks for recent operational needs and Athena for historical assurance. |
| Duplicate checks between replica and Athena | Define rule ownership and access pattern guidance. |
| Unclear curated dataset ownership | Assign dataset owners and metadata governance responsibilities. |

## 14. Final Recommendation

Use replica databases for recent operational read-only checks. Use S3, Parquet, Glue Data Catalog, and Athena for historical, aggregate, reconciliation, schema drift, trend analysis, and backtesting use cases.

Do not treat Athena as a low-latency online query engine. Athena should be used for approved, partition-aware analytical assurance queries executed through `SafeQueryExecutor`.

Use Glue Data Catalog as the metadata layer, but govern schema changes carefully. Glue Crawler may be acceptable for PoC discovery, while critical curated datasets should move toward controlled table definitions.

Use Parquet, partitioning, precomputed assurance metrics, and Athena workgroup controls to manage cost and performance. Feed Athena-generated findings into the same Review UI, Feedback Store, and Learning Loop as other rule findings.

This analytical layer strengthens the overall Cerberos Data Assurance Intelligence Layer by enabling historical context, scalable aggregate checks, reconciliation, backtesting, schema drift analysis, and trend-based learning without loading operational systems.
