---
tags:
  - domain-1
  - agentic-architecture
  - orchestration
  - exam-overview
domain: "1 - Agentic Architecture & Orchestration"
weight: 27
---

# Domain 1 — Agentic Architecture & Orchestration

> **27% of exam** — the single largest domain on the Claude Certified Architect – Foundations exam.

This domain tests your ability to design, implement, and troubleshoot agentic systems built on Claude — from single-agent loops through multi-agent orchestration, session management, and deterministic enforcement hooks.

---

## Task Statements

| Task | Title | Description |
|------|-------|-------------|
| 1.1 | [[Agentic Loop Lifecycle]] | Understand the core agentic loop (request → stop_reason check → tool execution or return), how tool results feed back into conversation history, and the anti-patterns that break it. |
| 1.2 | [[Multi-Agent Orchestration]] | Design hub-and-spoke multi-agent systems where a coordinator decomposes tasks, delegates to subagents with isolated context, and aggregates results — including iterative refinement. |
| 1.3 | [[Subagent Configuration and Context Passing]] | Configure subagent definitions (system prompts, tool restrictions), pass context explicitly, manage sessions with `fork_session`, and spawn parallel subagents. |
| 1.4 | [[Multi-Step Workflows and Handoffs]] | Enforce multi-step workflow ordering through programmatic gates and hooks rather than relying solely on prompt instructions; design structured handoff protocols. |
| 1.5 | [[Hooks and Interception]] | Use PostToolUse hooks and tool-call interception for deterministic guarantees — data normalization, compliance enforcement, and deciding when hooks beat prompts. |
| 1.6 | [[Task Decomposition Strategies]] | Choose between fixed sequential pipelines (prompt chaining) and dynamic adaptive decomposition based on the nature of the task. |
| 1.7 | [[Session Management]] | Resume named sessions, fork sessions for divergent exploration, and decide when to start fresh with summaries vs. resume with stale context. |

---

## Key Themes Across the Domain

1. **Model-driven control flow** — Let the model decide when to stop (via `stop_reason`) rather than building brittle parsing logic around assistant text.
2. **Explicit context, not implicit inheritance** — Subagents do not inherit the coordinator's conversation history; context must be injected into every subagent prompt.
3. **Deterministic enforcement for critical paths** — Use hooks and programmatic gates for anything that *must* happen (compliance, safety, ordering), and prompt instructions for things that *should* happen (style, tone, strategy).
4. **Appropriate granularity** — Avoid both monolithic single-agent designs and overly narrow task decomposition that fragments reasoning.

---

## Related Domains

- [[Domain 2 - Tool Design & MCP Integration]] — Tools are the building blocks agents invoke inside the agentic loop.
- [[Domain 3 - Claude Code Configuration & Customization]] — Configuration files, hooks, and permissions that shape agent behaviour.
- [[Domain 4 - Prompt Engineering & System Design]] — System prompts, role definitions, and instruction design that guide agent reasoning.
- [[Domain 5 - Context Management & Optimization]] — Token budgets, summarization, and context window strategies that constrain what agents can see.
