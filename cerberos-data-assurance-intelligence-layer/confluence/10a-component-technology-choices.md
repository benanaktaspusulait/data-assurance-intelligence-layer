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

## 7. Learning and Insight Engine

The first version should not require model fine-tuning or complex ML.

Learning can start deterministically:

- Approval/rejection ratios.
- Rule confidence scoring.
- False positive trends.
- Owner correction patterns.
- Severity correction patterns.
- Known exception detection.
- Repeated confirmed patterns.
- Clustering by source, event, domain, rule, and time window.

Technology options:

- SQL analytics in PostgreSQL.
- Scheduled analysis jobs.
- Java/Kotlin backend services.
- Python analytics jobs.
- Spark/Glue jobs for larger historical analysis.
- OpenSearch aggregations.
- ML later if needed.

Recommendation:

- PoC: deterministic feedback analytics in the backend service using PostgreSQL queries.
- Later: separate Learning Engine service if the capability grows.
- Future: optional ML/anomaly detection using historical baselines.

LLM/agent reasoning should consume structured summaries, not raw sensitive data.

## 8. Agent / Copilot Integration Technology

Options:

- Microsoft Copilot integration.
- Azure OpenAI or OpenAI-compatible API.
- AWS Bedrock.
- Internal approved LLM gateway.
- Koog for JVM/Kotlin agent orchestration.
- LangChain4j for Java LLM integration.
- Spring AI if Spring Boot is standard.
- No LLM in PoC.

All agent framework references should be validated during discovery before being treated as architecture recommendations.

Important principle: the architecture must not assume unrestricted LLM access.

Recommendation:

- PoC can work without LLM.
- Add an `AgentAnalysisService` interface for future integration.
- Use approved enterprise AI/Copilot capability only if governance allows.
- Agent receives controlled, masked, structured finding summaries.
- Agent outputs summaries, root cause hypotheses, and suggested rule refinements.
- Human review is required.

Interface example:

```java
public interface AgentAnalysisService {
    AgentAnalysis analyseFinding(FindingContext context);
    RuleSuggestion suggestRule(FeedbackPattern pattern);
}
```

Koog can be considered later for JVM/Kotlin agent orchestration, particularly if graph-based learning workflows become central. It should not be required for the first PoC.

Agents must not directly query Athena or S3 and must not generate arbitrary SQL for execution. The correct model is approved rule execution through `SafeQueryExecutor`, followed by controlled, masked, bounded results that may be summarised by an agent for human review.

## 9. AWS Technology Choices

| AWS Service | Likely Usage | Notes |
| --- | --- | --- |
| S3 | Historical and curated datasets. | Use partitioned Parquet where possible. |
| Parquet | Efficient analytical storage format. | Helps reduce scan cost and improve Athena performance. |
| Glue Data Catalog | Schema metadata for S3/Athena datasets. | Athena can use Glue Data Catalog metadata for S3 tables. |
| Glue Crawler | Optional metadata discovery. | Useful for PoC/discovery; critical curated schemas should move toward controlled IaC-managed table definitions. |
| Athena | Analytical DQ checks over S3/Parquet. | For aggregate, historical, reconciliation, schema drift, and backtesting checks; not a low-latency request path. |
| IAM | Least privilege access. | Separate roles per source/service where practical. |
| CloudTrail | AWS API audit. | Complements application audit events. |
| Secrets Manager | Source credentials and secrets. | Prefer over embedded secrets. |
| CloudWatch | Logs, metrics, alarms. | Useful baseline if AWS-native. |
| EventBridge | Scheduling or event triggers. | Optional for platform phase. |
| Step Functions | Workflow orchestration. | Useful for complex rule workflows later. |
| Lambda | Lightweight event handlers. | Avoid forcing core rule execution into Lambda if jobs are long-running. |
| ECS/EKS | Container runtime. | Choose based on enterprise platform standards. |
| RDS PostgreSQL | Findings and feedback store. | Strong fit for PoC. |
| DynamoDB | Optional high-scale key-value store. | Defer unless needed. |
| OpenSearch | Optional search/analytics layer. | Not primary source of truth. |

Managed AWS services reduce operational burden, but they can also increase platform coupling. The recommendation is to use AWS services where they naturally fit data access, security, and operations, while keeping core domain logic in portable service boundaries.

Replica DB checks and Athena analytical checks are complementary. Replica DBs are better for recent operational read-only checks. Athena over partitioned Parquet datasets is better for historical, aggregate, reconciliation, schema drift, trend analysis, and backtesting use cases. All Athena access should be routed through `SafeQueryExecutor`.

## 10. Scheduling and Orchestration

