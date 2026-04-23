---
tags:
  - domain-5
  - task-5-4
  - codebase-exploration
  - context-degradation
  - scratchpad
  - subagent-delegation
  - crash-recovery
domain: "5 - Context Management & Reliability"
task: "5.4"
---

# Large Codebase Exploration

> **Task 5.4** — Manage context degradation in extended coding sessions through scratchpad persistence, subagent delegation, structured state export, and `/compact` usage.

---

## Context Degradation in Extended Sessions

When Claude explores a large codebase over many turns, the context window gradually fills with file contents, grep results, directory listings, and intermediate reasoning. As this happens, observable degradation patterns emerge:

- **Inconsistent answers** — The model contradicts findings from earlier in the session because those findings have scrolled out of effective attention.
- **Referencing "typical patterns"** — Instead of citing specific code it read earlier, the model falls back to generic knowledge: *"Based on typical patterns in Express.js applications, this likely uses..."* This is a strong signal that concrete findings have been lost from context.
- **Redundant exploration** — The model re-reads files it already examined because it no longer remembers what it found.
- **Hallucinated file paths** — The model invents plausible-sounding paths (`src/utils/helpers.js`) that do not exist, because its mental model of the codebase has degraded.

These degradation symptoms typically appear after 15-25 tool calls in a single session, depending on how verbose the tool outputs are.

---

## Scratchpad Files for Persisting Key Findings

A scratchpad file is a plain text or markdown file that the agent writes to disk during exploration, recording key findings, file paths, function signatures, and architectural decisions. Because it exists on disk rather than only in the context window, it survives context compaction and can be re-read at any point.

### When to write to the scratchpad

- After discovering a key architectural pattern
- After mapping module dependencies
- After identifying the files relevant to a specific change
- Before transitioning to a new phase of exploration

### What to record

- File paths (absolute) with brief descriptions of what each file does
- Function/class signatures relevant to the task
- Architectural decisions discovered (e.g., "authentication is handled by middleware in `src/middleware/auth.ts`, not in individual route handlers")
- Open questions that need further investigation

---

## Subagent Delegation for Isolating Verbose Exploration

Reading file after file in the main agent's context window is the fastest way to exhaust it. Instead, delegate specific exploration questions to subagents, which operate in their own isolated context windows. The subagent reads the files, analyzes them, and returns a structured summary — only the summary enters the main agent's context.

### Division of labor

- **Main agent (coordinator)**: Maintains the overall plan, tracks progress, synthesizes findings, writes final output.
- **Subagents**: Answer specific, scoped questions ("What does the authentication middleware do?", "What database models are defined in `src/models/`?", "How are API routes registered?").

---

## Structured State Persistence for Crash Recovery

In long-running exploration sessions, crashes (network interruptions, token limit hits, process termination) are realistic. A structured state manifest allows a new session to pick up where the previous one left off.

The manifest captures:
- **Completed phases** and their findings
- **Current phase** and progress within it
- **Remaining phases** and their objectives
- **Key file paths** discovered so far
- **Open questions** and blockers

---

## Using `/compact` for Extended Exploration

The `/compact` command in Claude Code triggers context compaction — summarizing the conversation history to free up context window space. This is essential during extended exploration sessions but comes with trade-offs:

- **Benefit**: Frees up context for new exploration without starting a fresh session.
- **Risk**: Specific details (file paths, function signatures, exact code snippets) may be lost in summarization.
- **Mitigation**: Write key findings to the scratchpad file **before** running `/compact`, then re-read the scratchpad after compaction to restore critical context.

---

## Examples

### Example 1 — Spawning Subagents for Specific Questions

