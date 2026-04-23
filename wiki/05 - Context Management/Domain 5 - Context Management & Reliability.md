---
tags:
  - domain-5
  - context-management
  - reliability
  - exam-overview
domain: "5 - Context Management & Reliability"
weight: 15
---

# Domain 5 — Context Management & Reliability

> **15% of exam** — This domain tests your ability to manage context windows effectively, handle errors and ambiguity gracefully, and build systems that maintain accuracy and provenance as information flows through multi-step pipelines.

Context management is the connective tissue between architecture (Domain 1) and prompt engineering (Domain 4). Even well-designed agents with excellent prompts fail when critical information is lost mid-conversation, errors propagate silently, or outputs lack provenance. This domain focuses on the practical strategies that keep Claude-based systems reliable in production.

---

## Task Statements

| Task | Title | Description |
|------|-------|-------------|
| 5.1 | [[Context Window Management]] | Manage context window limits through progressive summarization, tool output trimming, position-aware ordering, and structured subagent outputs — while preserving critical transactional facts. |
| 5.2 | [[Escalation and Ambiguity Resolution]] | Design escalation triggers and ambiguity resolution strategies: explicit customer requests, policy gaps, multiple-match disambiguation, and few-shot escalation criteria. |
| 5.3 | [[Error Propagation in Multi-Agent Systems]] | Propagate errors with structured context, distinguish access failures from valid empty results, and enable coordinators to proceed with partial results and coverage annotations. |
| 5.4 | [[Large Codebase Exploration]] | Manage context degradation in extended coding sessions through scratchpad persistence, subagent delegation, structured state export, and `/compact` usage. |
| 5.5 | [[Human Review Workflows]] | Design human-in-the-loop review systems with stratified sampling, field-level confidence calibration, and routing logic that catches accuracy gaps masked by aggregate metrics. |
| 5.6 | [[Information Provenance and Uncertainty]] | Preserve source attribution through summarization and synthesis, handle conflicting data with annotations, enforce temporal metadata, and render findings with appropriate uncertainty markers. |

---

## Key Themes Across the Domain

1. **Information fidelity over brevity** — Summarization is useful but dangerous. Critical facts (amounts, dates, identifiers) must survive every compression step intact.
2. **Structured error handling** — Generic error messages destroy diagnostic value. Every error should carry type, context, partial results, and suggested alternatives.
3. **Appropriate human involvement** — Neither fully automated nor fully manual. Route to humans based on calibrated confidence, not gut feelings or aggregate accuracy.
4. **Provenance as a first-class concern** — Claims without sources, statistics without dates, and findings without uncertainty markers are liabilities in production systems.

---

## Related Domains

- [[Domain 1 - Agentic Architecture & Orchestration]] — Multi-agent patterns that generate the context management challenges addressed here.
- [[Domain 2 - Tool Design & MCP Integration]] — Tool output schemas directly affect context consumption and trimming strategies.
- [[Domain 3 - Claude Code Configuration & Customization]] — Configuration for `/compact`, hooks, and session persistence.
- [[Domain 4 - Prompt Engineering & System Design]] — System prompts that establish escalation criteria, output formats, and uncertainty handling.
