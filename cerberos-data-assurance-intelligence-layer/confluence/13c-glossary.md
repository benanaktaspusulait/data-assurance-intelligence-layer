# Glossary

**Parent page:** Cerberos Data Assurance Intelligence Layer - Initial Discovery Concept  
**Status:** Private working draft  
**Purpose:** Shared terminology for the Cerberos Data Assurance Intelligence Layer discovery pack

| Term | Meaning |
| --- | --- |
| Data Quality | Correctness, completeness, freshness, consistency, and usability of data. |
| Data Assurance | Governed capability to detect, review, evidence, learn from, and improve confidence in data quality over time. |
| Finding | A reviewable output from an approved rule execution, with severity, evidence, status, and ownership. |
| Rule | Approved definition of a check, including source, query template, threshold, owner, schedule, and governance metadata. |
| Rule Run | One execution of a specific rule version against an approved source and time window. |
| Feedback Pattern | Aggregated reviewer decisions that indicate rule reliability, exceptions, owner routing, or evidence quality. |
| False Positive | A finding where the rule is wrong or the evidence does not support the issue. |
| Known Exception | A technically true finding accepted under a documented business, source, or operational condition. |
| Backtesting | Running a candidate or changed rule against historical data in shadow mode before approval. |
| Shadow Mode | Execution mode that produces evaluation results without live findings, notifications, or remediation. |
| SafeQueryExecutor | Mandatory execution layer that enforces approved templates, read-only access, limits, masking, cost controls, and audit. |
| Operational DB | Database supporting live application behaviour or request paths. It should not be used directly for PoC checks. |
| Production DB | Live production database. This discovery explicitly excludes writes and uncontrolled direct querying. |
| Replica DB | Read-only replicated source suitable for bounded recent operational checks. |
| Analytical Source | Governed analytical dataset or query layer, such as S3/Parquet/Athena, used for aggregate or historical assurance checks. |
| Raw Zone | S3 zone containing source-aligned or minimally transformed data, usually more sensitive and tightly restricted. |
| Curated Zone | S3 zone containing governed, structured, partitioned datasets suitable for repeatable analytical checks. |
| Athena Workgroup | Athena control boundary for result location, encryption, query limits, cost controls, and execution isolation. |
| Agent Layer | Optional support layer that analyses controlled findings and feedback, never direct data sources. |
| Copilot | User-facing assistant experience, if approved, limited to summarisation, triage support, and draft suggestions. |
| LLM | Large language model used only behind approved enterprise controls and only with masked, bounded context. |
| AgentAnalysisService | Internal interface for optional agent support, allowing the PoC to run without an LLM dependency. |
