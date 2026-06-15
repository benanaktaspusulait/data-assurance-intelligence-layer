# Multi-Provider Agent Framework Strategy

**Parent page:** Cerberos Data Assurance Intelligence Layer - Initial Discovery Concept  
**Status:** Private working draft  
**Purpose:** Abstraction, evaluation, and governed switching strategy for multiple agent/LLM providers
**Companion pages:** **JVM Agent Framework Options** (option analysis), **Border-Security Constraints and Pre-Funding Conditions** (12-month freeze, red lines), **Architecture Decision Records and Reference Architecture** (ADR-015 to ADR-018)

## Summary

Agent frameworks (Koog, LangChain4j, Spring AI, Google ADK, custom) evolve quickly, and none currently
meets OFFICIAL-SENSITIVE accreditation. Locking into one framework now is the wrong move. Instead, the
platform should depend on a thin `AgentProvider` abstraction, run multiple providers behind it, collect
security and performance metrics, and switch between them under governance.

Crucially, this strategy is consistent with the 12-month agent-framework freeze (red flag #3): during
the freeze, multi-provider evaluation runs in **lab and shadow mode only**, never in production and
never driving real-time alerts or rule deployment. The harness exists to *generate the evidence* needed
for an eventual, accreditation-backed go/no-go decision - not to put agents into the loop early.

## Detailed Response

### 1. Abstraction Layer Design

**1.1 `AgentProvider` interface.** The existing `AgentAnalysisService` (defined in
**Component Technology Choices**) remains the platform-facing facade; it delegates to an `AgentRouter`,
which selects a concrete `AgentProvider`. The provider abstraction is:

```java
public interface AgentProvider {
    String getProviderName();
    ProviderCapabilities capabilities();           // structured output? tool calling? max context?
    AgentAnalysis analyseFinding(FindingContext context);   // masked, bounded input only
    RuleSuggestion suggestRule(FeedbackPattern pattern);    // advisory; never auto-deploys
    ProviderHealth healthCheck();
}
```

Data classes (all masked/bounded by construction):

- `FindingContext` - masked finding metadata only (rule id/version, severity, counts, baseline,
  current value, candidate owner list, recent deployment refs). Declares `maxInputClassification`.
- `AgentAnalysis` - summary, impact, labelled root-cause hypotheses, suggested owner (from provided
  list only), confidence; declares `outputClassification`.
- `RuleSuggestion` - structured candidate change for human review (never executable directly).
- `ProviderHealth` - status, latency sample, error window, quota state.

**Security guarantees the interface must enforce** (consistent with **Governance, Security, and Scale**):

- Input is built only from masked, bounded finding metadata; the boundary rejects any payload above the
  rule's declared `max_input_classification` (see the `classification` block in the rule schema).
- No raw PII may cross the interface - enforced in the adaptor, not trusted to the provider.
- Output is classified and written to the separate agent-output audit zone, with prompt + response
  hashed into the audit chain.
- Every call is attributable (provider name, prompt version, input refs, reviewer decision).

**1.2 Adapters per framework.** Each framework (Koog, LangChain4j, Spring AI, custom) has an adaptor
implementing `AgentProvider`. Adaptor responsibilities:

- Translate `FindingContext` to/from the framework's API; abstract the LLM call.
- Enforce timeouts, retries, and structured-JSON output validation (reject free-form output).
- Apply the redaction/PII guard on both input and output.
- Capture metrics (latency, errors, tokens/cost) and emit audit events.
- Map framework errors to a uniform failure model for the circuit breaker.

**1.3 Decoupling.** The rule engine, findings store, and review UI depend only on
`AgentAnalysisService`, never on a framework. Providers are wired via dependency injection
(Spring/Quarkus) and selected at runtime by the router. Example configuration:

```yaml
agent:
  enabled: false            # OFF in production during the 12-month freeze
  mode: shadow              # shadow | lab | off  (never "live" during freeze)
  routing:
    strategy: highest_approval   # fixed | ab_test | highest_approval | lowest_latency | failover
    metrics_window_days: 7
  providers:
    - name: langchain4j-bedrock
      enabled: true
      weight: 50
    - name: spring-ai-azure
      enabled: true
      weight: 50
    - name: koog
      enabled: false        # lab only until assessed
  guards:
    max_input_classification: internal_operational_metadata
    require_structured_output: true
    reject_raw_pii: true
```

### 2. Evaluation Metrics and Data Collection

**2.1 Metrics.**

- **Approval rate** = confirmed agent suggestions / total reviewed agent suggestions.
- **False-positive rate** = rejected suggestions / total reviewed.
- **Latency** = p50/p95/p99 of `analyseFinding`/`suggestRule` round-trip.
- **Error rate** = failed calls / total calls (timeouts, invalid output, framework errors).
- **Provider self-confidence calibration** = difference between the provider's own confidence and the
  actual confirmed rate (measures over/under-confidence).

**2.2 Storage and cadence.**

- Per call: `dq_agent_analysis` (existing) records provider, prompt version, latency, error, self-confidence.
- New `dq_agent_provider_metric` table stores aggregated metrics per provider + window.
- Raw captured per call; aggregated hourly and daily. Routing reads the rolling 7-day aggregate.

**2.3 Reviewer feedback feeds the metrics.** A reviewer accepting a suggestion increments approval for
that provider; rejecting/editing decrements it and attributes the miss to a dimension via the existing
`reason_code` vocabulary: `wrong_owner` lowers owner-routing quality, `wrong_severity` lowers severity
calibration, `insufficient_evidence` lowers evidence quality. This is the same reason-code set used for
finding feedback, so provider quality is measured the same way as rule quality.

### 3. Strategy and Routing

**3.1 Selection strategies.**

| Strategy | How it works | When to use | Risk |
| --- | --- | --- | --- |
| Fixed | Always one provider | Stable baseline, audits | No comparison signal |
| A/B (weighted/random) | Split traffic by weight | Early evaluation | Variance; needs volume |
| Highest approval | Pick best rolling approval rate | Mature metrics exist | Can starve challengers; add small exploration % |
| Lowest latency | Pick fastest healthy provider | Latency-sensitive | May trade quality for speed |
| Failover | Primary, fall back on failure | Resilience | Secondary must be pre-approved |

**3.2 Runtime cost.** By default **one selected provider per finding** is called - calling all
providers per finding multiplies cost and PII-surface and is only done in **lab shadow comparison**,
never in production. Routing uses the rolling 7-day aggregate refreshed hourly, so a single bad hour
cannot whipsaw selection.

**3.3 `AgentRouter`.**

```text
AgentAnalysisService --> AgentRouter --> selects AgentProvider (by strategy + metrics + health)
                                   \--> records metrics, emits audit event
```

```java
public class AgentRouter {
    private final List<AgentProvider> providers;
    private final ProviderMetricsStore metrics;
    private final RoutingStrategy strategy;

    public AgentAnalysis analyse(FindingContext ctx) {
        AgentProvider p = strategy.select(healthy(providers), metrics.window(7));
        try {
            AgentAnalysis a = p.analyseFinding(ctx);
            metrics.recordSuccess(p, a);
            return a;
        } catch (Exception e) {
            metrics.recordError(p, e);
            return strategy.onFailure(p, ctx, e);   // failover to next approved provider
        }
    }
    private List<AgentProvider> healthy(List<AgentProvider> all) { /* circuit-breaker filter */ }
}
```

### 4. Switch Mechanisms

**4.1 Soft switch (blue-green / canary).** A new provider is promoted gradually, mirroring the rule
deployment progression in **Supporting Technical Detail**: start at 1% of advisory traffic, then 5%,
25%, 50%, 100%. At each step, check error rate, approval rate, and latency against thresholds before
advancing. Any breach halts promotion.

**4.2 Forced failover triggers.**

- Error rate > 5% (rolling 5 minutes).
- Average latency > 5s (SLA breach).
- LLM API quota exhausted or returning errors.
- A security trigger: published CVE or NCSC advisory affecting the provider/framework.

Failover may only switch to a provider that is **already accredited and approved** - never to an
unassessed one.

**4.3 Rollback and data consistency.** A circuit breaker plus health checks drive automatic reversion
to the previous provider. Data consistency is straightforward here because **agent output is advisory
only**: no findings are created by the agent and no rule is deployed by it. On rollback, in-flight
suggestions from the failing provider are marked `withdrawn`/`pending` rather than acted upon, so there
is no live rule or finding to undo.

### 5. Governance, Security, and Audit (UK Border-Security)

**5.1 Red lines.**

- **No autonomous switch into production.** During the freeze there is no production agent path at all.
  After the freeze, promotion of a new provider to production requires human (two-person) approval;
  only same-tier *failover to an already-approved provider* may be automatic.
- **Mandatory pre-production dwell:** a provider must run in sandbox and shadow for a minimum period
  (for example >= 1 week shadow) and hold accreditation evidence before any promotion proposal.
- **Audit trail:** provider changes emit an `AGENT_PROVIDER_SWITCHED` event into `dq_audit_event`
  (hash-chained), capturing from/to provider, trigger, actor (human or system), and metric snapshot.
- **Masked-data guarantee:** providers never receive raw PII. This is enforced in the adaptor boundary
  (input built from masked metadata, classification check, redaction guard), not delegated to the
  provider or framework.

**5.2 NCSC / UK government compliance.** Before any provider is eligible for production, it must supply:
penetration-test results, a vulnerability disclosure/management programme, and a support SLA. Without
these, a provider must not enter production - it may only be used for lab/shadow evaluation, because an
unaccredited dependency in an OFFICIAL-SENSITIVE path is itself a security risk.

**5.3 Consistency with the 12-month freeze.** The freeze means *no agent framework in production for 12
months*. It does **not** forbid evaluation. The clean split:

- **Forbidden:** any agent output driving production findings, alerts, or rule changes.
- **Allowed:** lab and shadow evaluation against masked, governed data to collect the approval/latency/
  error metrics that will inform the post-freeze decision.

This strategy is therefore the *mechanism that makes the freeze productive* rather than idle.

### 6. Concrete Implementation Steps (PoC to Production)

**6.1 Months 0-6:** define `AgentProvider`/`AgentRouter`; build adapters for two providers; stand up
the metrics pipeline (`dq_agent_provider_metric`); create an A/B lab/shadow environment with masked
data. No production path.

**6.2 Months 6-12:** collect metrics from shadow suggestions reviewed by real reviewers; tune the
routing strategy; report results and accreditation status to the governance board.

**6.3 12-month go/no-go decision table.**

| Metric / gate | Required to promote to production |
| --- | --- |
| Approval rate | > 70% on shadow suggestions |
| Error rate | < 1% |
| Latency (p95) | < 2s |
| Accreditation | Pen-test + VDP + support SLA evidence present |
| DPIA | Completed and signed |
| PII leakage incidents | Zero (verified by adaptor-boundary tests) |
| Governance | Two-person approval + AGENT_PROVIDER_SWITCHED audit in place |

All gates must pass; any single failure means no production promotion.

## Risks and Mitigations

| Risk | Mitigation |
| --- | --- |
| Framework lock-in | Thin `AgentProvider` interface; business logic never depends on a framework. |
| Unaccredited provider reaches production | Hard gate: pen-test + VDP + SLA + two-person approval before promotion; failover only to approved providers. |
| PII leakage to a provider | Masked-only `FindingContext`; classification check and redaction guard at the adaptor boundary; boundary tests. |
| Auto-switch instability (flapping) | Rolling 7-day metrics refreshed hourly; circuit breaker; promotion only via canary with human approval. |
| Cost blow-up from calling all providers | One provider per finding in production; multi-call only in lab shadow. |
| Freeze misread as "no agent work allowed" | Explicit lab/shadow-allowed vs production-forbidden split. |
| Provider self-confidence miscalibration | Track calibration (self-confidence vs actual confirmed rate) as a first-class metric. |
| Audit gaps on switch | `AGENT_PROVIDER_SWITCHED` hash-chained audit events with trigger and metric snapshot. |

## Recommended Next Steps

1. Define `AgentProvider`, `AgentRouter`, and the masked `FindingContext`/`AgentAnalysis`/`RuleSuggestion` contracts.
2. Add `dq_agent_provider_metric` to the data model and wire reviewer `reason_code` feedback into it.
3. Build two adapters (for example LangChain4j and Spring AI) for lab/shadow only.
4. Stand up the A/B lab/shadow harness on masked data; collect approval/latency/error metrics.
5. Adopt ADR-015 to ADR-018 (see **Architecture Decision Records and Reference Architecture**).
6. Report metrics and accreditation status to the governance board ahead of the 12-month decision.
