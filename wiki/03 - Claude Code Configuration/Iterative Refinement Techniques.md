---
tags:
  - claude-code
  - iteration
  - prompt-patterns
  - test-driven
  - task-3-5
domain: "3 - Claude Code Configuration & Workflows"
task: "3.5"
---

# Iterative Refinement Techniques

> **Task 3.5** — Apply concrete input/output examples, test-driven iteration, and the interview pattern to converge on correct output. Know when to batch interacting issues vs. fix sequentially.

---

## Overview

Getting the right output from Claude Code often requires iteration. The difference between effective and ineffective iteration is the **communication method** you use. This task covers four key techniques:

1. **Concrete input/output examples** — the single most effective way to communicate intent
2. **Test-driven iteration** — write tests first, share failures, let Claude Code fix progressively
3. **Interview pattern** — let Claude Code ask questions to surface considerations you missed
4. **Batch vs. sequential** — knowing when to report multiple issues at once vs. one at a time

---

## Technique 1: Concrete Input/Output Examples

### Why Examples Are the Most Effective Communication Method

Natural language descriptions are inherently ambiguous. Consider:

> "Transform the user data into a summary format"

This could mean dozens of things. But providing 2-3 concrete examples makes the intent unambiguous:

```
Input:  { name: "Alice", age: 30, role: "admin", lastLogin: "2026-04-20" }
Output: "Alice (admin) — last active 2 days ago"

Input:  { name: "Bob", age: 25, role: "viewer", lastLogin: null }
Output: "Bob (viewer) — never logged in"

Input:  { name: "Charlie", age: 45, role: "editor", lastLogin: "2026-04-22" }
Output: "Charlie (editor) — last active today"
```

Now the transformation is crystal clear: name, role in parens, relative time for last login, "never logged in" for null.

### When to Use Examples

- **Data transformations** — input format to output format
- **Formatting rules** — how strings, dates, numbers should display
- **Edge cases** — null handling, empty arrays, boundary values
- **Complex business logic** — examples are faster than explaining rules

### How Many Examples

- **2-3 examples** covers most cases — one "normal" case plus edge cases
- Ensure examples cover the **boundaries**: null/empty, single item, multiple items, error cases
- Each example should demonstrate a **different behavior** — avoid redundant examples

---

## Technique 2: Test-Driven Iteration

### The Cycle

Test-driven iteration uses failing tests as the communication mechanism:

```
Step 1: You write tests that define the expected behavior
Step 2: Share the test file (or failures) with Claude Code
Step 3: Claude Code implements until tests pass
Step 4: If new edge cases emerge, add tests → repeat
```

This is powerful because:
- Tests are **unambiguous specifications** — they either pass or fail
- Claude Code can run the tests itself and iterate autonomously
- The feedback loop is tight: implement → test → see failure → fix → test
- You end up with both working code and a test suite

### Progressive Improvement

Start with basic tests and add complexity:

```
Round 1: Basic happy-path tests → Claude Code implements core logic
Round 2: Edge case tests (nulls, empty, boundaries) → Claude Code handles edges
Round 3: Error handling tests → Claude Code adds proper error handling
Round 4: Performance tests (optional) → Claude Code optimizes
```

Each round builds on the previous implementation rather than starting over.

---

## Technique 3: Interview Pattern

### How It Works

Instead of trying to specify everything upfront, you ask Claude Code to **interview you** about the requirements:

```
I need to implement a caching layer for our API responses.
Before you start, ask me questions about the requirements
that would affect your implementation decisions.
```

Claude Code then asks targeted questions:

```
1. What cache backend? (Redis, in-memory, filesystem)
2. What's the expected cache hit ratio and data volume?
3. Should cache invalidation be time-based, event-based, or both?
4. Do you need cache warming on startup?
5. Should cached responses include headers, or just bodies?
6. Is there a circuit-breaker requirement if the cache backend is down?
```

### Why This Works

- **Surfaces considerations you didn't think of** — Claude Code's questions reveal design decisions you might have missed
- **Prevents rework** — answering questions upfront is faster than implementing the wrong thing and refactoring
- **Documents decisions** — the Q&A exchange becomes a record of design choices

### When to Use the Interview Pattern

- Requirements are vague or incomplete
- The domain is complex with many interacting decisions
- You want to ensure nothing is missed before implementation
- You are working on infrastructure or architecture (caching, auth, data pipelines)

---

## Technique 4: Batch vs. Sequential Issue Resolution

### When to Batch Issues (Interacting Issues)

Batch multiple issues into a single message when they **interact** or **overlap**:

