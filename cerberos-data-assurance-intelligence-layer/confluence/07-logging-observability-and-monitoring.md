# Logging, Observability, and Monitoring

**Parent page:** Cerberos Data Assurance Intelligence Layer - Initial Discovery Concept  
**Status:** Private working draft

## Why Observability Matters

The Data Assurance Intelligence Layer monitors other systems for data quality. But who monitors the monitor?

Without structured observability, the platform cannot answer:

- Is a rule failing silently?
- Is the scheduler running on time?
- Are queries taking longer than expected?
- Is the findings store growing beyond capacity?
- Did a connector lose access to a replica?
- Is the learning engine producing stale suggestions?
- Are alerts being delivered?

A data assurance system that is itself unobservable is a liability, not an asset.

## Logging Strategy

### Log Categories

| Category | Purpose | Examples |
| --- | --- | --- |
| Rule Execution | Track each rule run lifecycle | Rule started, query submitted, query completed, threshold evaluated, finding created/not created |
| Scheduler | Track scheduling reliability | Rule triggered, rule skipped, schedule drift, backlog depth |
| Query Execution | Track query performance and safety | Query duration, rows scanned, rows returned, cost estimate, timeout, error |
| Connector Health | Track data source availability | Connection established, connection lost, reconnect attempt, auth failure |
| Feedback Capture | Track user interaction | Feedback submitted, decision recorded, exception created |
| Learning Engine | Track learning pipeline | Pattern analysis started, pattern detected, suggestion generated, suggestion expired |
| Notification Delivery | Track alert reliability | Alert sent, alert delivered, alert failed, delivery latency |
| Governance Actions | Track approval workflow | Rule approved, rule rejected, rule deployed, rule rolled back |
| Agent Activity | Track AI-assisted analysis | Agent invoked, agent response generated, agent guardrail triggered |
| System Health | Track platform internals | Memory usage, queue depth, dead letter events, error rates |

### Log Structure

All logs should follow a consistent structured format:

```json
{
  "timestamp": "2026-06-15T10:32:14.221Z",
  "level": "INFO",
  "service": "dq-rule-executor",
  "component": "safe-query-executor",
  "trace_id": "abc-123-def-456",
  "correlation_id": "rule-run-2026-06-15-00412",
  "rule_id": "dq-rule-identity-001",
  "rule_version": 3,
  "domain": "Identity",
  "source_system": "CarrierGateway-A",
  "event": "QUERY_COMPLETED",
  "duration_ms": 4230,
  "rows_returned": 847,
  "cost_estimate_units": 12.4,
  "message": "Rule query completed within threshold"
}
```

### Log Levels

| Level | Usage |
| --- | --- |
| ERROR | Unrecoverable failure requiring attention (query crash, connector auth failure, store write failure) |
| WARN | Degraded condition (query near timeout, high cost, retry triggered, stale data detected) |
| INFO | Normal operational event (rule started, finding created, feedback recorded) |
| DEBUG | Detailed trace for troubleshooting (query plan, threshold calculation, confidence adjustment) |

### PII in Logs

Logs must not contain raw PII. Any identifiers in log output should be:

- Hashed or tokenised.
- Replaced with masked references.
- Limited to aggregate counts.

This aligns with the overall platform principle of output minimisation.

## Metrics

### Key Metrics to Collect

**Rule Execution Metrics:**

- `dq.rule.execution.count` - Total rule executions (by rule, domain, source)
- `dq.rule.execution.duration_ms` - Query execution time (p50, p95, p99)
- `dq.rule.execution.success_rate` - Successful runs / total runs
- `dq.rule.execution.timeout_count` - Queries that hit timeout
- `dq.rule.execution.cost_units` - Query cost per execution
- `dq.rule.execution.rows_scanned` - Data volume per query

**Finding Metrics:**

- `dq.finding.created.count` - Findings created (by severity, domain, source)
- `dq.finding.open.count` - Currently open findings
- `dq.finding.age_hours` - Time since finding created without resolution
- `dq.finding.false_positive_rate` - Rejected findings / total findings (per rule)

**Feedback Metrics:**

- `dq.feedback.submitted.count` - Feedback events (by decision type)
- `dq.feedback.time_to_review_hours` - Time from finding creation to first feedback
- `dq.feedback.exception_rate` - Exceptions created / total feedback

**Learning Metrics:**

- `dq.learning.suggestions.count` - Suggestions generated
- `dq.learning.suggestions.approved_rate` - Approved / total reviewed
- `dq.learning.confidence_drift` - Confidence score changes over time

**Platform Health Metrics:**

- `dq.scheduler.drift_seconds` - Difference between expected and actual trigger time
- `dq.scheduler.backlog_depth` - Rules waiting to execute
- `dq.connector.health` - 1 (healthy) or 0 (unhealthy) per source
- `dq.notification.delivery_latency_ms` - Time from finding to alert delivery
- `dq.notification.failure_count` - Failed alert deliveries
- `dq.store.write_latency_ms` - Finding/feedback store write time
- `dq.store.size_gb` - Storage growth

## Tracing

Distributed tracing should link:

```text
Schedule trigger
  -> Rule execution
  -> Query submission
  -> Result processing
  -> Threshold evaluation
  -> Finding creation
  -> Notification dispatch
  -> User review
  -> Feedback capture
  -> Learning analysis
```

Each trace should carry a `correlation_id` (typically the `rule_run_id`) so that any step in the pipeline can be investigated end-to-end.

## Monitoring and Alerting (Self-Monitoring)

### Health Checks

