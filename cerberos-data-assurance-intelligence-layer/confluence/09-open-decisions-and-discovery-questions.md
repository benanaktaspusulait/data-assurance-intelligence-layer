# Open Decisions and Discovery Questions

**Parent page:** Cerberos Data Assurance Intelligence Layer - Initial Discovery Concept  
**Status:** Private working draft  
**Purpose:** Capture unresolved architecture, delivery, and governance questions for discovery

## Why This Page Exists

This concept is intentionally not a final implementation proposal. The following decisions should be resolved during discovery before any delivery commitment is made.

The aim is to make uncertainty explicit rather than hide it.

## Assumptions to Validate

- Safe read-only data sources are available.
- Data owners or domain SMEs can review findings.
- A small PoC can run without production impact.
- Existing platform standards allow a JVM-based service.
- Security permits masked evidence and audit storage.
- Analytical datasets are available or can be created for the PoC.
- Existing AI/Copilot restrictions allow optional summarisation, or the PoC can proceed without LLM support.

## Critical Architecture Decisions

| Area | Open Question |
| --- | --- |
| Technology stack | Which implementation stack should be used for the rule registry, findings store, review UI, scheduler, and agent assistance layer? |
| Findings store | Should findings and feedback be stored in PostgreSQL, DynamoDB, OpenSearch, or another governed data store? |
| Analytical assurance layer | Which S3 curated datasets, Glue tables, and Athena workgroups are safe and suitable for analytical DQ checks? |
| Replica vs Athena split | Which rule categories should run against replica DBs, and which should run against S3/Parquet/Athena? |
| Review UI | Should the review experience be a custom UI, an internal platform tool, or an extension of an existing operational workflow? |
| Scheduler | Should rule execution be orchestrated through Step Functions, Airflow, EventBridge, Kubernetes jobs, or another scheduler? |
| Deployment model | Should services run as containers, serverless components, managed workflows, or a hybrid? |
| Infrastructure as Code | Should Terraform, CDK, or an existing platform IaC standard be used? |
| API style | Should components communicate through REST, async events, gRPC, GraphQL, or a mixed model? |

## Security and Governance Decisions

| Area | Open Question |
| --- | --- |
| Authentication | Which identity provider and authentication mechanism should be used? |
| Authorisation | Which roles can create rules, approve rules, review findings, create exceptions, and view masked evidence? |
| Sensitive evidence | What fields are allowed in finding evidence, and which must be masked, hashed, tokenised, or excluded? |
| S3 data zones | Which raw, curated, assurance metrics, backtesting, and evidence zones are required for the PoC? |
| Glue schema governance | Can Glue Crawler be used for discovery only, and when should curated Glue table definitions be IaC-managed? |
| Athena access controls | Which workgroup, result location, encryption, cost controls, and query limits are mandatory? |
| Retention | How long should findings, feedback, samples, audit records, and agent analysis outputs be retained? |
| Approval workflow | Which rule changes require approval, and who is accountable for approval? |
| Compliance review | Which security, privacy, and data governance stakeholders must review the PoC before it runs? |

## Retention Decisions to Resolve

| Area | Open Question |
| --- | --- |
| Findings retention | How long should active and closed findings be retained? |
| Feedback retention | How long should reviewer decisions, reason codes, comments, and correction history be retained? |
| Masked evidence retention | How long should masked sample references, hashes, and evidence links remain available? |
| Audit event retention | What retention period is required for rule execution, access, approval, and review audit events? |
| Agent prompt/output retention | If an agent is enabled, should prompts, structured inputs, and outputs be retained, redacted, or excluded? Recommended default: retain for 90 days in the controlled agent-output audit zone, access limited to audit and security teams, then automatic destruction - to be confirmed by governance. |
| Backtest result retention | How long should shadow run outputs and backtest reports be retained for future rule approval evidence? |
| Athena query metadata | How long should query execution IDs, scan metrics, and workgroup cost metadata be retained? |

## Platform and Operations Decisions

| Area | Open Question |
| --- | --- |
| Eventing | Should component communication use SQS, SNS, EventBridge, Kafka, or existing platform messaging? |
| Error handling | What should happen when a query times out, a connector fails, or a downstream store is unavailable? |
| Retry policy | Which failures are retryable, and how should retries be bounded? |
| CI/CD | How should rule definitions, backend services, UI changes, and dashboards be deployed? |
| Testing strategy | What unit, integration, end-to-end, and backtesting coverage is required? |
| Athena cost/performance | What scan limits, partition requirements, and query cost dashboards are required by rule? |
| Rollback | How should a rule version, exception, or service deployment be rolled back? |
| Domain isolation | How should findings and permissions be isolated across domains such as Identity, Journey, and Processing? |

## Service Management Decisions

| Area | Open Question |
| --- | --- |
| SLA targets | What are the expected times for rule execution, finding creation, review, and alert delivery? |
| Ownership | Who owns the assurance platform itself if the scheduler, executor, or findings store fails? |
| Incident process | Which operational process handles platform failures versus data quality findings? |
| Capacity planning | What query volume, storage growth, and review workload should be expected at PoC and platform scale? |
| Disaster recovery | What HA and DR requirements apply to the rule registry, findings store, feedback store, and audit logs? |
| Migration | Should any existing data quality checks or historical findings be imported? |

## Delivery and Adoption Questions

| Area | Open Question |
| --- | --- |
| Sponsor | Who is the right senior sponsor for discovery? |
| First domain | Which domain has enough value, evidence, and stakeholder support for a narrow PoC? |
| Review users | Which roles will review findings: data engineers, domain experts, platform owners, assurance users, or operational leads? |
| RACI | Who is responsible, accountable, consulted, and informed for rule ownership and finding review? |
| Success criteria | What evidence would prove the PoC is useful enough to continue? |
| Timeline | What is a realistic PoC duration and sprint plan? |

## Later Enhancements

Most of the items below have now been delivered as concrete artefacts in the child page
**Supporting Technical Detail**. They are retained here for traceability, with their current status.

Delivered in **Supporting Technical Detail**:

- Sequence diagrams for rule execution and review flows.
- Rule YAML validation schema (JSON Schema).
- Notification templates for Teams, Email, and Jira/ServiceNow.
- Confidence score calculation details.
- Agent prompt templates and guardrails.
- Backtesting methodology.
- Integration test scenarios.
- ER diagram for the conceptual data model.
- Comparison with data quality tools such as Great Expectations, Deequ, and Soda Core.

Delivered since (now produced):

- Indicative dashboard wireframes (ASCII sketches) for the platform operations and data assurance
  effectiveness views are in **Logging, Observability, and Monitoring**. Full UI designs remain a
  later, UI-phase activity.

## Recommended Next Step

Before implementation choices are made, the discovery should answer three questions:

1. Which domain and data source are safe and valuable enough for a PoC?
2. Which governance controls are mandatory from day one?
3. How small can the agent-assisted layer remain while still proving the learning loop?

Initial technology recommendations and proposed ADRs are captured in the child page **Technology Selection and Architecture Decision Report**.

Analytical data lake decisions for S3, Parquet, Glue Data Catalog and Athena are captured in **S3, Parquet, Glue Data Catalog and Athena Analytical Assurance Layer**.
