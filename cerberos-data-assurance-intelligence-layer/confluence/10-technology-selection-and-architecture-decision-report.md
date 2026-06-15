# Technology Selection and Architecture Decision Report

**Parent page:** Cerberos Data Assurance Intelligence Layer - Initial Discovery Concept  
**Status:** Private working draft  
**Purpose:** Initial technology recommendation for discovery and PoC planning  
**Scope:** Technology selection, trade-offs, PoC stack, platform evolution, and initial architecture decisions

For the shorter technical summary, see **Architecture Decision Summary**.

For analytical assurance over S3, Parquet, Glue Data Catalog, and Athena, see **S3, Parquet, Glue Data Catalog and Athena Analytical Assurance Layer**.

For the technology decision matrix and the full Architecture Decision Records (ADR-001 to ADR-018), see **Architecture Decision Records and Reference Architecture**.

For the detailed per-component technology options, see **Component Technology Choices**.

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
| Safe Query Execution Engine | Enforce query safety and execute approved checks. | Custom `SafeQueryExecutor` over JDBC/jOOQ and Athena SDK. | Direct JDBC, Hibernate Native SQL, Hibernate Reactive, Apache Calcite, Trino/Presto, custom scripts. | Custom executor with JDBC connector. | Multi-connector executor with cost, concurrency, masking, and audit controls. |
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

The detailed per-component technology options and recommendations (backend, rule registry, query execution, data sources, findings and feedback store, review UI, learning engine, agent integration, AWS services, scheduling, observability, security, and deployment) are captured in the child page **Component Technology Choices** to keep this report at a readable length.

The technology decision matrix, the full set of Architecture Decision Records (ADR-001 to ADR-018), and the reference PoC and future platform architecture diagrams are captured in the child page **Architecture Decision Records and Reference Architecture** to keep this report at a readable length.

## 5. Risks and Trade-Offs

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

## 6. Build vs Buy / Existing Tools Consideration

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

## 7. Final Recommendation

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
