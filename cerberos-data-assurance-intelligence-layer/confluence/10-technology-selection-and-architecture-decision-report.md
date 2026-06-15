# Technology Selection and Architecture Decision Report

**Parent page:** Cerberos Data Assurance Intelligence Layer - Initial Discovery Concept  
**Status:** Private working draft  
**Purpose:** Initial technology recommendation for discovery and PoC planning  
**Scope:** Technology selection, trade-offs, PoC stack, platform evolution, and initial architecture decisions

For the shorter technical summary, see **Architecture Decision Summary**.

For analytical assurance over S3, Parquet, Glue Data Catalog, and Athena, see **S3, Parquet, Glue Data Catalog and Athena Analytical Assurance Layer**.

For the technology decision matrix and the full Architecture Decision Records (ADR-001 to ADR-014), see **Architecture Decision Records and Reference Architecture**.

## 1. Executive Summary

This document proposes an initial technology direction for implementing the Cerberos Data Assurance Intelligence Layer. It is a discovery and PoC recommendation, not a final enterprise-wide technology decision.

The recommended first direction is a deliberately controlled JVM-based service that runs approved read-only rules, stores findings and feedback in PostgreSQL, integrates with replica databases and Athena/S3/Parquet datasets, exposes a simple review UI, and keeps agent/Copilot capability optional behind a governed interface.

Candidate PoC direction:

- Backend: JVM service aligned to existing Cerberos platform standards; Java 21 + Quarkus is a candidate PoC option.
- Acceptable backend alternative: Spring Boot if Cerberos is already strongly Spring-based.
- Rule registry: Git-backed YAML rules for PoC, evolving to database-backed versioned registry.
- Query execution: custom `SafeQueryExecutor` abstraction over JDBC/jOOQ and AWS Athena SDK.
- Analytical data checks: Athena over S3/Parquet with Glue Data Catalog metadata where available.
- Operational data checks: read-only replica DB connectors.
- Findings and feedback store: PostgreSQL.
- UI: simple React or existing internal web UI pattern.
- Scheduler: in-service scheduler, Quartz, or Kubernetes CronJob for PoC.
- Observability: OpenTelemetry, structured JSON logs, metrics, and audit events from day one.
- Deployment: containerised service on the existing enterprise container platform.
- Agent support: optional `AgentAnalysisService` interface, initially stubbed or limited to approved enterprise LLM endpoints.

The first version should avoid overengineering. It should prove the governance loop: approved rule, read-only execution, controlled finding, human review, structured feedback, and human-approved rule improvement.

## 2. Technology Selection Criteria

Technology choices should be evaluated against the security and operational context of Cerberos, not only against developer convenience.

| Criterion | Weight | Why It Matters | Evaluation Notes |
| --- | ---: | --- | --- |
| Security and governance suitability | 10 | Border/security context requires controlled execution, least privilege, and auditability. | Must support RBAC/IAM, no production writes, query controls, and audit trail. |
| Read-only controlled execution | 10 | The platform must not become an uncontrolled query interface. | All query paths must pass through approved templates and safe execution controls. |
| Auditability | 9 | Findings, feedback, approvals, and rule changes must be explainable. | Prefer technologies that support immutable audit events and traceable metadata. |
| Scalability | 9 | Cerberos operates around 5 billion events per month. | Avoid raw full scans; support partitioned, bounded, and cost-aware execution. |
| Operational maintainability | 8 | The platform must be supportable by enterprise teams. | Prefer familiar runtime, deployment, monitoring, and incident patterns. |
| AWS integration | 8 | S3, Athena, Glue, IAM, CloudTrail, Secrets Manager, and CloudWatch are likely relevant. | Prefer first-class SDK and IAM integration. |
| JVM ecosystem fit | 8 | Java/Kotlin fits enterprise backend and data access patterns. | Prefer mature JDBC, AWS SDK, OpenTelemetry, testing, and security libraries. |
| Developer productivity | 7 | PoC must be deliverable without unnecessary platform build-out. | Prefer simple service boundaries and minimal framework ceremony. |
| Testability | 7 | Rule behaviour, query controls, and learning outputs must be testable. | Prefer clear interfaces and deterministic components. |
| Observability | 8 | The monitor must itself be monitored. | Require logs, metrics, traces, health checks, and dashboards. |
| Cost control | 8 | Athena and large data queries can create cost surprises. | Require query cost guardrails, row limits, concurrency limits, and schedule control. |
| Phased delivery | 8 | The PoC should evolve into platform capability without rewrite. | Prefer modular architecture and abstractions over one-off scripts. |
| Future AI/agent integration | 6 | Agent assistance may be useful but should not be central at first. | Add interface boundaries without requiring LLM in PoC. |
| Avoiding lock-in | 6 | Public-sector constraints may require portability. | Prefer open standards and isolating cloud-specific code behind interfaces. |

