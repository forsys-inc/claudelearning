---
tags:
  - domain-4
  - prompt-engineering
  - structured-output
  - exam-overview
domain: "4 - Prompt Engineering & Structured Output"
weight: 20
---

# Domain 4 — Prompt Engineering & Structured Output

> **20% of exam** — the third-largest domain on the Claude Certified Architect – Foundations exam.

This domain tests your ability to design prompts that produce precise, structured, and reliable outputs from Claude — spanning review criteria design, few-shot prompting, JSON schema enforcement via tool use, validation loops, batch processing, and multi-instance review architectures.

---

## Task Statements

| Task | Title | Description |
|------|-------|-------------|
| 4.1 | [[Explicit Criteria for Precision]] | Write explicit, specific review criteria that reduce false positives and build developer trust — including severity levels, category disabling, and concrete behavioral definitions. |
| 4.2 | [[Few-Shot Prompting]] | Use few-shot examples as the most effective technique for consistently formatted output, ambiguous-case handling, generalization, and hallucination reduction. |
| 4.3 | [[JSON Schemas and Tool Use]] | Leverage `tool_use` with JSON schemas for guaranteed schema-compliant output, including `tool_choice` modes, nullable fields, enum patterns, and format normalization. |
| 4.4 | [[Validation and Retry Loops]] | Design retry-with-error-feedback loops, distinguish when retries are effective vs. futile, and build self-correction flows with semantic validation. |
| 4.5 | [[Batch Processing]] | Use the Message Batches API for cost-efficient, latency-tolerant workloads — including custom_id correlation, failure resubmission, and prompt refinement strategies. |
| 4.6 | [[Multi-Instance Review Architectures]] | Overcome self-review limitations by using independent Claude instances, multi-pass architectures, and confidence-based routing. |

---

## Key Themes Across the Domain

1. **Precision over recall in review tasks** — Explicit criteria with concrete examples outperform vague instructions every time. High false-positive rates destroy trust faster than missed issues.
2. **Structure through tooling, not prompting** — `tool_use` with JSON schemas eliminates syntax errors; prompting alone cannot guarantee valid JSON.
3. **Validation is multi-layered** — Schema compliance (handled by tool use) is distinct from semantic correctness (handled by validation loops and self-correction fields).
4. **Cost-latency tradeoffs** — Batch API cuts costs 50% but sacrifices latency guarantees; choose based on whether the workflow is blocking or background.
5. **Independent review beats self-review** — A second Claude instance reviewing output is more effective than asking the same instance to check its own work.

---

## Related Domains

- [[Domain 1 - Agentic Architecture & Orchestration]] — Agentic loops provide the execution context where structured output and validation happen.
- [[Domain 2 - Tool Design & MCP Integration]] — Tool definitions are the vehicle for JSON schema enforcement.
- [[Domain 3 - Claude Code Configuration & Customization]] — Hooks and configuration shape how prompts are constructed and validated.
- [[Domain 5 - Context Management & Optimization]] — Token budgets constrain prompt design, especially for few-shot examples and batch processing.
