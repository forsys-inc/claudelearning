---
tags:
  - domain-overview
  - claude-code
  - configuration
  - workflows
domain: "3 - Claude Code Configuration & Workflows"
weight: 20
---

# Domain 3 — Claude Code Configuration & Workflows

> **Exam weight: 20%** — This domain tests your ability to configure Claude Code for team and individual workflows, use slash commands and skills, apply path-specific rules, choose between plan mode and direct execution, iterate effectively, and integrate Claude Code into CI/CD pipelines.

---

## Domain Scope

Domain 3 focuses on the **practical usage and configuration** of Claude Code as an agentic coding tool. Unlike Domains 1–2 (which cover architecture and tool design), this domain is concerned with how you set up, customize, and operate Claude Code in day-to-day development and automated pipelines.

Key themes:
- **Configuration hierarchy** — understanding which instructions load, in what order, and who sees them
- **Extensibility** — custom commands, skills, and rules that tailor Claude Code to your project
- **Execution strategy** — knowing when to plan vs. act, and when to use subagents
- **Iteration** — communication patterns that converge on correct output quickly
- **Automation** — running Claude Code headlessly in CI/CD with structured output

---

## Task Statements

| Task | Title | Description |
|------|-------|-------------|
| 3.1 | [[CLAUDE.md Configuration Hierarchy]] | Configure and troubleshoot the CLAUDE.md hierarchy (user, project, directory levels), `@import` syntax, and `.claude/rules/` |
| 3.2 | [[Custom Slash Commands and Skills]] | Create project-scoped and user-scoped slash commands; build skills with `context: fork`, `allowed-tools`, and `argument-hint` |
| 3.3 | [[Path-Specific Rules]] | Write path-scoped rules in `.claude/rules/` with glob-pattern frontmatter for conditional activation |
| 3.4 | [[Plan Mode vs Direct Execution]] | Choose between plan mode and direct execution based on task complexity; use the Explore subagent |
| 3.5 | [[Iterative Refinement Techniques]] | Apply input/output examples, test-driven iteration, and the interview pattern to converge on correct output |
| 3.6 | [[CI/CD Integration]] | Run Claude Code non-interactively with `-p`, structured JSON output, session isolation, and CLAUDE.md context |

---

## How These Tasks Relate

```
┌─────────────────────────────────────────────────┐
│            Project Configuration                │
│  ┌──────────┐  ┌──────────┐  ┌──────────────┐  │
│  │ 3.1      │  │ 3.3      │  │ 3.2          │  │
│  │ CLAUDE.md│──│ Path     │──│ Commands &   │  │
│  │ Hierarchy│  │ Rules    │  │ Skills       │  │
│  └──────────┘  └──────────┘  └──────────────┘  │
└────────────────────┬────────────────────────────┘
                     │
        ┌────────────┼────────────┐
        ▼            ▼            ▼
  ┌──────────┐ ┌──────────┐ ┌──────────┐
  │ 3.4      │ │ 3.5      │ │ 3.6      │
  │ Plan vs  │ │ Iterative│ │ CI/CD    │
  │ Execute  │ │ Refine   │ │ Integr.  │
  └──────────┘ └──────────┘ └──────────┘
```

- **3.1 + 3.3** define *what instructions* Claude Code loads and *when*.
- **3.2** defines *reusable actions* that leverage those instructions.
- **3.4 + 3.5** govern *how you interact* with Claude Code during a session.
- **3.6** governs *how Claude Code runs unattended* in automation.

---

## Study Tips

1. **Hands-on practice** — Set up a sample repo with a multi-level CLAUDE.md hierarchy, a few slash commands, and path-specific rules. Run Claude Code and verify with `/memory` which files loaded.
2. **Scenario questions** — Expect exam questions that describe a team problem (e.g., "a new developer's Claude Code doesn't follow team conventions") and ask you to identify the root cause (e.g., instructions are in user-level config, not project-level).
3. **CI/CD flags** — Memorize the key flags: `-p`, `--output-format json`, `--json-schema`. Know when session isolation matters.

---

## Related Domains

- [[Domain 1 - Agentic Architecture]] — the model behavior that Claude Code orchestrates
- [[Domain 2 - Tool Design & MCP]] — MCP servers that Claude Code can connect to
- [[Domain 4 - Prompt Engineering]] — prompt techniques apply inside CLAUDE.md and slash commands
- [[Domain 5 - Context Management]] — context window considerations when loading rules and instructions
