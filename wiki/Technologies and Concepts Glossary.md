---
tags: [reference, glossary]
---

# Technologies and Concepts Glossary

Alphabetical reference of all technologies and concepts tested on the Claude Certified Architect – Foundations exam.

---

## B

### Batch Processing → [[Batch Processing]]
The **Message Batches API** provides 50% cost savings with up to 24-hour processing. Use `custom_id` to correlate request/response pairs. Appropriate for latency-tolerant workloads (overnight reports, weekly audits). Does NOT support multi-turn tool calling within a single request.

### Built-in Tools → [[Built-in Tools]]
Six core tools available in Claude Code and the Agent SDK:
- **Read** — load file contents
- **Write** — create/overwrite files
- **Edit** — targeted text replacement using unique anchors
- **Bash** — execute shell commands
- **Grep** — search file contents by pattern
- **Glob** — find files by name/path patterns

---

## C

### Claude Agent SDK → [[Domain 1 - Agentic Architecture & Orchestration]]
Framework for building agentic applications. Key concepts: `AgentDefinition` (descriptions, system prompts, tool restrictions), agentic loops (`stop_reason` handling), hooks (`PostToolUse`, tool call interception), subagent spawning via `Task` tool, `allowedTools` configuration.

### Claude API → [[JSON Schemas and Tool Use]]
The Messages API for interacting with Claude. Key features: `tool_use` with JSON schemas for structured output, `tool_choice` options (`"auto"`, `"any"`, forced selection), `stop_reason` values (`"tool_use"`, `"end_turn"`), `max_tokens`, system prompts.

### Claude Code → [[Domain 3 - Claude Code Configuration & Workflows]]
Anthropic's CLI tool for AI-assisted development. Key features: CLAUDE.md configuration hierarchy, `.claude/rules/` with path-scoping, `.claude/commands/` for slash commands, `.claude/skills/` with SKILL.md frontmatter, plan mode, direct execution, `/memory`, `/compact`, `--resume`, `fork_session`, Explore subagent.

### Claude Code CLI → [[CI-CD Integration]]
Command-line flags for automation: `-p` / `--print` for non-interactive mode, `--output-format json` for structured output, `--json-schema` for schema enforcement in CI contexts.

### Confidence Scoring → [[Human Review Workflows]]
Field-level confidence scores output by the model, calibrated using labeled validation sets. Used for routing review attention and implementing stratified sampling for error rate measurement.

### Context Window Management → [[Context Window Management]]
Strategies for managing token budgets: progressive summarization (with risks of losing numerical details), "lost in the middle" effect, trimming verbose tool outputs, extracting structured facts into persistent blocks, scratchpad files for extended sessions.

---

## F

### Few-Shot Prompting → [[Few-Shot Prompting]]
Providing 2-4 targeted examples in the prompt to achieve consistent output format, demonstrate ambiguous-case handling, and enable generalization to novel patterns. Most effective technique for format consistency when instructions alone produce inconsistent results.

### Fork Session → [[Session Management]]
`fork_session` creates independent exploration branches from a shared analysis baseline. Useful for comparing approaches (e.g., two refactoring strategies) without contaminating either path.

---

## H

### Hooks → [[Hooks and Interception]]
Agent SDK mechanism for intercepting tool calls and results. `PostToolUse` hooks normalize data after tool execution. Tool call interception hooks enforce compliance (e.g., blocking refunds over thresholds). Provide deterministic guarantees where prompt instructions offer only probabilistic compliance.

---

## J

### JSON Schema → [[JSON Schemas and Tool Use]]
Schema definition language used with `tool_use` for structured output. Key concepts: required vs optional fields, enum types, nullable fields (prevent hallucination), "other" + detail string patterns for extensible categories, strict mode for syntax error elimination.

---

## M

### Message Batches API → [[Batch Processing]]
50% cost savings, up to 24-hour processing window. Uses `custom_id` for request/response correlation. No multi-turn tool calling support. Polling for completion status.

### Model Context Protocol (MCP) → [[Domain 2 - Tool Design & MCP Integration]]
Protocol for connecting Claude to external systems. Components: MCP servers (tools + resources), `.mcp.json` for project-level config, `~/.claude.json` for user-level config, `isError` flag for error signaling, environment variable expansion for credentials.

### MCP Resources → [[MCP Server Configuration]]
Content catalogs exposed by MCP servers (e.g., issue summaries, documentation hierarchies, database schemas). Reduces exploratory tool calls by giving agents visibility into available data.

### MCP Tools → [[Tool Interface Design]]
Action endpoints exposed by MCP servers. Tool descriptions are the primary mechanism LLMs use for selection — detailed, differentiated descriptions are critical for reliable tool routing.

---

## P

### Plan Mode → [[Plan Mode vs Direct Execution]]
Claude Code mode for complex tasks: large-scale changes, multiple valid approaches, architectural decisions, multi-file modifications. Enables safe exploration before committing. Contrast with direct execution for simple, well-scoped changes.

### Prompt Chaining → [[Task Decomposition Strategies]]
Sequential task decomposition: break complex tasks into focused passes (e.g., per-file analysis → cross-file integration). Appropriate for predictable multi-aspect reviews.

### Pydantic → [[Validation and Retry Loops]]
Python schema validation library. Used for semantic validation beyond JSON schema syntax (which `tool_use` already guarantees). Enables validation-retry loops with specific error feedback.

---

## S

### Session Management → [[Session Management]]
Session resumption with `--resume <session-name>`. Named sessions for continuing across work sessions. Context considerations: resuming with mostly-valid context vs starting fresh with injected summaries when prior tool results are stale.

### Structured Error Responses → [[Structured Error Responses]]
MCP tool error pattern: `isError` flag, `errorCategory` (transient/validation/business/permission), `isRetryable` boolean, human-readable descriptions. Enables agent recovery decisions.

---

## T

### Task Tool → [[Subagent Configuration and Context Passing]]
The mechanism for spawning subagents in the Agent SDK. Coordinator's `allowedTools` must include `"Task"`. Subagents receive context explicitly in their prompt — no automatic inheritance.

### Tool Choice → [[Tool Distribution and Tool Choice]]
API parameter controlling tool selection: `"auto"` (model decides, may return text), `"any"` (must call a tool, model picks which), forced `{"type": "tool", "name": "..."}` (must call specific tool).

---

## Related

- [[In-Scope vs Out-of-Scope Topics]]
- [[00 - Home]]
