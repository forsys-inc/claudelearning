---
tags:
  - domain-1
  - task-1-3
  - subagent
  - context-passing
  - task-tool
  - fork-session
  - parallel-agents
domain: "1 - Agentic Architecture & Orchestration"
task: "1.3"
---

# Subagent Configuration and Context Passing

> **Task 1.3** — Configure subagent definitions, pass context explicitly, manage fork-based sessions, and spawn parallel subagents.

---

## The Task Tool

In Claude Code's multi-agent framework, subagents are spawned using the **Task tool**. The Task tool lets the coordinator create a new agent instance with its own conversation, tools, and instructions.

Critical requirement: if a subagent needs to spawn its own subagents, its `allowedTools` list **must include `"Task"`**. Without this, nested delegation is impossible.

---

## Context Must Be Explicitly Provided

Subagents start with an empty conversation history. They do not inherit the coordinator's messages, tool results, or accumulated knowledge. Every piece of information a subagent needs must be included in its prompt.

This means:
- If the coordinator has already gathered data, it must paste relevant findings into the subagent's prompt.
- If subagent B depends on subagent A's output, the coordinator must extract A's results and inject them into B's prompt.
- Summaries and structured data are preferred over raw conversation dumps to conserve tokens.

---

## AgentDefinition

An `AgentDefinition` specifies how a subagent behaves:

| Field | Purpose |
|-------|---------|
| `name` | Identifier for the subagent type |
| `description` | What the subagent does — helps the coordinator decide when to use it |
| `systemPrompt` | Instructions, persona, constraints for the subagent |
| `allowedTools` | Whitelist of tools the subagent can invoke |
| `model` | (Optional) Model override — use a smaller model for simple subtasks |

Tool restrictions are critical for security and cost: a research subagent should not have access to `process_refund`, and a data-fetching subagent may not need `web_search`.

---

## Fork-Based Session Management

`fork_session` creates a copy of the current session state, allowing divergent exploration without polluting the original session. Use cases:

- **Strategy comparison** — Fork the session to try approach A, fork again to try approach B, compare results.
- **Speculative exploration** — Fork to investigate a hypothesis; if it fails, the original session is unaffected.
- **Parallel deep-dives** — Fork for each research direction, then merge findings back.

Forked sessions inherit the conversation history at the point of forking but diverge from there.

---

## Parallel Subagent Spawning

When multiple subagents are independent (no dependency between them), the coordinator can spawn them in a single response by including multiple `Task` tool calls. This is analogous to `Promise.all` — all subagents run concurrently, and the coordinator processes all results when they return.

---

## Examples

### Example 1 — AgentDefinition Configuration

```json
{
  "agents": [
    {
      "name": "code_reviewer",
      "description": "Reviews code changes for correctness, style, and security vulnerabilities. Use for PR reviews and code audits.",
      "systemPrompt": "You are a senior code reviewer. For each file you review:\n1. Check for logical errors and edge cases.\n2. Identify security vulnerabilities (injection, auth bypass, data exposure).\n3. Evaluate code style and maintainability.\n4. Suggest concrete improvements with code examples.\n\nReturn findings as structured JSON with severity levels: critical, warning, info.",
      "allowedTools": ["read_file", "grep_search", "list_directory"],
      "model": "claude-sonnet-4-20250514"
    },
    {
      "name": "test_generator",
      "description": "Generates unit and integration tests for code changes. Use after code review identifies gaps in test coverage.",
      "systemPrompt": "You are a test engineering specialist. Given source code and review findings:\n1. Identify untested code paths.\n2. Generate tests covering happy paths, edge cases, and error conditions.\n3. Use the project's existing test framework and conventions.\n4. Include both unit tests and integration tests where appropriate.",
      "allowedTools": ["read_file", "write_file", "run_tests", "grep_search"],
      "model": "claude-sonnet-4-20250514"
    },
    {
      "name": "research_coordinator",
      "description": "Manages multi-source research by spawning sub-researchers. Use for complex queries requiring multiple perspectives.",
      "systemPrompt": "You are a research coordinator. Decompose research queries into focused subtasks, delegate to sub-researchers, and synthesize findings into a coherent report.",
      "allowedTools": ["web_search", "read_document", "Task"],
      "model": "claude-sonnet-4-20250514"
    }
  ]
}
```

Note that `research_coordinator` has `"Task"` in its `allowedTools` — this is what allows it to spawn its own subagents for nested delegation.

---

### Example 2 — Passing Complete Findings in Subagent Prompt