```
I found three issues in the payment processing flow:
1. The retry logic doesn't respect the idempotency key
2. The error handler catches too broadly, swallowing validation errors
3. The amount calculation uses floating-point, causing rounding errors

These are all in the same flow (processPayment → chargeCard → handleResult).
Please fix all three together since they affect each other.
```

**Why batch?** Fixing the retry logic might change how errors propagate, which affects the error handler fix. Fixing the amount calculation might change the values the retry logic works with. Fixing them independently could cause conflicts or require rework.

### When to Fix Sequentially (Independent Issues)

Fix issues one at a time when they are **independent** — in different files, different features, no shared state:

```
Message 1: "Fix the typo in the login page title — it says 'Sing In' instead of 'Sign In'"
Message 2: "Add the missing index on the orders.created_at column"
Message 3: "Update the README to reflect the new Node.js 20 requirement"
```

**Why sequential?** These are in completely different parts of the codebase with no interactions. Batching them would make the conversation harder to follow and the changes harder to review.

### Decision Guide

```
Do the issues share code, state, or data flow?
├── YES → Batch them (fix together to avoid conflicts)
└── NO
    ├── Are they in the same file?
    │   ├── YES → Consider batching (easier to review as one diff)
    │   └── NO → Fix sequentially (cleaner commits, easier review)
    └── Could fixing one change the context for another?
        ├── YES → Batch them
        └── NO → Fix sequentially
```

---

## Examples

### Example 1: Input/Output Examples Clarifying a Transformation

**Scenario:** You need Claude Code to build a function that converts structured log entries into human-readable summaries.

**Prompt:**

```
Build a function `formatLogEntry(entry: LogEntry): string` that converts
structured log entries into human-readable one-line summaries.

Here are examples of the exact output I expect:

Input:  { level: "error", service: "auth", message: "Token expired", timestamp: 1745337600, userId: "u_123", metadata: { tokenAge: 3600 } }
Output: "[2026-04-22 12:00:00] ERROR auth: Token expired (user: u_123, tokenAge: 3600)"

Input:  { level: "info", service: "api", message: "Request completed", timestamp: 1745337601, userId: null, metadata: { method: "GET", path: "/users", duration: 45 } }
Output: "[2026-04-22 12:00:01] INFO  api: Request completed (method: GET, path: /users, duration: 45)"

Input:  { level: "warn", service: "db", message: "Slow query", timestamp: 1745337602, userId: "u_456", metadata: {} }
Output: "[2026-04-22 12:00:02] WARN  db: Slow query (user: u_456)"
```

**What the examples communicate without words:**
- Timestamp format: `[YYYY-MM-DD HH:MM:SS]`
- Level is uppercase and right-padded to 5 characters (`INFO `, `WARN `, `ERROR`)
- userId appears as `user: u_xxx` in the parens, but is omitted when null
- Empty metadata object means no metadata items in the parens
- Metadata keys are listed as `key: value` pairs, comma-separated

Three examples communicated all of this more precisely than a paragraph of description could.

### Example 2: Test-Driven Iteration Cycle

**Scenario:** Building a rate limiter middleware.

**Round 1 — Basic behavior:**

```typescript
// rate-limiter.test.ts — Round 1

describe('RateLimiter', () => {
  it('allows requests under the limit', () => {
    const limiter = createRateLimiter({ maxRequests: 5, windowMs: 60000 });
    for (let i = 0; i < 5; i++) {
      expect(limiter.check('user_1')).toEqual({ allowed: true, remaining: 4 - i });
    }
  });

  it('blocks requests over the limit', () => {
    const limiter = createRateLimiter({ maxRequests: 2, windowMs: 60000 });
    limiter.check('user_1'); // 1st — allowed
    limiter.check('user_1'); // 2nd — allowed
    const result = limiter.check('user_1'); // 3rd — blocked
    expect(result).toEqual({ allowed: false, remaining: 0, retryAfterMs: expect.any(Number) });
  });
});
```

**Prompt:** "Here are my tests for the rate limiter. Implement `createRateLimiter` to make them pass."

**Round 2 — Edge cases (after basic tests pass):**

```typescript
  it('tracks separate limits per key', () => {
    const limiter = createRateLimiter({ maxRequests: 1, windowMs: 60000 });
    expect(limiter.check('user_1').allowed).toBe(true);
    expect(limiter.check('user_2').allowed).toBe(true); // different user, separate limit
    expect(limiter.check('user_1').allowed).toBe(false); // user_1 is now blocked
  });

  it('resets after the window expires', () => {
    vi.useFakeTimers();
    const limiter = createRateLimiter({ maxRequests: 1, windowMs: 1000 });
    limiter.check('user_1'); // use up the limit
    expect(limiter.check('user_1').allowed).toBe(false);

    vi.advanceTimersByTime(1001);
    expect(limiter.check('user_1').allowed).toBe(true); // window reset
    vi.useRealTimers();
  });
```

