# Cerberos Data Assurance Intelligence Layer - Parent + 20 Child Page Discovery Pack

This folder contains the Cerberos Data Assurance Intelligence Layer Confluence upload material.

The only reader-facing deliverable is the Confluence page set in `confluence/`: one parent page plus 20 child pages.

## Combined Reference File

`cerberos-data-assurance-intelligence-layer-combined.md` is a generated long-form copy of the parent
page plus all child pages in one file. It is for private review, export, or continuity checks only;
the authoritative source is always the individual pages in `confluence/`.

It is produced by `build-combined.sh`. Regenerate it after editing any `confluence/*.md` page:

```bash
./build-combined.sh
```

Do not edit the combined file by hand, as changes will be overwritten on the next run.

## Final Confluence Page Set

Upload the `confluence/` folder content. The set is one parent page (item 1) plus 20 child pages (items 2-21):

1. `confluence/00-parent-page.md`
   - Main Confluence landing page.
2. `confluence/01-context-and-problem.md`
   - Child page for context, motivation, and problem statement.
3. `confluence/02-capability-and-architecture.md`
   - Child page for proposed capability and architecture.
4. `confluence/03-human-review-and-learning-loop.md`
   - Child page for feedback, learning, agent role, and AI-assisted suggestion lifecycle.
5. `confluence/04-governance-security-and-scale.md`
   - Child page for governance, safety, privacy, scale, alerting, and reporting.
6. `confluence/05-poc-roadmap-and-risks.md`
   - Child page for PoC, roadmap, risks, mitigations, and success criteria.
7. `confluence/05a-border-security-constraints-and-pre-funding-conditions.md`
   - Child page for OFFICIAL-SENSITIVE constraints, red/amber/green assessment, and pre-funding conditions.
8. `confluence/06-rule-types-data-model-and-examples.md`
   - Child page for rule categories, SQL examples, data model, and YAML examples.
9. `confluence/07-logging-observability-and-monitoring.md`
   - Child page for logging strategy, metrics, tracing, self-monitoring, and dashboards.
10. `confluence/08-jvm-agent-framework-options.md`
   - Child page for JVM agent framework options and PoC technology direction.
11. `confluence/08a-multi-provider-agent-framework-strategy.md`
   - Child page for the multi-provider agent abstraction, evaluation metrics, routing, switching, and governance.
12. `confluence/09-open-decisions-and-discovery-questions.md`
   - Child page for unresolved architecture, governance, delivery, and adoption questions.
13. `confluence/10-technology-selection-and-architecture-decision-report.md`
   - Child page for technology selection, trade-offs, candidate PoC stack, risks, and build-vs-buy.
14. `confluence/10a-component-technology-choices.md`
   - Child page for per-component technology options (backend, registry, query execution, sources, stores, UI, learning, agent, AWS, scheduling, observability, security, deployment).
15. `confluence/10b-architecture-decision-records.md`
   - Child page for the technology decision matrix, ADR-001 to ADR-018, and reference architecture.
16. `confluence/11-architecture-decision-summary.md`
   - Child page for the one-page summary of the technology and architecture decisions.
17. `confluence/12-executive-one-page-discovery-proposal.md`
   - Short discovery note suitable for initial senior stakeholder review (the first-share version).
18. `confluence/13-supporting-technical-detail.md`
   - Child page for ER diagram, rule YAML schema, sequence diagrams, confidence scoring, backtesting, notification templates, agent guardrails, and test scenarios.
19. `confluence/13a-glossary.md`
   - Child page for shared terminology across the discovery pack.
20. `confluence/14-s3-parquet-glue-athena-analytical-assurance-layer.md`
   - Child page for the analytical S3/Parquet/Glue/Athena assurance layer concept, governance, cost, and scope.
21. `confluence/14a-athena-analytical-dq-examples.md`
   - Child page for worked Athena query examples, reconciliation, backtesting, schema drift, and agent-layer relationship.

## Publication Guidance

This should remain a private discovery concept until the framing, evidence, and stakeholder route are clear.

Recommended Confluence positioning:

- Page title: `Cerberos Data Assurance Intelligence Layer - Initial Discovery Concept`
- Status: `Private working draft`
- Labels: `cerberos`, `data-assurance`, `architecture-discovery`, `private-draft`
- Restrictions: limit to the intended architecture or discovery audience first
- Tone: strategic, careful, discovery-focused, not a final proposal

The core message to preserve across all versions:

> This is not autonomous decision-making. It is governed data assurance supported by controlled automation, structured human feedback, and optional AI-assisted analysis.