## 3. Proposed Logical Components

| Component | Purpose | Recommended Technology | Alternatives | PoC Option | Future Platform Option |
| --- | --- | --- | --- | --- | --- |
| Rule Registry | Store approved rules, versions, owners, schedules, thresholds, and governance state. | Git-backed YAML initially, later PostgreSQL-backed registry. | JSON, Drools, OPA/Rego, custom DSL, Great Expectations/dbt style checks. | Git-backed YAML with review process. | Database-backed registry with approval workflow and audit. |
| Rule Definition Format | Human-readable rule format. | YAML with schema validation. | JSON, DSL, SQL files, DQDL, Rego. | YAML plus JSON Schema validation. | YAML/DSL stored as versioned registry records. |
| Scheduler / Orchestration | Trigger rules on schedule and manage backfill. | Quarkus/Spring scheduler or Kubernetes CronJob. | Quartz, EventBridge, Step Functions, Airflow, Dagster. | In-service scheduler or CronJob. | Enterprise scheduler/orchestrator depending on platform standards. |
| Safe Query Execution Engine | Enforce query safety and execute approved checks. | Custom `SafeQueryExecutor` over JDBC/jOOQ and Athena SDK. | Direct JDBC, Apache Calcite, Trino/Presto, custom scripts. | Custom executor with JDBC connector. | Multi-connector executor with cost, concurrency, masking, and audit controls. |
| Source Connectors | Isolate access to replica DBs, Athena/S3, metrics, and reporting datasets. | Connector interface per source type. | Direct SQL from rule code, ETL job integration, DQ tool connectors. | One read-only replica connector and optional Athena connector. | Governed connector library with health checks and source metadata. |
| Findings Store | Persist findings and lifecycle state. | PostgreSQL. | DynamoDB, OpenSearch, S3, graph DB. | PostgreSQL. | PostgreSQL plus OpenSearch for search/analytics and S3 archival if required. |
| Feedback Store | Capture structured review decisions and corrections. | PostgreSQL. | DynamoDB, event store, ticketing system only. | PostgreSQL. | PostgreSQL with audit events and analytics views. |
| Learning and Insight Engine | Analyse feedback and suggest refinements. | Deterministic Java/Kotlin service using PostgreSQL queries. | Python jobs, Spark/Glue, OpenSearch aggregations, ML. | SQL analytics in backend service. | Separate learning service and historical analytics jobs. |
| Review UI | Human review and structured feedback. | React or existing internal UI pattern. | Angular, server-rendered UI, low-code tool. | Simple React/admin UI. | Enterprise-aligned UI with dashboards and workflow integration. |
| Dashboard / Reporting | Assurance posture and operational views. | UI dashboards plus PostgreSQL views; later Grafana/OpenSearch. | BI tool, CloudWatch dashboard, custom reporting. | Basic UI/report views. | Executive and technical dashboards. |
| Notification / Ticketing | Route findings to owners. | Teams/email/Jira placeholder. | ServiceNow, Jira, PagerDuty, EventBridge. | Teams/email or Jira placeholder. | Jira/ServiceNow/Teams integration with deduplication. |
| Agent / Copilot Integration | Optional summaries, triage, and rule suggestions. | `AgentAnalysisService` interface; no LLM dependency initially. | Koog, Spring AI, LangChain4j, Bedrock, Azure OpenAI, Copilot. | Stub/manual or approved LLM endpoint. | Governed agent orchestration if value is proven. |
| Audit and Governance | Trace decisions, approvals, access, and rule changes. | Append-only audit table plus structured logs. | Event store, SIEM-only logging. | PostgreSQL audit table. | Immutable audit store, SIEM integration, retention controls. |
| Observability | Monitor platform health. | OpenTelemetry, structured logs, metrics. | CloudWatch-only, Datadog, Prometheus/Grafana. | OTel traces/logs/metrics and basic dashboard. | Full distributed tracing, cost metrics, SLOs, and alerting. |
| Deployment Platform | Run backend, UI, and scheduled jobs. | Containerised service on existing platform. | EKS, OpenShift, ECS/Fargate, VM, Lambda. | Containerised app. | Existing enterprise container platform with IaC and CI/CD. |

