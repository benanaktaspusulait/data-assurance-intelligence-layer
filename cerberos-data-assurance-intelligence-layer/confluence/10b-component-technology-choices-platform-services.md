# Component Technology Choices - Platform Services

**Parent page:** Cerberos Data Assurance Intelligence Layer - Initial Discovery Concept  
**Status:** Private working draft  
**Purpose:** Per-component technology options for platform services (learning engine, agent integration, AWS, scheduling, observability, security, deployment)
**Companion pages:** **Component Technology Choices** (backend, registry, query execution, sources, stores, UI) and **Technology Selection and Architecture Decision Report** (overview and decisions).

## 1. Learning and Insight Engine

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

## 2. Agent / Copilot Integration Technology

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

Agents must not directly query Athena or S3 and must not generate arbitrary SQL for execution. The correct model is approved rule execution through `SafeQueryExecutor`, followed by controlled, masked, bounded results that may be summarised by an agent for human review. The strategy for abstracting over multiple providers is in **Multi-Provider Agent Framework Strategy**.

## 3. AWS Technology Choices

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

## 4. Scheduling and Orchestration

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

## 5. Observability and Audit

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

## 6. Security Architecture

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

## 7. Deployment Architecture

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