| Option | Strengths | Concerns | Recommendation |
| --- | --- | --- | --- |
| In-service scheduler | Simple and fast for PoC. | Needs care for clustering and failover. | Good PoC option. |
| Quartz | Mature scheduling model. | More configuration and state management. | Good if persistent schedules are needed. |
| Kubernetes CronJob | Simple operational model on Kubernetes. | Less expressive for dependencies and retries. | Good PoC option if platform is Kubernetes. |
| AWS EventBridge | Managed scheduling/eventing. | AWS coupling. | Good if AWS-native operations are standard. |
| Step Functions | Explicit workflows and retries. | More orchestration overhead. | Good later for complex flows. |
| Airflow | Strong DAG orchestration. | Operational overhead. | Consider for complex data workflows. |
| Dagster/Prefect | Modern data orchestration. | Additional platform dependency. | Consider only if already adopted. |
| AWS Glue Jobs | Managed data processing. | Less natural for review/feedback workflow. | Useful for large historical analysis. |

Recommendation:

- PoC: in-service scheduler or Kubernetes CronJob.
- Platform: use existing enterprise scheduler/orchestrator standard.
- Complex DAGs and reconciliation dependencies: consider Airflow, Dagster, or Step Functions later.

Scheduling must support retries, concurrency limits, dependency handling, backfill, audit, and operational visibility.

## 11. Observability and Audit

Recommended observability:

- OpenTelemetry traces.
- Structured JSON logs.
- Metrics for rule execution.
- Query execution logs.
- Data source health checks.
- Dashboard metrics.

Key metrics:

- Rule execution count.
- Failed rule runs.
- Average execution time.
- Findings generated.
- Findings confirmed/rejected.
- False positive rate.
- Rule confidence score.
- Pending review count.
- Alert volume.
- Query timeout count.
- Athena cost estimate.
- Data source health.

Audit events:

- Rule created.
- Rule approved.
- Rule executed.
- Finding generated.
- Feedback submitted.
- Rule suggestion created.
- Rule suggestion approved/rejected.
- Exception created.
- Notification sent.

Audit and observability should be treated separately. Observability shows whether the system is healthy. Audit shows who did what, when, and why. Detailed observability strategy is in **Logging, Observability, and Monitoring**.

## 12. Security Architecture

Security controls:

- IAM/RBAC.
- Service accounts.
- Read-only DB users.
- Separate credentials per source.
- No shared superuser.
- Network restrictions.
- Secrets management.
- PII masking layer.
- Output minimisation.
- Query validation.
- SQL deny-list.
- Allow-listed query templates.
- Approval workflow.
- Audit trail.
- Encryption at rest and in transit.
- Least privilege.
- Data retention policy.
- Access review.

AI-specific controls:

- No raw PII to LLM.
- Prompt and output audit.
- Approved AI endpoint only.
- No training on sensitive data unless explicitly approved.
- Human review.
- No autonomous actions.

## 13. Deployment Architecture

Deployment options:

- Kubernetes/OpenShift.
- EKS.
- ECS/Fargate.
- VM-based deployment.
- Serverless Lambda.

Recommendation:

- Use a containerised service on the existing enterprise container platform if available.
- Use RDS PostgreSQL for the findings/feedback store.
- Use managed AWS integrations where they reduce operational burden.
- Avoid serverless-only design for core rule execution unless rule durations and platform constraints are well understood.

Simple deployment diagram:

```text
                  +----------------------------+
                  | Enterprise Identity / IAM  |
                  +-------------+--------------+
                                |
                                v
         +----------------------+----------------------+
         | Container Platform / Kubernetes / OpenShift |
         |                                             |
         |  +----------------+    +----------------+   |
         |  | DQ Backend     |    | Review UI      |   |
         |  | Quarkus        |<-->| React/Admin UI |   |
         |  +-------+--------+    +----------------+   |
         |          |                                  |
         |          v                                  |
         |  +-------+--------+                         |
         |  | SafeQuery      |                         |
         |  | Executor       |                         |
         |  +-------+--------+                         |
         +----------+----------------------------------+
                    |
      +-------------+--------------+
      |                            |
      v                            v
+-----+------+             +-------+--------+
| Replica DB |             | Athena / S3 /  |
| Read Only  |             | Glue Catalog   |
+------------+             +----------------+
                                ^
                                |
                     +----------+-----------+
                     | Glue Crawler /       |
                     | metadata workflow    |
                     +----------+-----------+
                                ^
                                |
              +-----------------+----------------+
              |                                  |
              v                                  v
       +------+--------+                 +-------+--------+
       | S3 raw zone   |                 | S3 curated     |
       | restricted    |                 | Parquet zone   |
       +---------------+                 +----------------+

        Query results and findings flow to:

        +-------------------------------------+
        | RDS PostgreSQL                      |
        | findings, feedback, rules, audit    |
        +----------------+--------------------+
                         |
                         v
        +----------------+--------------------+
        | Review UI -> Feedback Store ->      |
        | Learning Loop -> Human Approval     |
        +-------------------------------------+
```