## 4. Candidate PoC Technology Stack

Candidate minimal PoC stack:

- Backend: JVM service aligned to existing Cerberos platform standards; Java 21 + Quarkus is a candidate lightweight option.
- Alternative backend: Spring Boot if the existing Cerberos estate is strongly Spring-based; Kotlin/Ktor if the estate is Kotlin-heavy.
- Rule definition: YAML with validation.
- Rule registry: Git-backed files initially.
- Query execution: JDBC/jOOQ for replica DBs; AWS Athena SDK for Athena checks.
- Findings and feedback store: PostgreSQL.
- UI: React or existing internal web UI pattern.
- Scheduler: Quarkus Scheduler, Spring Scheduler, Quartz, or Kubernetes CronJob.
- Notifications: Teams webhook, email, or Jira placeholder.
- Agent support: optional; initially stubbed or routed through approved enterprise LLM capability.
- Observability: OpenTelemetry, structured JSON logs, metrics, health checks.
- Deployment: containerised service on Kubernetes/OpenShift/EKS depending on platform standards.
- Secrets: AWS Secrets Manager, Kubernetes secrets, or enterprise secrets solution.
- Audit: append-only audit table plus structured logs.

Why this is suitable for discovery:

- It uses familiar enterprise technologies.
- It keeps the first version small and explainable.
- It avoids making LLM access a dependency.
- It supports read-only rule execution and audit from day one.
- It can evolve toward a governed platform without rewriting the core model.

## 5. Backend Technology Choice

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

## 6. Rule Definition and Rule Registry

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

Sample YAML rule:

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

## 7. Query Execution Technology

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
| Apache Calcite | Query planning and SQL abstraction. | Likely overkill for PoC. | Consider only if multi-source SQL abstraction becomes necessary. |
| Presto/Trino | Powerful distributed query engine. | Platform dependency and operational overhead. | Consider later if already available. |
| Athena SDK | Native AWS path for S3/Parquet analytical checks. | Needs cost and timeout controls. | Recommended for Athena checks. |
| Custom executor | Centralises governance. | Must be designed carefully. | Required abstraction. |

Interface example:

```java
public interface SafeQueryExecutor {
    QueryResult execute(ApprovedRule rule, QueryParameters parameters);
}
```

All query execution must go through this layer. No UI, scheduler, agent, or rule service should query data sources directly.

## 8. Data Sources and Connectors

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

## 9. Findings and Feedback Store

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

## 10. Review UI Technology

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

## 11. Learning and Insight Engine

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

## 12. Agent / Copilot Integration Technology

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

## 13. AWS Technology Choices

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

## 14. Scheduling and Orchestration

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

## 15. Observability and Audit

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

Audit and observability should be treated separately. Observability shows whether the system is healthy. Audit shows who did what, when, and why.

## 16. Security Architecture

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

## 17. Deployment Architecture

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

The technology decision matrix, the full set of Architecture Decision Records (ADR-001 to ADR-014), and the reference PoC and future platform architecture diagrams are captured in the child page **Architecture Decision Records and Reference Architecture** to keep this report at a readable length.

## 18. Risks and Trade-Offs

