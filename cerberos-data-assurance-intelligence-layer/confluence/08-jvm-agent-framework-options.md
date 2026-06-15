# JVM Agent Framework Options

**Parent page:** Cerberos Data Assurance Intelligence Layer - Initial Discovery Concept  
**Status:** Private working draft  
**Purpose:** Technology option analysis for the optional agent/Copilot-assisted layer

## Why This Page Exists

The main architecture concept intentionally focuses on assurance, governance, human review, and the learning loop. However, if the capability is implemented in the JVM ecosystem, the agent framework choice becomes an important architecture decision.

This page compares realistic JVM-oriented options for the optional agent/Copilot-assisted layer. It should not be read as a final technology selection. The strategy for abstracting over several providers and switching between them under governance is covered in the child page **Multi-Provider Agent Framework Strategy**.

> **Important:** No agent framework is selected or used in production during the 12-month freeze (see **Border-Security Constraints and Pre-Funding Conditions**, red flag #3). The options below are assessed only in lab/shadow mode to gather evidence for a later, accreditation-backed decision.

The agent layer remains bounded in all options:

- It analyses controlled findings, structured feedback, masked evidence, and approved metadata.
- It does not run uncontrolled production queries.
- It does not write production data.
- It does not make operational or security decisions.
- It does not deploy rule changes without human approval.

## Decision Criteria

The framework choice should be assessed against:

- Fit with the existing Cerberos application stack.
- Java versus Kotlin preference.
- Spring Boot integration needs.
- LLM provider flexibility.
- Tool-calling maturity.
- Graph or workflow orchestration support.
- Human-in-the-loop support.
- Structured output support.
- Observability and tracing.
- Persistence and replay capability.
- RAG and vector store integration.
- Governance friendliness.
- Community maturity and documentation.
- Cloud alignment, especially if GCP or Vertex AI is strategic.
- Ease of delivering a small PoC quickly.

## Option 1 - Koog

Koog is a JetBrains JVM-native framework for building AI agents in Java and Kotlin. It is a strong fit where the agent layer needs more than a simple request-response LLM wrapper.

Strengths:

- JVM-native and Kotlin-friendly.
- Supports structured agent workflows.
- Suitable for graph-style orchestration and multi-step agent behaviour.
- Good fit for modelling the assurance learning loop as explicit workflow stages.
- OpenTelemetry support is available for tracing agent execution.
- Spring integration exists, including a Spring AI bridge.

Potential concerns:

- Newer ecosystem than LangChain4j.
- Team familiarity may be lower.
- Spring AI integration should be treated carefully if still beta in the version adopted.

Best fit:

- Cerberos is Kotlin-heavy.
- The agent layer needs graph-based workflow orchestration.
- The learning loop is expected to become a first-class platform workflow.
- Observability and traceability of agent execution are important.

## Option 2 - Spring AI + Koog Hybrid

This option uses Spring AI for model access, vector store integration, memory/RAG patterns, and Spring ecosystem observability, while using Koog for agent orchestration.

Strengths:

- Natural fit if Cerberos is already Spring Boot-based.
- Preserves Spring idioms for configuration, dependency injection, observability, and integration.
- Allows Koog to provide higher-level orchestration on top.
- Keeps model and vector store integration closer to the Spring ecosystem.
- Supports a cleaner separation between AI infrastructure and agent workflow.

Potential concerns:

- More moving parts than using one framework.
- Requires clear ownership boundaries between Spring AI concerns and Koog orchestration.
- The integration version and maturity should be validated during discovery.

Best fit:

- Cerberos is primarily Spring Boot.
- The team wants Spring-native model access and observability.
- The agent layer needs more structured orchestration than Spring AI alone.
- The platform may later need richer workflow modelling for learning, triage, and rule suggestion.

## Option 3 - LangChain4j

LangChain4j is a mature Java library for building LLM-powered JVM applications. It has broad provider support, Spring Boot integration, tool calling, RAG support, and a large community footprint.

Strengths:

- Mature Java ecosystem option.
- Broad LLM and vector store integrations.
- Good documentation and community familiarity.
- Strong for tool calling, RAG, and AI-service style abstractions.
- Fast path for a simple PoC.

Potential concerns:

- Less naturally suited to explicit graph-based assurance workflow modelling than Koog.
- Better fit for simpler "single agent plus tools" patterns than complex governed learning loops.
- If the platform later requires richer orchestration, migration or additional orchestration code may be needed.

Best fit:

- The PoC agent role is deliberately minimal.
- The initial use cases are finding summarisation, owner suggestion, and root cause hints.
- Speed of delivery and broad provider support matter more than graph orchestration.
- The team wants to validate value before committing to a heavier orchestration model.

## Option 4 - Google ADK for Java

Google Agent Development Kit for Java is a Google-backed framework for building, evaluating, and deploying Java agents. It is most relevant where GCP, Vertex AI, and agent-to-agent patterns are strategically important.

Strengths:

- Strong alignment with Google Cloud and Vertex AI.
- Supports more sophisticated agent development patterns.
- Relevant where multi-agent workflows are expected.
- May be attractive if A2A or Google ecosystem integration becomes important.

Potential concerns:

- Strongest fit may depend on GCP/Vertex AI adoption.
- May be less natural if Cerberos is not already aligned to Google Cloud.
- Could be more platform-shaping than needed for a narrow PoC.

Best fit:

- Cerberos is GCP/Vertex AI-aligned.
- Multi-agent architecture is a serious near-term requirement.
- Agent-to-agent integration is strategically important.
- Google Cloud operational tooling is already part of the target platform.

## Option 5 - Custom Lightweight LLM Client

This option avoids an agent framework initially and calls an approved LLM API directly through a small internal service wrapper.

Strengths:

- Maximum control.
- Minimal dependency footprint.
- Low framework lock-in.
- Easier to govern for very narrow use cases.
- Good fit if the agent role is only summarisation and suggestion drafting.

Potential concerns:

- The team must build orchestration, retries, structured output validation, tracing, safety controls, prompt/version management, evaluation, and testing patterns.
- Can become expensive to maintain if agent workflows become more complex.
- May recreate framework capabilities over time.

Best fit:

- The PoC keeps AI assistance extremely limited.
- Governance requires very explicit control over every LLM interaction.
- The first version only needs summarisation, report drafting, and candidate wording.

## Recommendation

For this concept, the safest recommendation is staged rather than absolute.

All agent framework references should be validated during discovery before being treated as architecture recommendations.

| Scenario | Recommended Direction |
| --- | --- |
| Cerberos is Spring Boot-based | Assess Spring AI + Koog hybrid |
| Cerberos is Kotlin-heavy | Assess Koog |
| GCP/Vertex AI is strategic | Google ADK for Java should be assessed |
| PoC agent role is minimal | LangChain4j or custom lightweight wrapper |
| Maximum control and minimal dependency are required | Custom lightweight LLM client |

## Potential Direction

If Cerberos is Spring Boot-based, **Spring AI + Koog hybrid** is a potential direction to assess.

Rationale:

- Spring AI can handle Spring-native model access, vector stores, RAG patterns, observability, and integration.
- Koog can handle the higher-level orchestration needed for the learning loop.
- The split maps well to this platform concept: Spring AI as AI infrastructure, Koog as governed agent workflow orchestration.

If Cerberos is not Spring Boot-heavy but is Kotlin/JVM-heavy, **Koog alone** may be a stronger architectural fit.

If the PoC keeps the agent role very small, **LangChain4j** is a pragmatic fast-start option, provided the implementation keeps a clean internal abstraction so the framework can be replaced later if the orchestration requirement grows.

## PoC Guidance

For the first PoC, avoid overbuilding the agent layer. The PoC should prove the assurance loop first:

1. Approved read-only rules.
2. Findings.
3. Human review.
4. Structured feedback.
5. Basic confidence scoring.
6. One useful rule refinement suggestion.
7. Optional AI-generated finding summary.

The agent framework should support this without becoming the centre of the architecture.

Recommended PoC approach:

- Define an internal `AgentAssistanceService` boundary.
- Keep prompts and model calls versioned.
- Require structured JSON output for suggestions.
- Log model, prompt version, input references, output, and reviewer decision.
- Do not expose raw PII to the agent layer.
- Do not allow direct production query access.
- Keep all rule changes behind human approval.

## Architecture Decision to Defer

The first discovery should not choose a framework purely by feature list. It should validate:

- Existing Cerberos runtime stack.
- Spring Boot and Kotlin usage.
- Cloud and LLM provider constraints.
- Security and governance constraints.
- Whether graph-based orchestration is needed in the PoC or only later.
- Whether the agent layer is genuinely central or only supportive.

Until those facts are known, the most balanced position is:

> Assess Spring AI + Koog if Cerberos is Spring Boot-based and the learning loop becomes a real workflow. Use LangChain4j or a custom lightweight wrapper if the PoC keeps the AI role limited to summarisation and suggestion drafting.

## References for Technology Validation

- Koog overview: https://docs.koog.ai/
- Koog Java/Spring positioning: https://blog.jetbrains.com/ai/2026/03/koog-comes-to-java/
- Koog Spring AI integration: https://docs.koog.ai/spring-ai-integration/
- Koog OpenTelemetry example: https://docs.koog.ai/examples/OpenTelemetry/
- LangChain4j documentation: https://docs.langchain4j.dev/
- Spring AI observability: https://docs.spring.io/spring-ai/reference/observability/index.html
- Google ADK: https://adk.dev/
- Google ADK for Java: https://github.com/google/adk-java
