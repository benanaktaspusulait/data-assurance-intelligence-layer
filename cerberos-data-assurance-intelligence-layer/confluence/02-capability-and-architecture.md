# Capability and Architecture

**Parent page:** Cerberos Data Assurance Intelligence Layer - Initial Discovery Concept  
**Status:** Private working draft

## Proposed Capability

The Cerberos Data Assurance Intelligence Layer would run approved, controlled, read-only data quality checks against safe data sources. It would create findings, present them for human review, capture structured feedback, and use that feedback to suggest improvements over time.

It should be positioned as a platform assurance capability, not as autonomous AI.

## Core Components

- **Data Quality Rule Registry:** Versioned catalogue of approved rules, schedules, thresholds, sources, owners, PII level, and approval status.
- **Scheduler:** Triggers approved rules at defined intervals.
- **Safe Query Executor:** Runs approved query templates with guardrails, timeouts, result limits, cost limits, and audit logging.
- **Read-only Connectors:** Connects to read replicas, analytical stores, S3/Parquet datasets, Athena tables, reporting datasets, and metrics sources.
- **Findings Store:** Stores normalised findings, evidence references, severity, confidence, affected counts, and status.
- **Review UI:** Enables users and domain experts to validate findings and provide structured feedback.
- **Feedback Store:** Captures decisions, comments, corrections, reason codes, exceptions, and routing changes.
- **Learning and Insight Engine:** Analyses feedback patterns to identify refinements and hidden data quality patterns.
- **Rule Suggestion Engine:** Suggests candidate rule refinements, exceptions, and new rules for human approval.
- **Dashboard and Notifications:** Provides reporting and routes findings to owners.
- **Governance and Audit Layer:** Tracks approvals, rule versions, access, execution, and feedback.

## High-Level Architecture

```text
                       +-----------------------------+
                       | Data Quality Rule Registry  |
                       +--------------+--------------+
                                      |
                                      v
                               +------+------+
                               | Scheduler   |
                               +------+------+
                                      |
                                      v
                        +-------------+-------------+
                        | SafeQueryExecutor         |
                        | approved templates,       |
                        | limits, audit, masking    |
                        +------+--------------------+
                               |
              +----------------+----------------+
              |                                 |
              v                                 v
      +-------+--------+              +---------+---------+
      | Replica DB     |              | Athena Connector  |
      | read-only      |              | analytical checks |
      +-------+--------+              +---------+---------+
              |                                 |
              |                                 v
              |                       +---------+---------+
              |                       | Athena            |
              |                       +---------+---------+
              |                                 |
              |                                 v
              |                       +---------+---------+
              |                       | Glue Data Catalog |
              |                       +---------+---------+
              |                                 ^
              |                                 |
              |                       +---------+---------+
              |                       | Glue Crawler /    |
              |                       | metadata workflow |
              |                       +---------+---------+
              |                                 ^
              |                                 |
              |        +------------------------+-------------------+
              |        |                                            |
              |        v                                            v
              | +------+-------------+                    +---------+---------+
              | | S3 raw zone        |                    | S3 curated        |
              | | source-aligned     |                    | Parquet zone      |
              | +--------------------+                    +-------------------+
              |
              +----------------+----------------+
                               |
                               v
                        +------+------+
                        | Findings    |
                        | Store       |
                        +------+------+
                               |
                               v
                        +------+------+
                        | Review UI   |
                        +------+------+
                               |
                               v
                        +------+------+
                        | Feedback    |
                        | Store       |
                        +------+------+
                               |
                               v
                    +----------+-----------+
                    | Learning Loop        |
                    | confidence,          |
                    | backtesting,         |
                    | suggestions          |
                    +----------+-----------+
                               |
                               v
                    +----------+-----------+
                    | Human Approval       |
                    | updated rules only   |
                    +----------------------+
```

Replica DB checks and Athena analytical checks are complementary. Replica DB checks are better suited to recent, near-operational read-only checks. Athena is better suited to analytical, aggregate, historical, partition-aware checks such as reconciliation, trend analysis, schema drift, and backtesting. Athena must not be used as a low-latency user request path.

All Athena access must go through `SafeQueryExecutor`. Agents or Copilot-style assistants must never directly query Athena or S3; they may only analyse controlled, masked, bounded outputs from approved rule executions.

## End-to-End Flow

1. A rule is defined and approved.
2. The scheduler triggers the rule.
3. The Safe Query Executor validates and runs the query against a read-only source.
4. Results are normalised.
5. Thresholds are evaluated.
6. Findings are created.
7. Findings are displayed in the Review UI.
8. Users review and provide structured feedback.
9. Feedback is stored.
10. The learning engine analyses feedback.
11. The system updates confidence scores.
12. The system suggests exceptions, refinements, evidence changes, routing changes, or new rules.
13. A human owner reviews suggestions.
14. Approved changes are versioned and backtested.
15. Updated rules are deployed through governance.

Automatic rule deployment should not happen without approval.

## Initial Rule Categories

Initial rule categories include completeness, freshness, duplicate detection, referential integrity, status validation, reconciliation, schema drift, anomaly detection, cross-system consistency, and data contract validation.

Detailed examples are captured in the child page **Rule Types, Data Model, and Examples**.

Technology options for the optional agent layer are captured in the child page **JVM Agent Framework Options**.

Operational visibility for the assurance platform itself is captured in the child page **Logging, Observability, and Monitoring**.

Implementation technology choices and initial ADRs are captured in the child page **Technology Selection and Architecture Decision Report**.

The analytical data lake access pattern for S3, Parquet, Glue Data Catalog, and Athena is captured in the child page **S3, Parquet, Glue Data Catalog and Athena Analytical Assurance Layer**.