```python
# Coordinator delegates specific codebase questions to subagents.
# Each subagent has its own context window and returns structured results.

COORDINATOR_PLAN = """
Phase 1: Understand the codebase architecture
  - Subagent A: Map the directory structure and identify key modules
  - Subagent B: Analyze the authentication and authorization system
  - Subagent C: Document the database schema and ORM models

Phase 2: Identify files to modify for the feature request
  - Subagent D: Find all API routes related to user profiles
  - Subagent E: Trace the data flow from API endpoint to database
"""

# Subagent A system prompt — scoped to a specific question
SUBAGENT_A_PROMPT = """You are analyzing a codebase to map its directory structure.

Your task: Read the top-level directory, identify key modules, and return a
structured summary.

Return your findings as JSON:
{
  "project_type": "string — framework and language",
  "key_directories": [
    {"path": "string", "purpose": "string", "key_files": ["string"]}
  ],
  "entry_point": "string — main application entry file",
  "config_files": ["string — paths to configuration files"],
  "test_directory": "string — where tests live"
}
"""

# Coordinator receives structured summaries from all three subagents:
subagent_results = {
    "directory_structure": {
        "project_type": "Express.js with TypeScript",
        "key_directories": [
            {"path": "src/routes/", "purpose": "API route definitions", "key_files": ["users.ts", "orders.ts", "auth.ts"]},
            {"path": "src/models/", "purpose": "Sequelize ORM models", "key_files": ["User.ts", "Order.ts", "Product.ts"]},
            {"path": "src/middleware/", "purpose": "Express middleware", "key_files": ["auth.ts", "validation.ts", "rateLimit.ts"]},
        ],
        "entry_point": "src/app.ts",
        "config_files": ["tsconfig.json", ".env.example", "sequelize.config.js"],
        "test_directory": "tests/",
    },
    "auth_system": {
        "strategy": "JWT with refresh tokens",
        "middleware_path": "src/middleware/auth.ts",
        "key_functions": ["verifyToken()", "requireRole(role)"],
        "token_storage": "httpOnly cookies",
    },
    "database_schema": {
        "orm": "Sequelize",
        "models": ["User", "Order", "Product", "OrderItem", "RefreshToken"],
        "key_relationships": "User hasMany Orders, Order belongsToMany Products through OrderItems",
    },
}

# Only these structured summaries enter the coordinator's context —
# not the hundreds of lines of source code the subagents read.
```

---

### Example 2 — Scratchpad File Recording Key Findings

```markdown
# File written to disk: /workspace/exploration-scratchpad.md
# Agent writes to this file throughout the session.

# Codebase Exploration Scratchpad
## Project: e-commerce-api

### Architecture Overview (Phase 1 — Complete)
- **Framework**: Express.js + TypeScript
- **ORM**: Sequelize with PostgreSQL
- **Auth**: JWT with refresh tokens, middleware at `src/middleware/auth.ts`
- **Entry point**: `src/app.ts`

### Key File Paths
| File | Purpose |
|------|---------|
| `src/routes/users.ts` | User profile CRUD endpoints |
| `src/routes/orders.ts` | Order management endpoints |
| `src/models/User.ts` | User model — includes tier, preferences |
| `src/models/Order.ts` | Order model — status enum, refund fields |
| `src/middleware/auth.ts` | JWT verification, role-based access |
| `src/middleware/validation.ts` | Zod schema validation middleware |

### Key Findings
1. User profile updates go through `PATCH /api/users/:id` in `src/routes/users.ts:47`
2. Authorization check: `requireRole('admin')` or `requireSelf()` (user can only edit own profile)
3. Validation uses Zod schemas defined inline in each route file (not centralized)
4. **Important**: The `User` model has a `preferences` JSONB column — schema is NOT validated by Sequelize, only by the route-level Zod schema

### Open Questions
- [ ] Where is email notification triggered on profile update? (not in routes or models)
- [ ] Is there a webhook system for profile changes?

### Phase 2 Progress
- [x] Identified API routes for user profiles
- [x] Traced data flow from endpoint to database
- [ ] Find email notification trigger
- [ ] Check for webhook/event system
```

The agent reads this scratchpad file at the beginning of each phase or after `/compact` to restore critical context. It is far cheaper to re-read a curated 50-line scratchpad than to re-explore dozens of source files.

---

### Example 3 — Summarizing Phase 1 Findings Before Phase 2 Subagents

```python
# After Phase 1 subagents complete, the coordinator summarizes findings
# and includes the summary in Phase 2 subagent prompts.
# This prevents Phase 2 subagents from re-exploring what Phase 1 already found.

PHASE_1_SUMMARY = """
## Phase 1 Findings (Context for Phase 2)

Architecture: Express.js + TypeScript, Sequelize ORM, PostgreSQL.
Auth: JWT middleware at src/middleware/auth.ts — verifyToken(), requireRole().
User profiles: PATCH /api/users/:id in src/routes/users.ts:47.
Validation: Zod schemas inline per route (not centralized).
User model: src/models/User.ts — has JSONB `preferences` column.

Key constraint: Zod validates preferences on input, but Sequelize does NOT
enforce schema on the JSONB column. Direct database writes could bypass validation.
"""

# Phase 2 subagent gets Phase 1 findings as context:
SUBAGENT_D_PROMPT = f"""You are analyzing a codebase to find where email
notifications are triggered on user profile updates.

{PHASE_1_SUMMARY}

Based on the above findings, investigate:
1. Check src/routes/users.ts for any event emission after profile update
2. Look for an event bus or pub/sub system in src/
3. Search for email-related modules (nodemailer, sendgrid, etc.)

Return structured findings with file paths and line numbers.
"""
```

