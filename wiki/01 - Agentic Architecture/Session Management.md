---
tags:
  - domain-1
  - task-1-7
  - session-management
  - resume
  - fork-session
  - parallel-exploration
domain: "1 - Agentic Architecture & Orchestration"
task: "1.7"
---

# Session Management

> **Task 1.7** — Manage sessions effectively using named resumption, forked sessions for parallel exploration, and strategies for choosing between resumption and fresh starts.

---

## Named Session Resumption with `--resume`

Claude Code supports named sessions, allowing you to pick up a prior conversation by name rather than relying on anonymous session IDs. This is critical for long-running investigations that span multiple work periods.

```bash
# Start a named session
claude --session-name "auth-refactor-investigation"

# Later, resume by name
claude --resume "auth-refactor-investigation"
```

### When Resumption Works Well

- The prior conversation established a **mental model** of the codebase (identified key files, understood architecture) that would be expensive to rebuild.
- The investigation is **multi-day** and you need continuity — e.g., a security audit where you identified 5 areas of concern on Day 1 and want to deep-dive into each on subsequent days.
- The prior session's **tool results are still mostly valid** — files have not changed significantly since the session was last active.

### The Stale Context Problem

When you resume a session, Claude retains the full conversation history including tool results (file contents, search results, command outputs). If the underlying files have changed since the session was last active, those cached tool results are **stale** — Claude will reason about code that no longer exists in its original form.

**This is the single most important consideration for session resumption.** Stale tool results do not produce errors — they produce silently wrong analysis.

---

## `fork_session` for Parallel Exploration

`fork_session` creates an independent branch from a shared analysis baseline. Both the original and forked session retain the conversation history up to the fork point, but subsequent messages in each are independent.

This is the canonical approach when you want to **compare divergent strategies** from a common starting point without cross-contamination of reasoning.

```
Session A: [shared analysis] → [approach 1: extract-method refactor]
                 ↘
Session B: [shared analysis] → [approach 2: strategy-pattern refactor]
```

### Why Fork Instead of Two Fresh Sessions?

Without forking, each session would need to independently re-analyze the codebase to reach the same baseline understanding. Forking preserves the shared analysis investment and ensures both branches reason from identical context.

---

## Resume vs. Fresh Start Decision Framework

| Factor | Resume | Fresh Start |
|--------|--------|-------------|
| **Files changed since last session** | Few or none | Many critical files changed |
| **Prior analysis depth** | Deep — took many turns to build understanding | Shallow — easy to rebuild |
| **Tool results** | Still valid (code unchanged) | Stale (code refactored, tests modified) |
| **Time since last session** | Hours to 1-2 days | Weeks |
| **Session goal** | Continue same investigation | New investigation on same codebase |

**Key exam principle:** Starting a new session with a structured summary of prior findings is often **more reliable** than resuming a session with stale tool results. The summary provides the *conclusions* without the outdated *evidence*, letting the new session re-verify against current code.

---

## Informing Resumed Sessions About Changes

When you must resume (because the prior context is too valuable to discard), explicitly tell the agent what has changed. This triggers targeted re-analysis rather than full re-exploration.

```
"Since our last session, the following files have been modified:
- src/auth/middleware.ts — Added rate limiting logic
- src/auth/types.ts — New RateLimitConfig interface
- tests/auth/middleware.test.ts — New test cases for rate limiting

Our prior analysis of the auth flow is still valid for the token validation
path, but the middleware layer needs re-examination."
```

This approach is far more efficient than either (a) resuming blindly with stale context or (b) starting fresh and losing all prior analysis.

---

## Examples

### Example 1 — Using `--resume` to Continue a Named Investigation Across Work Days

A security audit spans three days. On Day 1, the engineer starts a named session and Claude analyzes the authentication subsystem, identifying 5 areas of concern:

```bash
# Day 1: Start the investigation
claude --session-name "security-audit-q2"

# Prompt: "Analyze the authentication subsystem in src/auth/ for security
# vulnerabilities. Focus on token handling, session management, and input
# validation."

# Claude explores 12 files, identifies 5 areas of concern, and builds
# a detailed understanding of the auth flow.
```

```bash
# Day 2: Resume to deep-dive into the first two findings
claude --resume "security-audit-q2"

# Prompt: "Let's deep-dive into findings #1 (JWT token not validated on
# the /api/admin routes) and #2 (session fixation in the login flow).
# For each, show me the exact vulnerable code path and suggest a fix."

# Claude already knows the file structure and auth flow — it can
# immediately focus on the specific vulnerable paths without re-exploring.
```

```bash
# Day 3: Resume for remaining findings
claude --resume "security-audit-q2"

# Prompt: "Continue with findings #3 through #5. Also, the team pushed
# a fix for finding #1 yesterday — src/auth/adminRoutes.ts now validates
# the JWT. You can skip re-analyzing that file for the original issue."
```

The Day 3 prompt explicitly informs Claude that `adminRoutes.ts` has changed and the prior finding is resolved. Without this, Claude might re-flag the already-fixed issue based on stale cached file contents.

---

### Example 2 — `fork_session` to Compare Two Refactoring Approaches