| Check | Frequency | Alert Condition |
| --- | --- | --- |
| Scheduler heartbeat | Every minute | No heartbeat for 5 minutes |
| Connector availability | Every 5 minutes | Source unreachable for 2 consecutive checks |
| Findings store write test | Every 5 minutes | Write failure or latency > 5s |
| Rule execution backlog | Every minute | Backlog > 50 rules or > 30 minutes old |
| Query cost accumulation | Hourly | Hourly cost exceeds budget threshold |
| Notification delivery | Per alert | Delivery failure or latency > SLA |
| Dead letter queue depth | Every minute | DLQ depth > 0 for more than 10 minutes |

Indicative PoC default thresholds (to be tuned with real data): daily Athena/query budget alert at
USD 10/day and a hard cap at USD 25/day; hourly cost alert at USD 2/hour; "cost spike" defined as 3x
the trailing 7-day hourly average; backlog alert at > 50 rules or > 30 minutes old. These are starting
values, not commitments.

### Self-Monitoring Alerts

| Condition | Severity | Action |
| --- | --- | --- |
| Rule executor down | Critical | Page on-call team |
| Multiple connectors failing | High | Alert platform team |
| Scheduler drift > 15 minutes | High | Alert platform team |
| Query cost spike (3x normal) | High | Alert and pause non-critical rules |
| Finding store write failures | Critical | Page on-call, halt new executions |
| Learning engine stale (no output for 7 days) | Low | Dashboard flag |
| Notification delivery failure rate > 10% | Medium | Alert notification owner |

### Circuit Breaker

Health checks detect problems; a circuit breaker contains them. Each connector and the
`SafeQueryExecutor` should implement the circuit-breaker pattern so a failing or exhausted data source
does not cause a backlog of retrying rule runs.

- If the error rate exceeds 50% over the last 5 minutes, the circuit **opens**.
- While open, rule executions against that source fail fast (marked failed immediately with an alert)
  instead of queuing - this prevents a thundering-herd / pile-up effect.
- After 1 minute the circuit moves to **half-open** and allows a single test query; if it succeeds the
  circuit **closes**, otherwise it re-opens.
- Circuit state transitions are logged (Connector Health category) and surfaced on the operations
  dashboard.

This is particularly important when Athena workgroups throttle or a replica connection pool is
exhausted, allowing the platform to protect itself and recover automatically.

## Dashboards

### Platform Operations Dashboard

- Rule execution success/failure over time.
- Average and p99 query duration.
- Scheduler backlog and drift.
- Connector health status map.
- Cost accumulation (daily, weekly, monthly).
- Error rate by component.
- Dead letter queue depth.

### Data Assurance Effectiveness Dashboard

- Findings created vs confirmed vs rejected.
- False positive rate trend (per rule, per domain).
- Mean time to review.
- Confidence score distribution.
- Learning suggestions generated vs approved.
- Top recurring issues.
- Coverage: domains and sources with active rules vs total.

### Dashboard Wireframes (Indicative)

Simple indicative layouts; not final UI designs.

Platform Operations Dashboard:

```text
+-------------------------------------------------------------+
| Platform Operations                          [last 24h | 7d]|
+----------------------+----------------------+---------------+
| Rule runs  OK / FAIL | Query p99 latency    | Cost (day/wk) |
|   12,340 / 27        |   4.2s               |  $18 / $120   |
+----------------------+----------------------+---------------+
| Scheduler backlog/drift | Connector health map            |
|   3 rules / 0s         | A:OK  B:OK  Athena:OPEN(breaker) |
+----------------------+----------------------+---------------+
| Dead-letter queue depth: 0     Error rate by component: ... |
+-------------------------------------------------------------+
```

Data Assurance Effectiveness Dashboard:

```text
+-------------------------------------------------------------+
| Data Assurance Effectiveness                 [domain v][7d] |
+----------------------+----------------------+---------------+
| Findings created     | Confirmed vs Rejected | False-pos %  |
|   4,210 (clustered)  |   1,980 / 410        |   9.4%        |
+----------------------+----------------------+---------------+
| Mean time to review  | Confidence distribution             |
|   3.1h               |   [#### high ## mid # low ]          |
+----------------------+----------------------+---------------+
| Learning suggestions: generated 12 / approved 7            |
| Top recurring issues: missing nationality (Source A) ...   |
+-------------------------------------------------------------+
```

## Technology Considerations

Specific tooling decisions are deferred to the PoC phase, but candidate options include:

| Concern | Options |
| --- | --- |
| Structured logging | CloudWatch Logs, Fluent Bit, OpenTelemetry Collector |
| Metrics | CloudWatch Metrics, Prometheus, Datadog |
| Tracing | AWS X-Ray, OpenTelemetry, Jaeger |
| Dashboards | CloudWatch Dashboards, Grafana, Datadog |
| Alerting | CloudWatch Alarms, PagerDuty, OpsGenie |
| Log analysis | CloudWatch Insights, OpenSearch, Splunk |

The PoC should start with the simplest viable option (likely CloudWatch-native) and evolve as requirements become clearer.

## Audit vs Observability

It is important to distinguish:

- **Audit logging** (covered in Governance page): Who did what, when, and why. Immutable. Compliance-driven. Long retention.
- **Observability** (this page): Is the system healthy, performant, and reliable? Operational. Shorter retention. Used for debugging and capacity planning.

Both are necessary. Audit logging alone does not tell you the system is working correctly. Observability alone does not satisfy governance or compliance.

## PoC Logging Scope

For the initial PoC, the minimum viable observability includes:

- Structured JSON logs for rule execution lifecycle.
- Basic metrics: execution count, duration, success rate, finding count.
- One health check: scheduler heartbeat.
- One dashboard: rule execution and findings overview.
- Alerts: rule executor failure, connector failure.

Advanced tracing, cost monitoring, and learning engine observability can be added in later phases.
