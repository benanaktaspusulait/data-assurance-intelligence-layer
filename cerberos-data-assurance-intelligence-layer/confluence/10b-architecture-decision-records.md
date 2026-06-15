# Architecture Decision Records and Reference Architecture

**Parent page:** Cerberos Data Assurance Intelligence Layer - Initial Discovery Concept  
**Status:** Private working draft  
**Purpose:** Technology decision matrix, Architecture Decision Records (ADR-001 to ADR-014), and reference PoC/platform architecture
**Companion page:** See **Technology Selection and Architecture Decision Report** for the full analysis and rationale.

## 1. Technology Decision Matrix

| Decision Area | Option | Pros | Cons | PoC Suitability | Platform Suitability | Recommendation |
| --- | --- | --- | --- | --- | --- | --- |
| Backend framework | Java 21 + Quarkus | Lightweight JVM, container-friendly, OTel support. | Requires Quarkus familiarity. | High if aligned to standards | High if adopted by platform | Candidate. |
| Backend framework | Spring Boot | Mature enterprise stack. | May be heavier if not already standard. | High if already used | High | Acceptable alternative. |
| Backend framework | Kotlin/Ktor | Lightweight Kotlin-first. | Less standard in some estates. | Medium | Medium/High if Kotlin-heavy | Conditional. |
| Rule registry | Git-backed YAML | Simple, reviewable, versioned. | Limited runtime workflow. | High | Medium | PoC. |
| Rule registry | Database-backed registry | Strong governance and workflow. | More build effort. | Medium | High | Platform. |
| Findings store | PostgreSQL | Relational, transactional, audit-friendly. | Needs scale planning. | High | High | Primary. |
| Findings store | DynamoDB | Serverless scale. | Less natural for workflow analytics. | Medium | Conditional | Defer. |
| Query execution | SafeQueryExecutor + JDBC/Athena SDK | Central governance and source flexibility. | Custom implementation needed. | High | High | Primary. |
| Query execution | ORM/JPA | Familiar app data access. | Poor fit for governed DQ queries. | Low | Low | Avoid. |
| Query execution | Hibernate Native SQL | Reuses Hibernate connection/session infrastructure if already standard. | Still needs external guardrails and does not fit Athena or multi-source execution well. | Medium only behind SafeQueryExecutor | Conditional | Limited use only. |
| Query execution | Hibernate Reactive | Non-blocking access for reactive services where supported. | Does not solve query governance, has limited source coverage, and adds complexity for scheduled checks. | Low/Medium if reactive standard | Conditional | Defer. |
| UI | React/internal UI | Flexible, fast, familiar. | Requires UI ownership. | High | High | Primary unless standard differs. |
| Scheduler | In-service/Kubernetes CronJob | Simple. | Limited workflow capability. | High | Medium | PoC. |
| Scheduler | Step Functions/Airflow/Dagster | Strong orchestration. | More complexity. | Medium | High for complex DAGs | Later. |
| Agent integration | Interface/stub | No LLM dependency, governance-friendly. | Limited intelligence initially. | High | Medium | PoC. |
| Agent integration | Koog/Spring AI/LangChain4j | Richer AI integration. | Governance and maturity assessment needed. | Medium | High if justified | Later. |
| Deployment | Container platform | Enterprise-friendly, portable. | Needs platform support. | High | High | Primary. |
| Deployment | Lambda-only | Managed and scalable for small jobs. | Risk for long-running query execution. | Low/Medium | Conditional | Avoid as core runtime initially. |

## 2. Architecture Decision Records

### ADR-001: Use a JVM-based backend service for PoC

Status: Proposed

Context: Cerberos requires an enterprise-grade backend for rule execution, APIs, audit, integration, and scheduled jobs.

Decision: Use a JVM-based backend service. Java 21 + Quarkus is a candidate PoC option if it aligns with existing platform standards.

Consequences: Strong fit for JDBC, AWS SDK, OpenTelemetry, containers, and enterprise maintainability. Team familiarity must be confirmed.

Alternatives considered: Spring Boot, Kotlin/Ktor, Python FastAPI, Node.js/NestJS.

### ADR-002: Use Git-backed YAML rules for initial PoC

Status: Proposed

Context: The PoC needs readable, reviewable rule definitions without building a full rule management product.

Decision: Use Git-backed YAML rules with validation and review.

Consequences: Fast start with version control. Runtime approval workflow is limited until a database-backed registry is introduced.