**Prompt:** "Added more tests. The first two pass but these new ones fail. Fix the implementation."

Each round adds specificity, and Claude Code progressively improves the implementation.

### Example 3: Interview Pattern Surfacing Cache Invalidation Strategies

**Prompt:**

```
I need to add a caching layer to our product catalog API.
Before implementing anything, interview me about the requirements
so you can make the right design decisions.
```

**Claude Code's questions:**

```markdown
Before I implement the caching layer, I have several questions:

1. **Cache backend**: Should I use Redis (shared across instances) or
   in-memory (per-instance)? Do you already have Redis in your infrastructure?

2. **Invalidation strategy**:
   - Time-based (TTL): How stale can product data be? (seconds/minutes/hours?)
   - Event-based: Does the system emit events when products are updated?
   - Manual: Should there be an admin endpoint to flush the cache?

3. **Cache granularity**:
   - Per-product caching (`product:{id}`) — good for individual lookups?
   - Per-query caching (`products:category=shoes&sort=price`) — good for listing pages?
   - Both?

4. **Cache warming**: Should the cache be pre-populated on startup,
   or is cold-start latency acceptable?

5. **Failure mode**: If Redis is down, should the API:
   - Serve requests without caching (degraded performance)?
   - Return an error?
   - Fall back to in-memory cache?

6. **Serialization**: Products include images as URLs — are there any
   fields that should NOT be cached (e.g., real-time inventory counts)?
```

**Your answers:**

```
1. Redis — we already have it for sessions
2. TTL of 5 minutes + event-based invalidation via our existing EventBus
3. Both — per-product for detail pages, per-query for listing pages
4. No warming needed, cold start is fine
5. Fallback to in-memory with shorter TTL (30 seconds)
6. Exclude `inventory_count` from cache — that must always be real-time
```

Now Claude Code has all the information needed to implement the caching layer correctly on the first attempt, including the Redis fallback, the exclusion of inventory counts, and the dual-granularity strategy.

### Example 4: Batch vs. Sequential Issue Resolution

**Batch (interacting issues):**

```
I found these issues in the checkout flow that are all interconnected:

1. The `calculateTotal()` function doesn't apply the discount before tax,
   so discounted orders are taxed on the full price
2. The `applyDiscount()` function mutates the cart object instead of
   returning a new one, causing stale state in the React component
3. The `CartSummary` component reads `cart.total` directly instead of
   using the computed total from `calculateTotal()`

These three issues compound — the discount calculation feeds into the total
calculation which feeds into the display. Please fix all three together
so the data flows correctly end to end.
```

**Why batched?** Fixing `calculateTotal` alone would still show wrong numbers because `CartSummary` bypasses it. Fixing `applyDiscount` alone would break the current (buggy) flow because other code relies on the mutation. All three must be fixed together for the checkout flow to work correctly.

**Sequential (independent issues):**

```
Message 1:
"The /api/health endpoint returns 200 even when the database is unreachable.
Fix it to check the DB connection and return 503 if it fails."

--- (after reviewing the fix) ---

Message 2:
"The user avatar upload doesn't validate file type. Add validation to
only accept PNG, JPG, and WebP files under 5MB."

--- (after reviewing the fix) ---

Message 3:
"Add a dark mode toggle to the settings page. The CSS variables are
already set up in globals.css — just need the toggle component and
the localStorage persistence."
```

**Why sequential?** These are in completely different domains (health checks, file uploads, UI theming). Fixing them one at a time produces cleaner diffs, clearer commit messages, and easier code review.

---

## Exam Tips

- **Concrete examples** are the most effective communication method — always prefer 2-3 examples over a paragraph of description
- **Test-driven iteration** uses failing tests as unambiguous specifications; add tests in rounds for progressive improvement
- **Interview pattern** is best when requirements are vague — Claude Code surfaces considerations you missed
- **Batch** interacting issues (same flow, shared state); **sequential** for independent issues (different files, no overlap)
- The key question for batch vs. sequential: "Could fixing one issue affect the fix for another?"

---

## Related Topics

- [[Plan Mode vs Direct Execution]] — plan mode often precedes iterative refinement
- [[Custom Slash Commands and Skills]] — you can encode iterative patterns as reusable slash commands
- [[CLAUDE.md Configuration Hierarchy]] — testing and iteration standards can be documented in CLAUDE.md
- [[Domain 4 - Prompt Engineering]] — the examples and interview techniques overlap with prompt engineering principles