An engineering team needs to refactor a monolithic `OrderProcessor` class. They want to compare two approaches from the same baseline analysis:

```bash
# Initial session: Establish shared understanding
claude --session-name "order-processor-refactor"

# Prompt: "Analyze src/orders/OrderProcessor.ts. Map out all public methods,
# their dependencies, and which internal state they share. Identify the
# natural seams for decomposition."

# Claude analyzes the class, identifies 4 method clusters with shared state,
# and maps the dependency graph. This took 8 turns of exploration.
```

Now fork to explore two strategies independently:

```bash
# Fork A: Extract-method approach
# (uses fork_session from within the session)

# Prompt in Fork A: "Refactor OrderProcessor using the extract-class pattern.
# Create separate classes for OrderValidation, InventoryReservation,
# PaymentProcessing, and ShipmentScheduling. Show me the resulting class
# interfaces and how they compose."
```

```bash
# Fork B: Strategy-pattern approach

# Prompt in Fork B: "Refactor OrderProcessor using the strategy pattern.
# Define an OrderStep interface and implement each processing phase as a
# strategy. Show me how the pipeline would be configured for different
# order types (standard, expedited, international)."
```

Both forks share the 8-turn baseline analysis (dependency graph, method clusters, shared state mapping). Each explores a different decomposition without the other's reasoning polluting the analysis. The team can compare the outputs side-by-side.

---

### Example 3 — Decision Matrix: Resume vs. Fresh Start

**Scenario A — Resume is the right choice:**

You spent 15 turns on Day 1 analyzing a complex data pipeline, mapping how data flows through 8 microservices. Overnight, only one service (`enrichment-svc`) was updated with a minor logging change. The core data flow is unchanged.

- Prior analysis: deep, expensive to rebuild
- File changes: minimal, non-structural
- Decision: **Resume** and note the logging change in `enrichment-svc`

**Scenario B — Fresh start is the right choice:**

You analyzed a REST API on Monday. By Thursday, the team has migrated 6 of 10 endpoints from REST to GraphQL, renamed multiple files, and restructured the routing layer.

- Prior analysis: deeply tied to file structure that no longer exists
- File changes: extensive, structural
- Decision: **Fresh start** with a summary:

```
"Last Monday I analyzed the API layer and found these issues:
1. No rate limiting on /api/users (now migrated to GraphQL query)
2. Missing input validation on /api/orders POST (still REST)
3. N+1 query in /api/products listing (now GraphQL, may be resolved)

Please re-analyze the current API layer. Issue #2 likely still applies.
Issues #1 and #3 may be resolved by the GraphQL migration — verify."
```

**Scenario C — Fork is the right choice:**

You've spent 10 turns analyzing a performance bottleneck and identified the hot path. Now you want to compare two optimization strategies (caching vs. denormalization) without either exploration influencing the other. Fork from the current point.

---

### Example 4 — Informing a Resumed Session About Specific File Changes

After resuming a session that previously analyzed a test suite:

```
"Since our last session, these changes were made to the test suite:

MODIFIED FILES:
- tests/integration/payment.test.ts
  → Added 3 new test cases for subscription billing
  → The existing tests we analyzed are unchanged

- src/payments/subscription.ts
  → New file implementing the SubscriptionBillingService class
  → This is the production code for the new test cases

DELETED FILES:
- tests/integration/legacy_payment.test.ts
  → Removed (we flagged this as redundant in our prior analysis)

UNCHANGED:
- All other files in src/payments/ and tests/integration/
- The PaymentGateway interface we analyzed is unchanged

Please re-analyze only the modified and new files in the context of our
prior understanding. Specifically:
1. Do the 3 new subscription tests provide adequate coverage?
2. Does SubscriptionBillingService follow the same patterns as the
   existing PaymentProcessor we analyzed?"
```

This targeted approach lets Claude leverage its existing understanding of `PaymentProcessor` and the test patterns while focusing re-analysis only on what actually changed. Without this guidance, Claude would either (a) re-read all files wastefully or (b) reason about deleted files that no longer exist.

---

## Key Takeaways for the Exam

1. **`--resume <session-name>`** continues a specific named session — use it when prior analysis is deep and files are mostly unchanged.
2. **`fork_session`** creates independent branches from a shared baseline — use it to compare divergent approaches without cross-contamination.
3. **Stale tool results** are the primary risk of resumption — Claude will reason about outdated file contents without warning.
4. **A structured summary in a fresh session** is more reliable than resuming with extensively stale context — it preserves conclusions while allowing re-verification.
5. **Explicitly inform resumed sessions about changes** — this triggers targeted re-analysis instead of blind reliance on cached results or wasteful full re-exploration.

---

## Related Topics

- [[Domain 1 - Agentic Architecture & Orchestration]] — Parent domain overview
- [[Agentic Loop Lifecycle]] — Understanding how conversation history accumulates across session turns
- [[Multi-Agent Orchestration]] — Forked sessions as lightweight alternative to full multi-agent setups
- [[Context Window Management]] — Session resumption implications for context window usage
- [[Large Codebase Exploration]] — Exploration strategies that benefit from session continuity
- [[Subagent Configuration and Context Passing]] — Context passing patterns relevant to forked sessions