Alternatives considered: JSON, database-backed rules, Drools, OPA/Rego, custom DSL.

### ADR-003: Use PostgreSQL for findings, feedback, and rule metadata

Status: Proposed

Context: Findings, feedback, approvals, suggestions, and audit metadata are relational workflow data.

Decision: Use PostgreSQL as the primary store.

Consequences: Strong querying, transactions, and audit modelling. Scale and retention must be managed.

Alternatives considered: DynamoDB, OpenSearch, S3, graph database.

### ADR-004: Use a SafeQueryExecutor abstraction for all data source access

Status: Proposed

Context: Query execution must be controlled, read-only, audited, and source-aware.

Decision: All rule queries must go through `SafeQueryExecutor`.

Consequences: Governance controls are centralised. The executor becomes a critical component requiring strong tests.

Alternatives considered: direct JDBC, ORM/JPA, Hibernate Native SQL, Hibernate Reactive, direct Athena calls from rules.

### ADR-005: Use read-only replica and analytical sources only

Status: Proposed

Context: The platform must avoid production impact and production writes.

Decision: Use read-only replicas, Athena/S3/Parquet, reporting datasets, and approved metrics sources.

Consequences: Reduced operational risk. Some checks may lag behind real time depending on replica and analytical data freshness.

Alternatives considered: direct production DB access, uncontrolled ad hoc queries.

### ADR-006: Keep LLM/agent integration optional for PoC

Status: Proposed

Context: The assurance loop should not depend on AI governance approvals or LLM availability.

Decision: Add `AgentAnalysisService` interface but keep implementation optional or stubbed.

Consequences: PoC can proceed without LLM dependency. Agent integration can be added later behind governance.

Alternatives considered: Koog, LangChain4j, Spring AI, Bedrock, Azure OpenAI from day one.

### ADR-007: Use human approval before rule changes

Status: Proposed

Context: Learning outputs may suggest refinements, exceptions, or new rules, but autonomous changes are unsafe.

Decision: Require human approval before rule changes are deployed.

Consequences: Safer governance model. Slower iteration than full automation, but appropriate for Cerberos context.

Alternatives considered: automatic rule tuning, autonomous exception creation.

### ADR-008: Use OpenTelemetry and audit logging from the beginning

Status: Proposed

Context: The platform must be observable and auditable from the start.

Decision: Implement OpenTelemetry, structured logs, metrics, and append-only audit events in the PoC.

Consequences: Slightly more initial work, but avoids blind spots and supports governance.

Alternatives considered: add observability after PoC, rely only on platform logs.

### ADR-009: Use S3/Parquet for historical analytical assurance datasets

Status: Proposed

Context: Historical, aggregate, reconciliation, schema drift and backtesting checks should not repeatedly scan operational databases.

Decision: Store curated analytical datasets in S3 using Parquet where appropriate.

Consequences: Enables scalable, partition-aware analysis without loading operational systems. Requires dataset ownership, retention, compaction, and schema governance.

Alternatives considered: repeated replica DB scans, raw JSON-only S3 data, external data warehouse only.

### ADR-010: Use Glue Data Catalog as metadata layer for Athena-accessible datasets

Status: Proposed

Context: Athena requires table and partition metadata to query S3 datasets effectively.

Decision: Use Glue Data Catalog as the metadata layer for analytical assurance datasets.

Consequences: Provides queryable table and partition metadata. Requires governance over schema changes and catalog permissions.

Alternatives considered: unmanaged external table definitions, separate metadata registry, direct file scanning.

### ADR-011: Use Athena for partitioned analytical DQ checks and backtesting

Status: Proposed

Context: Analytical assurance requires historical and aggregate access without loading operational systems.

Decision: Use Athena for approved, partition-aware analytical checks, reconciliation and backtesting.

Consequences: Good fit for historical analytics and backtesting. Not suitable for low-latency online request paths.

Alternatives considered: replica DB scans, Spark/Glue jobs for every check, Trino/Presto, commercial DQ tools.

### ADR-012: Require all Athena access through SafeQueryExecutor

Status: Proposed

Context: Athena access must be governed, bounded, auditable and cost-controlled.

Decision: Route all Athena rule execution through `SafeQueryExecutor` and an Athena connector.

Consequences: Centralises partition validation, cost controls, masking, row limits and audit. Requires robust executor tests.

