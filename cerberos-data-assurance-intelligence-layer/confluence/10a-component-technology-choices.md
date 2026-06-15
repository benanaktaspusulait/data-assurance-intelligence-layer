# Component Technology Choices

**Parent page:** Cerberos Data Assurance Intelligence Layer - Initial Discovery Concept  
**Status:** Private working draft  
**Purpose:** Per-component technology options and recommendations (backend, registry, query execution, sources, stores, UI, learning, agent, AWS, scheduling, observability, security, deployment)
**Companion page:** See **Technology Selection and Architecture Decision Report** for the executive summary, selection criteria, candidate PoC stack, risks, and final recommendation.

## 1. Backend Technology Choice

| Option | Strengths | Concerns | Fit |
| --- | --- | --- | --- |
| Java 21 + Quarkus | Lightweight JVM service, strong container fit, good OpenTelemetry support, good JDBC and AWS SDK fit, suitable for scheduled jobs and REST APIs. | Team must be comfortable with Quarkus if not already used. | Candidate PoC option if it aligns with platform standards. |
| Kotlin + Ktor | Lightweight, expressive, good fit for Kotlin-heavy teams, simple service design. | Less conventional in some enterprise Java estates; more custom integration choices. | Good if Cerberos is Kotlin-heavy or Koog alignment is important. |
| Spring Boot | Highly mature enterprise framework, strong ecosystem, Spring AI integration, familiar operations. | Can be heavier than needed for a small PoC if the estate is not already Spring-based. | Best alternative if Cerberos already uses Spring Boot. |
| Python FastAPI | Fast prototyping, strong data/ML ecosystem. | Weaker enterprise JVM alignment, more risk around long-term platform fit if Cerberos is JVM-based. | Useful for analytics side jobs, not the main backend candidate if Cerberos is JVM-based. |
| Node.js/NestJS | Productive API/UI integration and good web ecosystem. | Less natural for JDBC-heavy governed query execution in JVM enterprise context. | Not recommended as primary backend for this platform. |

Java 21 + Quarkus is a candidate PoC option if it aligns with existing platform standards. Spring Boot, Kotlin/Ktor, or another existing JVM standard may be more appropriate depending on the current Cerberos estate.

Acceptable alternative: **Spring Boot** if Cerberos is already Spring Boot-based and team/platform standards favour it.

Kotlin/Ktor is a good option if Kotlin alignment or future Koog orchestration becomes a major driver, but it should be selected based on team and platform fit rather than novelty.