| Risk | Mitigation |
| --- | --- |
| Overengineering | Keep PoC small and prove assurance loop first. |
| Choosing too many technologies | Use one backend, one store, one source connector initially. |
| LLM governance delays | Make LLM optional; use interface/stub first. |
| Query performance | Use time windows, partition pruning, row limits, timeouts, and cost limits. |
| False positives | Capture structured feedback and track confidence. |
| UI adoption | Keep review flow lightweight and evidence-rich. |
| Security approvals | Use read-only sources, PII masking, least privilege, and audit from day one. |
| Unclear ownership | Assign rule owners and capture corrected owner feedback. |
| Operational burden | Prefer existing platform standards and managed services where appropriate. |
| AWS cost surprises | Add Athena cost controls, concurrency limits, and budget monitoring. |
| Vendor lock-in | Keep domain logic in portable service interfaces. |
| Building custom DQ platform unnecessarily | Evaluate existing tools and reuse them where they fit. |

## 19. Build vs Buy / Existing Tools Consideration

Existing data quality tools should be considered, but the differentiator in this concept is not only rule execution. It is the governed learning loop: reviewed findings, structured feedback, rule confidence, exception discovery, owner routing learning, and human-approved rule evolution.

Options to consider:

- Great Expectations.
- Deequ.
- dbt tests.
- Soda.
- Monte Carlo or other commercial data observability platforms.
- AWS Glue Data Quality.
- Custom lightweight platform.

Evaluation factors:

- Public-sector constraints.
- Data sensitivity.
- Deployment model.
- Integration with review UI.
- Structured human feedback.
- Governance and audit.
- Cost.
- Custom Cerberos domain knowledge.
- Existing platform standards.

Recommendation:

- For PoC, a custom lightweight rule/findings/feedback loop is justified because the core differentiator is Cerberos-specific assurance and human-in-the-loop learning.
- Existing DQ tools can be evaluated later for rule execution, validation, or analytical checks.
- Avoid premature commitment to a large commercial platform before validating the learning loop.

## 20. Final Recommendation

Candidate PoC stack:

- JVM backend aligned to existing platform standards; Java 21 + Quarkus is a candidate lightweight option.
- Git-backed YAML rules.
- PostgreSQL findings, feedback, rule metadata, and audit.
- `SafeQueryExecutor` abstraction.
- JDBC connector to one read-only replica DB.
- Optional Athena connector for S3/Parquet checks.
- Simple React or internal review UI.
- OpenTelemetry, structured logs, metrics, and audit events.
- Containerised deployment on existing enterprise platform.
- Optional `AgentAnalysisService` stub, with no LLM dependency initially.

Avoid initially:

- Production writes.
- Direct production DB access.
- Uncontrolled ad hoc SQL.
- Autonomous remediation.
- Full ML/agent platform build-out.
- Large commercial platform commitment.
- Multi-domain rollout before one domain is proven.

Decisions that can be deferred:

- Full rule registry product.
- Advanced orchestration engine.
- Agent framework selection.
- OpenSearch search layer.
- ML anomaly detection.
- Enterprise workflow integration.

Governance-first design matters because the value of the platform depends on trust. The system must prove that it can execute safely, explain findings, capture human feedback, and improve over time without becoming an uncontrolled automation surface.

The learning loop can be built incrementally without heavy AI dependency. The first version can learn from structured feedback using deterministic analytics. AI/agent support can then be added behind governed interfaces once the basic assurance loop is proven.

## References for Technology Validation

- Quarkus OpenTelemetry: https://quarkus.io/guides/opentelemetry
- OpenTelemetry Quarkus instrumentation: https://opentelemetry.io/docs/zero-code/java/quarkus/
- AWS Athena and Glue Data Catalog: https://docs.aws.amazon.com/athena/latest/ug/data-sources-glue.html
- AWS Glue Data Quality: https://docs.aws.amazon.com/glue/latest/dg/glue-data-quality.html
- AWS Glue Data Quality features: https://aws.amazon.com/glue/features/data-quality/
- Great Expectations: https://greatexpectations.io/
- LangChain4j: https://docs.langchain4j.dev/
- Koog: https://docs.koog.ai/
- Spring AI observability: https://docs.spring.io/spring-ai/reference/observability/index.html
- Google ADK: https://adk.dev/