Alternatives considered: direct Athena access from UI, agent, notebooks or rule code.

### ADR-013: Use controlled Glue table definitions for critical curated datasets

Status: Proposed

Context: Glue Crawler is useful for discovery, but uncontrolled schema updates are risky for critical curated assurance datasets.

Decision: Use crawler-assisted discovery for PoC where appropriate; move critical curated datasets to controlled table definitions via IaC or approved metadata workflow.

Consequences: Improves governance and reduces surprise schema changes. Requires schema ownership process.

Alternatives considered: crawler-only catalog management, fully manual unmanaged tables.

### ADR-014: Keep raw and curated S3 zones separate

Status: Proposed

Context: Raw source-aligned payloads and curated analytical datasets have different sensitivity, access and query characteristics.

Decision: Maintain separate raw and curated zones with different access controls and usage rules.

Consequences: Supports investigation while keeping routine DQ checks on safer curated datasets. Requires clear data zone ownership.

Alternatives considered: single shared S3 zone, direct raw querying for all checks.

## 3. Recommended PoC Architecture

Concrete PoC architecture:

- JVM backend service aligned to existing platform standards; Java 21 + Quarkus is one candidate option.
- Git-backed YAML rules.
- PostgreSQL findings/feedback/audit store.
- `SafeQueryExecutor`.
- JDBC connector to one read-only replica DB.
- Athena connector optional.
- Simple React review UI.
- Teams/email notification.
- OpenTelemetry logging and metrics.
- Append-only audit table.
- Optional `AgentAnalysisService` stub.

```text
       +----------------------+
       | Git-backed YAML      |
       | Approved DQ Rules    |
       +----------+-----------+
                  |
                  v
        +---------+----------+
        | JVM Backend        |
        | Rule Scheduler     |
        | SafeQueryExecutor  |
        +----+------------------+
             |
     +-------+-------------------------------+
     |                                       |
     v                                       v
 +---+---------+                    +--------+--------+
 | Read-only   |                    | Athena Connector|
 | Replica DB  |                    | analytical DQ   |
 +---+---------+                    +--------+--------+
     |                                       |
     |                                       v
     |                              +--------+--------+
     |                              | Athena          |
     |                              +--------+--------+
     |                                       |
     |                                       v
     |                              +--------+--------+
     |                              | Glue Data       |
     |                              | Catalog         |
     |                              +--------+--------+
     |                                       ^
     |                                       |
     |                              +--------+--------+
     |                              | Glue Crawler /  |
     |                              | metadata flow   |
     |                              +--------+--------+
     |                                       ^
     |                                       |
     |                 +---------------------+------------------+
     |                 |                                        |
     |                 v                                        v
     |        +--------+--------+                      +--------+--------+
     |        | S3 raw zone    |                      | S3 curated     |
     |        | restricted     |                      | Parquet zone   |
     |        +----------------+                      +----------------+
     |
     v
 +---+---------------------------+
 | PostgreSQL                    |
 | findings, feedback, audit     |
 +---+---------------------------+
     |
     v
 +---+---------+       +----------------+       +----------------+
 | Review UI   |-----> | Feedback Store |-----> | Learning Loop  |
 | React/Admin |       | structured     |       | suggestions    |
 +-------------+       +----------------+       +----------------+
     |
     v
 +---+------------+
 | Teams / Email  |
 | Notification   |
 +----------------+
```

Athena access in the PoC remains optional and must be routed through `SafeQueryExecutor`. The agent/Copilot layer is not shown as a query path because it must not query S3 or Athena directly.

Explicitly out of scope for PoC:

- Automatic remediation.
- Production writes.
- Unrestricted AI access.
- Full dashboard suite.
- Multi-domain rollout.
- ML anomaly detection.
- Enterprise workflow engine.
- Autonomous rule deployment.

## 4. Future Platform Architecture

The PoC can evolve into a platform by adding:

- Database-backed rule registry.
- Rule approval workflow.
- Advanced dashboard.
- OpenSearch for search and analytics.
- Athena/S3 integration at scale.
- Governed connector library.
- Anomaly detection.
- Copilot/agent-assisted summaries.
- ServiceNow/Jira integration.
- Owner routing.
- Backtesting engine.
- Rule simulation.
- Exception management.
- Multi-domain support.
- Governed remediation suggestions.

Platform evolution should be driven by reviewed findings and user feedback, not by technology expansion for its own sake.