**Accreditation note.** In the OFFICIAL-SENSITIVE border-security context, **Spring Boot is the preferred default** for accreditation familiarity (see Amber flag #8 in **Border-Security Constraints and Pre-Funding Conditions**). Quarkus remains a candidate only where the team already has Quarkus experience and accreditation is not blocked; it should not be chosen for lightweight-runtime reasons alone. The backend decision must be confirmed before the PoC.

## 2. Rule Definition and Rule Registry

| Option | Strengths | Concerns | Recommendation |
| --- | --- | --- | --- |
| YAML files | Readable, reviewable, Git-friendly, easy for PoC. | Needs validation and approval workflow. | Recommended for PoC. |
| JSON files | Easy schema validation and machine processing. | Less readable for complex SQL and governance metadata. | Acceptable alternative. |
| Database-backed rules | Strong lifecycle, approvals, audit, UI management. | More build effort early. | Recommended for platform phase. |
| Drools | Powerful rules engine. | Likely too complex for SQL-driven DQ checks. | Not for PoC. |
| Custom DSL | Can encode domain semantics. | Risk of building a language too early. | Defer until rule complexity justifies it. |
| OPA/Rego | Strong policy model. | Better for policy decisions than SQL DQ rules. | Consider for access/governance policy, not rule execution initially. |
| dbt / Great Expectations style rules | Existing data quality patterns. | May not fit human feedback learning and review workflow directly. | Evaluate later for execution layer reuse. |

Recommended approach:

- PoC: Git-backed YAML rules.
- Platform: database-backed registry with versioning, approval workflow, and audit trail.
- Future: optional DSL or policy engine only if the rule model becomes too complex for YAML.

Sample YAML rule (canonical flat format, validated by the JSON Schema in **Supporting Technical Detail**):

```yaml
id: CERB-DQ-001
version: 1
name: Missing critical identity fields
domain: passenger-identity
description: Detects missing document number, nationality or date of birth by source system.
severity: HIGH
source_type: replica_db
source_name: passenger_replica
schedule: every_15_minutes
query_type: aggregate
owner_team: ingestion-platform-team
pii_level: sensitive
query: |
  SELECT source_system,
         COUNT(*) AS total_count,
         SUM(CASE WHEN document_number IS NULL THEN 1 ELSE 0 END) AS missing_document_count,
         SUM(CASE WHEN nationality IS NULL THEN 1 ELSE 0 END) AS missing_nationality_count,
         SUM(CASE WHEN date_of_birth IS NULL THEN 1 ELSE 0 END) AS missing_dob_count
  FROM passenger_identity
  WHERE created_at >= now() - interval '15 minutes'
  GROUP BY source_system;
threshold:
  expression: missing_document_count > 0 OR missing_nationality_count > 0 OR missing_dob_count > 0
result_limits:
  max_rows: 1000
  timeout_seconds: 30
governance:
  approved_by: TBD
  requires_masking: true
  allow_raw_samples: false
actions:
  create_finding: true
  notify: false
```

## 3. Query Execution Technology

Safe query execution must be a governed platform capability, not direct database access from rule code.

Required features:

- SELECT-only enforcement.
- Allow-listed data sources.
- Approved query templates.
- Parameterisation.
- Timeouts.
- Row limits.
- Athena cost limits.
- Concurrency limits.
- Query audit.
- Result masking.
- Error isolation.
- Query explain/validation where possible.
- Execution metadata capture.
- Circuit breaker per data source (fail fast when a source's error rate is high; see **Logging, Observability, and Monitoring**).

| Option | Strengths | Concerns | Recommendation |
| --- | --- | --- | --- |
| Direct JDBC | Simple, mature, portable for replica DBs. | Easy to misuse without a wrapper. | Use only behind `SafeQueryExecutor`. |
| jOOQ | Type-safe SQL building, good control over SQL. | Adds library complexity; generated schema may be awkward for multiple sources. | Useful for controlled SQL and metadata queries. |
| Hibernate/JPA | Familiar ORM for application data. | Poor fit for arbitrary aggregate DQ queries and query guardrails. | Avoid for rule execution. |
| Hibernate Native SQL | Allows SQL execution through an existing Hibernate/EntityManager stack while reusing connection management and transactions. | Still tied to ORM/session semantics, weaker fit for cross-source execution, Athena, partition/cost controls, and query governance. Native SQL does not by itself solve masking, allow-listing, audit, or result bounding. | Acceptable only behind `SafeQueryExecutor` for fixed replica DB checks if Hibernate is already the platform standard. Not the core execution abstraction. |
| Hibernate Reactive | Non-blocking database access for reactive JVM services where supported drivers and runtime are already standard. | Limited source coverage, no fit for Athena SDK, more complexity for scheduled/batch assurance jobs, and does not address query safety, masking, cost control, or audit. | Defer. Consider only if Cerberos is already reactive end-to-end and the source connector supports it, still behind `SafeQueryExecutor`. |
| Apache Calcite | Query planning and SQL abstraction. | Likely overkill for PoC. | Consider only if multi-source SQL abstraction becomes necessary. |
| Presto/Trino | Powerful distributed query engine. | Platform dependency and operational overhead. | Consider later if already available. |
| Athena SDK | Native AWS path for S3/Parquet analytical checks. | Needs cost and timeout controls. | Recommended for Athena checks. |
| Custom executor | Centralises governance. | Must be designed carefully. | Required abstraction. |

Why Hibernate is not the default query execution technology:

- The assurance layer is not primarily persisting application entities; it is running approved aggregate, reconciliation, freshness, schema drift, and backtesting checks.
- Rules need source-aware controls across replica databases and Athena, not only ORM-managed relational tables.
- Governance controls must live outside the ORM: approved templates, allow-listed sources, SELECT-only enforcement, row limits, masking, audit, and Athena scan controls.
- Reactive database access improves a specific runtime model, but it does not reduce the need for query governance and may add complexity for scheduled PoC jobs.

Interface example:

```java
public interface SafeQueryExecutor {
    QueryResult execute(ApprovedRule rule, QueryParameters parameters);
}
```

All query execution must go through this layer. No UI, scheduler, agent, or rule service should query data sources directly.

## 4. Data Sources and Connectors

Target source types:

- Replica PostgreSQL, Oracle, or SQL Server databases.
- Athena over S3/Parquet.
- S3 raw and curated zones.
- Glue Data Catalog.
- Observability metrics such as Prometheus or Grafana APIs.
- Kafka lag metrics.
- Reporting datasets.

Connector abstraction:

```java
public interface DataSourceConnector {
    SourceType type();
    QueryResult execute(QueryRequest request);
    HealthStatus health();
}
```

Why connectors should be isolated:

- Source credentials differ.
- Query safety rules may differ by source.
- Cost model differs between replica DBs and Athena.
- Health checks differ.
- Result masking may depend on source metadata.
- Audit metadata must identify the connector and source.

PoC should implement one replica DB connector and optionally one Athena connector. Additional connectors should be added only when justified by PoC value.

## 5. Findings and Feedback Store

| Option | Strengths | Concerns | Recommendation |
| --- | --- | --- | --- |
| PostgreSQL | Relational modelling, transactions, audit-friendly, easy querying, strong PoC fit. | Needs scaling plan if write volume becomes very high. | Recommended for PoC. |
| DynamoDB | Serverless scale and key-value access. | Less natural for relational review workflows and ad hoc analysis. | Defer unless scale/key-value access requires it. |
| OpenSearch | Search and analytics. | Not ideal as source of truth for workflow state. | Optional later for search/reporting. |
| Elasticsearch | Similar search benefits. | Operational and governance complexity. | Optional later if already standard. |
| S3 | Low-cost archival. | Not suitable for transactional workflow. | Use for archival only. |
| Graph database | Can model relationships. | Overkill for PoC. | Not recommended initially. |

Recommendation:

- PoC and early platform: PostgreSQL as primary store for findings, feedback, rule metadata, and audit events.
- Later: OpenSearch for search/analytics if needed.
- Later: S3 for long-term archival.
- DynamoDB only if serverless/high-scale key-value workflow becomes necessary.

Conceptual schema (authoritative full definitions in **Rule Types, Data Model, and Examples**):

- `dq_rule`
- `dq_rule_run`
- `dq_finding`
- `dq_feedback`
- `dq_feedback_pattern`
- `dq_rule_suggestion`
- `dq_exception`
- `dq_agent_analysis`
- `dq_metric_snapshot`
- `dq_audit_event`

## 6. Review UI Technology

| Option | Strengths | Concerns | Recommendation |
| --- | --- | --- | --- |
| React | Fast to build, flexible, widely understood. | Requires frontend ownership and design discipline. | Recommended if no existing UI standard blocks it. |
| Angular | Enterprise-friendly, structured. | Heavier for PoC unless already standard. | Use if existing Cerberos frontend standard is Angular. |
| Server-rendered UI | Simple and low overhead. | May limit richer review workflows. | Acceptable for very small PoC. |
| Internal admin UI framework | Fast and consistent if available. | May constrain UX. | Prefer if enterprise-approved. |
| Low-code/internal tool | Very fast PoC. | Governance, extensibility, and data access constraints. | Consider only if approved and controlled. |

UI must support:

- Finding review.
- Confirm/reject/reclassify.
- Comment.
- Corrected severity.
- Corrected owner.
- Known exception flag.
- Link to incident.
- Rule suggestion review.

Recommendation:

- PoC: simple React or existing internal UI pattern.
- Platform: align with existing enterprise frontend standards.
- If UI investment is not possible initially: provide APIs plus a minimal admin page.

The remaining platform-service component choices (learning and insight engine, agent/Copilot
integration, AWS services, scheduling and orchestration, observability and audit, security
architecture, and deployment) are captured in the child page
**Component Technology Choices - Platform Services** to keep this page at a readable length.
