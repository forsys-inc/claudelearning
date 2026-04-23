---
tags: [home, index]
---

# Claude Certified Architect – Foundations Study Wiki

Welcome to your study guide for the **Claude Certified Architect – Foundations** certification exam.

## Exam Overview

- **Format:** Multiple choice (1 correct, 3 distractors)
- **Passing Score:** 720/1000 (scaled)
- **Scenarios:** 4 of 6 scenarios presented per exam
- **No penalty for guessing** — answer every question

## Domains & Weights

| Domain | Weight | Folder |
|--------|--------|--------|
| [[Domain 1 - Agentic Architecture & Orchestration]] | **27%** | `01 - Agentic Architecture/` |
| [[Domain 2 - Tool Design & MCP Integration]] | **18%** | `02 - Tool Design & MCP/` |
| [[Domain 3 - Claude Code Configuration & Workflows]] | **20%** | `03 - Claude Code Configuration/` |
| [[Domain 4 - Prompt Engineering & Structured Output]] | **20%** | `04 - Prompt Engineering/` |
| [[Domain 5 - Context Management & Reliability]] | **15%** | `05 - Context Management/` |

## Exam Scenarios

The exam uses **scenario-based questions**. Each scenario frames a set of questions around a realistic production context. You'll see 4 of these 6:

1. **Customer Support Resolution Agent** — Agentic loops, tool design, escalation (Domains 1, 2, 5)
2. **Code Generation with Claude Code** — Configuration, workflows, plan mode (Domains 3, 5)
3. **Multi-Agent Research System** — Orchestration, context passing, synthesis (Domains 1, 2, 5)
4. **Developer Productivity with Claude** — Built-in tools, MCP, codebase exploration (Domains 2, 3, 1)
5. **Claude Code for Continuous Integration** — CI/CD pipelines, structured output (Domains 3, 4)
6. **Structured Data Extraction** — JSON schemas, validation, batch processing (Domains 4, 5)

## Learning Path

### Phase 1: Foundations (Start Here)
1. [[Agentic Loop Lifecycle]] — understand stop_reason, tool_use flow
2. [[Built-in Tools]] — Read, Write, Edit, Bash, Grep, Glob
3. [[CLAUDE.md Configuration Hierarchy]] — user/project/directory levels
4. [[JSON Schemas and Tool Use]] — structured output basics

### Phase 2: Architecture & Design
5. [[Multi-Agent Orchestration]] — coordinator-subagent patterns
6. [[Tool Interface Design]] — descriptions, naming, boundaries
7. [[MCP Server Configuration]] — project vs user scope
8. [[Few-Shot Prompting]] — examples for consistency

### Phase 3: Advanced Patterns
9. [[Hooks and Interception]] — PostToolUse, policy enforcement
10. [[Task Decomposition Strategies]] — chaining vs dynamic decomposition
11. [[Context Window Management]] — trimming, summarization, position effects
12. [[Validation and Retry Loops]] — error feedback, self-correction

### Phase 4: Production & Operations
13. [[CI/CD Integration]] — -p flag, --output-format json, review pipelines
14. [[Batch Processing]] — Message Batches API, custom_id
15. [[Human Review Workflows]] — confidence calibration, stratified sampling
16. [[Error Propagation in Multi-Agent Systems]] — structured errors, recovery

### Phase 5: Practice & Review
17. [[Sample Questions]] — work through all sample questions
18. [[Exercise 1 - Multi-Tool Agent]]
19. [[Exercise 2 - Team Workflow Configuration]]
20. [[Exercise 3 - Data Extraction Pipeline]]
21. [[Exercise 4 - Multi-Agent Research Pipeline]]

## Key Technologies

- **Claude Agent SDK** — Agent definitions, agentic loops, hooks, subagents
- **Model Context Protocol (MCP)** — Servers, tools, resources, .mcp.json
- **Claude Code** — CLAUDE.md, .claude/rules/, commands, skills, plan mode
- **Claude Code CLI** — `-p`, `--output-format json`, `--json-schema`, `--resume`
- **Claude API** — `tool_use`, `tool_choice`, `stop_reason`, `max_tokens`
- **Message Batches API** — 50% cost savings, 24-hour window, `custom_id`
- **JSON Schema / Pydantic** — Validation, nullable fields, enum patterns

## Quick Reference

- [[Technologies and Concepts Glossary]]
- [[In-Scope vs Out-of-Scope Topics]]