```python
# The coordinator has already run a code_reviewer subagent and received findings.
# Now it needs to spawn a test_generator subagent that depends on those findings.

code_review_findings = {
    "file": "src/payments/refund.py",
    "issues": [
        {"severity": "critical", "line": 45, "description": "No validation on refund amount — negative values accepted"},
        {"severity": "warning", "line": 72, "description": "Race condition if two refunds processed simultaneously"},
        {"severity": "info", "line": 88, "description": "Magic number 30 should be a named constant (REFUND_WINDOW_DAYS)"},
    ],
}

# The subagent prompt must contain ALL context it needs — it cannot
# see the coordinator's earlier conversation with code_reviewer.

test_generator_prompt = f"""Generate tests for the refund processing module.

## Source File
File: src/payments/refund.py

## Code Review Findings (from prior analysis)
The following issues were identified and need test coverage:

{json.dumps(code_review_findings, indent=2)}

## Instructions
1. Write tests that reproduce each critical and warning issue.
2. Add regression tests that will fail if the fixes are reverted.
3. Use pytest conventions matching the existing test suite.
4. Read the source file first to understand the current implementation.

## Existing Test File
Check tests/test_refund.py for existing tests to avoid duplication."""

# Spawn the subagent with the fully self-contained prompt
result = run_subagent(
    agent="test_generator",
    prompt=test_generator_prompt,
)
```

The key pattern: the coordinator serializes everything the subagent needs (file paths, prior findings, instructions) into the prompt text. The subagent never sees the coordinator's conversation history.

---

### Example 3 — Structured Data Format Separating Content from Metadata

```python
# When passing context to subagents, separate the actual content from
# metadata about that content. This helps the subagent understand
# what it is looking at and how to use it.

subagent_context = """## Task
Analyze the customer support transcript below and recommend resolution actions.

## Metadata
- Ticket ID: SUP-2847
- Customer tier: Premium
- Account age: 3 years
- Previous escalations: 2 (both resolved satisfactorily)
- Current sentiment: Frustrated (detected from prior analysis)

## Transcript Content
[2024-03-15 09:23] Customer: I've been trying to export my data for three days
and keep getting a timeout error. This is unacceptable for a premium account.

[2024-03-15 09:24] Agent (bot): I understand your frustration. Let me look
into the export issue for you.

[2024-03-15 09:25] Customer: Your bot already said that yesterday. I want to
talk to a real person who can actually fix this.

## Available Actions
You can recommend any of the following:
- escalate_to_human(priority: str, summary: str)
- trigger_export_retry(ticket_id: str, timeout_override: int)
- apply_service_credit(amount: float, reason: str)
- schedule_callback(time_preference: str)

## Constraints
- Service credits over $50 require human approval.
- Premium customers should receive callback within 2 hours."""

# The metadata section gives the subagent structured context about the
# situation. The transcript is the raw content to analyze. The available
# actions and constraints tell it what it can recommend.
```

This three-part structure (metadata / content / action space) makes it easy for the subagent to understand the full picture without parsing unstructured text.

---

### Example 4 — Parallel Subagent Spawning

```python
# When the coordinator determines that multiple subtasks are independent,
# it spawns all subagents in a single response. The framework executes
# them concurrently.

# Coordinator's tool calls in a single response:
tool_calls = [
    {
        "type": "tool_use",
        "name": "Task",
        "input": {
            "agent": "environmental_analyst",
            "prompt": """Research water consumption impacts of lithium mining
in the Atacama Desert. Focus on: aquifer depletion rates, brine extraction
volumes, impact on local agriculture. Use web_search for recent studies.""",
        },
    },
    {
        "type": "tool_use",
        "name": "Task",
        "input": {
            "agent": "economic_analyst",
            "prompt": """Research economic impact of lithium mining on Chile's
economy. Focus on: export revenue contribution, employment in Antofagasta
region, comparison to copper mining revenue. Use web_search and data_analysis.""",
        },
    },
    {
        "type": "tool_use",
        "name": "Task",
        "input": {
            "agent": "policy_analyst",
            "prompt": """Research Chile's evolving lithium mining regulations.
Focus on: national lithium strategy, state ownership requirements, indigenous
consultation mandates, comparison to Argentina's approach. Use web_search.""",
        },
    },
]

# All three subagents run in parallel because:
# 1. They have no data dependencies on each other.
# 2. They are included in a single assistant response.
# 3. The coordinator will aggregate their results in the next turn.

# When all three return, the coordinator receives three tool_results
# and can synthesize them into a unified analysis.
```

Parallel spawning is efficient: instead of sequential subagent calls (3 round trips), the coordinator sends all three in one turn and processes all results in the next turn (1 round trip for spawning + 1 for aggregation).

---

## Key Takeaways for the Exam

1. **Task tool** is how subagents are spawned. Nested delegation requires `"Task"` in `allowedTools`.
2. **Context is never inherited** — the coordinator must pass all relevant information in the subagent prompt.
3. **AgentDefinitions** control the subagent's persona, capabilities, and tool access.
4. **fork_session** enables divergent exploration without polluting the original session.
5. **Parallel spawning** (multiple Task calls in one response) is the correct pattern for independent subtasks.

---

## Related Topics

- [[Domain 1 - Agentic Architecture & Orchestration]] — Parent domain overview
- [[Multi-Agent Orchestration]] — How the coordinator manages subagents
- [[Session Management]] — Session resumption and forking in detail
- [[Task Decomposition Strategies]] — Deciding what each subagent should work on
- [[Context Window Management]] — Token budgets when passing context to subagents
