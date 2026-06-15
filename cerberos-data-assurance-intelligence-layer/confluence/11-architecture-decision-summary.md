# Architecture Decision Summary

**Parent page:** Cerberos Data Assurance Intelligence Layer - Initial Discovery Concept  
**Status:** Private working draft  
**Purpose:** One-page summary of the Technology Selection and Architecture Decision Report for fast stakeholder review  
**Full detail:** See `10-technology-selection-and-architecture-decision-report.md`

## Recommended PoC Stack

| Layer | Choice |
| --- | --- |
| Backend | JVM service aligned to existing platform standards; Java 21 + Quarkus is a candidate lightweight option (Spring Boot if already standard; Kotlin/Ktor if Kotlin-heavy) |
| Rule definition | Git-backed YAML with schema validation |
| Query execution | `SafeQueryExecutor` over JDBC/jOOQ + AWS Athena SDK |
| Data sources | One read-only replica DB connector (Athena/S3 connector optional) |
| Findings & feedback store | PostgreSQL |
| Review UI | Simple React or existing internal admin UI |
| Scheduler | In-service scheduler or Kubernetes CronJob |
| Notifications | Teams webhook / email / Jira placeholder |
| Observability | OpenTelemetry traces, structured JSON logs, metrics, append-only audit table |
| Agent/LLM | Optional `AgentAnalysisService` interface, stubbed initially |
| Deployment | Containerised service on existing enterprise platform + RDS PostgreSQL |

## Key Technology Decisions

1. JVM-based backend service aligned to existing Cerberos platform standards.
2. Git-backed YAML rules for PoC, evolving to a database-backed registry later.
3. PostgreSQL as the single source of truth for findings, feedback, rule metadata, and audit.
4. All data access routed through a mandatory `SafeQueryExecutor` abstraction.
5. Read-only replica and analytical sources only; no production writes.
6. LLM/agent integration kept optional behind a governed interface.
7. Human approval required before any rule change.
8. OpenTelemetry and audit logging built in from day one.

## Why Java 21 + Quarkus (or Kotlin/Ktor)

- Lightweight, cloud-native JVM runtime with strong container fit for scheduled jobs and REST APIs.
- First-class JDBC and AWS SDK support for governed query execution.
- Native OpenTelemetry support, so observability comes early without extra build-out.
- Fits enterprise maintainability, deployment, and incident patterns.
- Java 21 + Quarkus is suitable as a candidate PoC option if it aligns with platform standards. Kotlin/Ktor is a valid alternative for Kotlin-heavy teams or if future Koog orchestration becomes a driver. Spring Boot may be more appropriate if Cerberos is already strongly Spring-based.

## Why PostgreSQL for Findings/Feedback

- Findings, feedback, approvals, suggestions, and audit are inherently relational workflow data.
- Transactions and relational modelling support reliable lifecycle state and audit trails.
- Easy ad hoc querying for confidence scoring, false-positive trends, and review analytics.
- Strong PoC fit with a clear path to scale (OpenSearch for search/analytics, S3 for archival, later if needed).

## Why YAML Rules Are Enough for PoC

- Human-readable, reviewable, and Git-friendly with built-in version control.
- Fast to start without building a full rule-management product.
- JSON Schema validation provides safety; the Git review process provides governance.
- A database-backed registry with runtime approval workflow can be introduced in the platform phase once the model is proven.

## Why SafeQueryExecutor Is Mandatory

The platform must never become an uncontrolled query interface. All queries must pass through a single layer enforcing:

- SELECT-only execution and a SQL deny-list.
- Allow-listed sources and approved, parameterised query templates.
- Timeouts, row limits, concurrency limits, and Athena cost controls.
- Result masking, output minimisation, and full query audit.

No UI, scheduler, agent, or rule service may query a data source directly. This centralises governance and keeps the security model verifiable.

## Why LLM/Agent Integration Should Be Optional Initially

- The assurance loop must not depend on AI governance approvals or LLM availability.
- The learning loop can start deterministically (approval/rejection ratios, confidence scoring, false-positive trends, exception detection) using SQL analytics.
- An `AgentAnalysisService` interface reserves the integration point without creating a dependency.
- When added, the agent receives only masked, structured finding summaries, never raw PII, and outputs remain subject to human review.

## Out of Scope (PoC)

- Automatic remediation and autonomous rule deployment.
- Production writes and direct production DB access.
- Unrestricted AI access and ML anomaly detection.
- Full dashboard suite and enterprise workflow engine.
- Multi-domain rollout before one domain is proven.
- Large commercial DQ platform commitment.

## Key Risks and Mitigations

| Risk | Mitigation |
| --- | --- |
| Overengineering / too many technologies | One backend, one store, one source connector; prove the loop first. |
| LLM governance delays | Keep LLM optional behind an interface/stub. |
| Query performance impact | Time windows, partition pruning, row limits, timeouts, cost limits. |
| AWS cost surprises | Athena cost and concurrency controls, budget monitoring. |
| False positives | Structured feedback capture and rule confidence tracking. |
| Security approvals | Read-only sources, PII masking, least privilege, audit from day one. |
| Vendor lock-in | Keep domain logic behind portable service interfaces. |
| Building a custom DQ platform unnecessarily | Evaluate existing tools (Great Expectations, Deequ, Soda, Glue DQ) for reuse later. |

## Next Recommended Steps

1. Confirm existing Cerberos platform standards, then select Quarkus, Spring Boot, Kotlin/Ktor, or another JVM standard accordingly.
2. Identify one data domain, one safe read-only source, and five candidate rules.
3. Stand up the PoC skeleton: JVM service, PostgreSQL schema, `SafeQueryExecutor`, one replica connector.
4. Implement YAML rule loading with schema validation and the findings/feedback model.
5. Build the minimal review UI and structured feedback capture.
6. Wire OpenTelemetry, structured logging, and the append-only audit table.
7. Add deterministic confidence scoring and one rule-refinement suggestion.
8. Review evidence with stakeholders before proposing platformisation.
