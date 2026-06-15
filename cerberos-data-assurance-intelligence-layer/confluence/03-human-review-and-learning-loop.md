# Human Review and Learning Loop

**Parent page:** Cerberos Data Assurance Intelligence Layer - Initial Discovery Concept  
**Status:** Private working draft

## Why Human Review Matters

The most important part of the concept is not automation alone. It is the learning loop created by human review.

Data quality findings often require domain context. A finding may be a genuine issue, a false positive, a known exception, a duplicate, a low-impact condition, or a sign that the business rule has been misunderstood. Capturing that distinction structurally is what allows the assurance capability to improve.

## Review UI

A finding screen should show:

- Finding summary.
- Rule name and version.
- Severity.
- Confidence score.
- Affected count.
- Source system.
- Domain.
- Time window.
- Baseline comparison.
- Trend.
- Masked sample evidence.
- Related previous findings.
- Related incidents.
- Suggested owner/team.
- Suggested root cause.
- Recommended action.

## Structured Feedback

The decision panel should allow:

- Confirm issue.
- False positive.
- Known acceptable exception.
- Duplicate.
- Needs investigation.
- Wrong severity.
- Wrong owner.
- Insufficient evidence.
- Business rule misunderstood.
- Low impact.

Additional fields should include:

- Comment.
- Corrected severity.
- Corrected owner/team.
- Exception reason.
- Link to incident.
- Link to runbook.
- Create candidate exception.
- Suggest rule refinement.

Structured feedback is more useful than free text alone because it can be measured, trended, audited, and used to improve future checks.

False positive and known exception should be separated clearly:

- **False positive** means the rule is wrong or the evidence does not support the issue.
- **Known exception** means the finding is technically true, but accepted under a documented business, source, or operational condition.

## Learning Loop

```text
Finding
  -> Human decision
  -> Structured feedback
  -> Feedback pattern
  -> Learning insight
  -> Suggested improvement
  -> Human approval
  -> Improved rule
```

## What the System Learns

The learning engine may identify:

- Rules with high false positive rates.
- Sources where a rule is reliable.
- Sources where a rule is unreliable.
- Known exception patterns.
- Incorrect severity classification.
- Incorrect owner routing.
- Evidence gaps.
- Hidden patterns across source, event type, transformation version, provider, field, status, or time window.
- Candidate new rules.

Example:

> Most confirmed findings are concentrated in JOURNEY_UPDATE events from Source C after transformation version v2.14.

This type of insight does not make a decision. It gives teams a better investigation starting point.

## Agent / Copilot Role

Agents or Copilot-style capabilities may support:

- Summarising findings.
- Drafting incident descriptions.
- Suggesting likely root causes.
- Explaining potential impact.
- Suggesting owners.
- Grouping related findings.
- Analysing feedback patterns.
- Drafting rule refinements.
- Drafting candidate rules.
- Generating daily or weekly quality reports.
- Creating runbook drafts.

Agents must not:

- Write production data.
- Run uncontrolled queries.
- Make operational or security decisions.
- Expose raw PII.
- Automatically remediate.
- Automatically deploy rules.
- Bypass governance.
- Replace human review.

## AI-Assisted Suggestion Lifecycle

This is the heart of the concept. The AI/agent-assisted layer should produce suggestions only,
never decisions. Every reviewer action on a suggestion becomes feedback for future learning, so the
platform learns not only from data quality findings, but also from the quality of its own suggestions.

A suggestion may be:

- Accepted.
- Rejected.
- Edited.
- Downgraded.
- Escalated.
- Marked as not enough evidence.
- Converted into a candidate rule.
- Converted into a known exception.
- Linked to an incident or runbook.

For each suggestion, the system should track:

- Suggestion type.
- Suggested owner.
- Suggested severity.
- Suggested root cause.
- Reviewer decision.
- Reviewer correction.
- Reason code.
- Confidence before review.
- Confidence after review.
- Whether the suggestion led to a rule refinement.

```text
AI/agent suggestion
  -> human review (accept / reject / edit / downgrade / escalate)
  -> reviewer correction + reason code
  -> suggestion feedback pattern
  -> confidence adjustment (before vs after review)
  -> improved routing, severity, evidence, or rule refinement
  -> human approval
```

This closes the loop twice: once on the data quality finding, and once on the suggestion itself. Over
time the platform can tell which suggestion types are reliable, which need more evidence, and which
should be suppressed. A worked accept/reject/edit example is in the child page
**Rule Types, Data Model, and Examples**.

## Key Principle

This follows the safe operating model defined on the parent page
**Cerberos Data Assurance Intelligence Layer - Initial Discovery Concept**, with the detailed
governance, security, and privacy controls captured in **Governance, Security, and Scale**.

Supporting rule examples and the conceptual data model are captured in the child page **Rule Types, Data Model, and Examples**.

Framework options for implementing the optional agent/Copilot-assisted layer are captured in the child page **JVM Agent Framework Options**.