By injecting the Phase 1 summary into Phase 2 subagent prompts, the coordinator prevents redundant exploration and gives subagents a head start.

---

### Example 4 — Crash Recovery with Structured Manifest

```json
// File: /workspace/.exploration-manifest.json
// Written periodically by the agent during long exploration sessions.
// If the session crashes, a new session loads this manifest to resume.

{
  "project": "e-commerce-api",
  "session_id": "explore-2026-04-22-001",
  "objective": "Implement user preference sync across devices",
  "last_updated": "2026-04-22T11:45:00Z",

  "phases": [
    {
      "id": "phase-1",
      "name": "Architecture mapping",
      "status": "complete",
      "findings_file": "/workspace/exploration-scratchpad.md",
      "key_findings": [
        "Express.js + TypeScript + Sequelize",
        "JWT auth middleware at src/middleware/auth.ts",
        "User preferences stored as JSONB in User model"
      ]
    },
    {
      "id": "phase-2",
      "name": "Profile update flow analysis",
      "status": "in_progress",
      "progress": "3/5 subtasks complete",
      "completed_subtasks": [
        "Identified API routes for user profiles",
        "Traced data flow from endpoint to database",
        "Found Zod validation schema for preferences"
      ],
      "remaining_subtasks": [
        "Find email notification trigger",
        "Check for webhook/event system"
      ]
    },
    {
      "id": "phase-3",
      "name": "Implementation plan",
      "status": "not_started",
      "objective": "Draft implementation plan for cross-device preference sync"
    }
  ],

  "key_files": [
    "src/routes/users.ts",
    "src/models/User.ts",
    "src/middleware/auth.ts",
    "src/middleware/validation.ts"
  ],

  "open_questions": [
    "Where is email notification triggered on profile update?",
    "Is there a webhook system for profile changes?"
  ]
}
```

```python
# Recovery logic in the coordinator:
def resume_exploration(manifest_path: str) -> str:
    """Load exploration manifest and generate a resumption prompt."""
    with open(manifest_path) as f:
        manifest = json.load(f)

    current_phase = next(
        (p for p in manifest["phases"] if p["status"] == "in_progress"), None
    )

    prompt = f"""Resuming codebase exploration for: {manifest['objective']}

## Completed Work
{json.dumps([p for p in manifest['phases'] if p['status'] == 'complete'], indent=2)}

## Current Phase: {current_phase['name']}
Progress: {current_phase['progress']}
Remaining: {json.dumps(current_phase['remaining_subtasks'])}

## Key Files (already identified)
{chr(10).join('- ' + f for f in manifest['key_files'])}

## Open Questions
{chr(10).join('- ' + q for q in manifest['open_questions'])}

Please read the scratchpad at {manifest['phases'][0]['findings_file']}
and continue from where we left off.
"""
    return prompt
```

The manifest enables a new session to skip all completed work and resume exactly where the previous session ended — no re-exploration required.

---

## Key Takeaways for the Exam

1. **Watch for degradation signals** — "Based on typical patterns" or inconsistent answers across turns mean the agent has lost its concrete findings from context.
2. **Scratchpad files are cheap insurance** — Write key findings to disk before they scroll out of context. Re-reading a 50-line scratchpad is vastly cheaper than re-exploring 20 source files.
3. **Delegate verbose exploration to subagents** — Only structured summaries should enter the coordinator's context, not raw file contents.
4. **Use `/compact` with protection** — Write findings to the scratchpad before compacting, then re-read after.
5. **Structured manifests enable crash recovery** — Periodic state export means long explorations can survive session interruptions.

---

## Related Topics

- [[Domain 5 - Context Management & Reliability]] — Parent domain overview
- [[Context Window Management]] — Token management strategies that apply to codebase exploration
- [[Multi-Agent Orchestration]] — Coordinator-subagent patterns for delegating exploration
- [[Error Propagation in Multi-Agent Systems]] — Handling failures during multi-phase exploration
- [[Session Management]] — Session persistence and resumption strategies
