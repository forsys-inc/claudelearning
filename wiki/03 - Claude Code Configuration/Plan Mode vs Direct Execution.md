---
tags:
  - claude-code
  - plan-mode
  - direct-execution
  - subagent
  - task-3-4
domain: "3 - Claude Code Configuration & Workflows"
task: "3.4"
---

# Plan Mode vs Direct Execution

> **Task 3.4** — Choose between plan mode and direct execution based on task complexity. Use the Explore subagent for isolating verbose discovery. Combine plan mode for investigation with direct execution for implementation.

---

## Overview

Claude Code supports two primary execution modes:

- **Plan mode** — Claude Code reasons through the problem, explores the codebase, and proposes a plan before making any changes
- **Direct execution** — Claude Code immediately begins implementing changes

Choosing the right mode for the task at hand is a core competency. Using plan mode for trivial changes wastes time; using direct execution for complex changes risks incorrect or incomplete implementations.

---

## Plan Mode

### When to Use Plan Mode

Plan mode is appropriate when:

| Indicator | Example |
|-----------|---------|
| **Complex task** | "Restructure the authentication module to support OAuth2 and SAML" |
| **Large-scale changes** | Modifications spanning 10+ files |
| **Multiple valid approaches** | "Should we use a queue or a cron job for this?" |
| **Architectural decisions** | Choosing between patterns that have long-term implications |
| **Multi-file modifications** | Changes where file ordering and dependencies matter |
| **Unfamiliar codebase** | First time working in a module, need to understand existing patterns |

### What Plan Mode Does

1. **Explores** the codebase — reads relevant files, searches for patterns
2. **Analyzes** the problem — identifies dependencies, constraints, edge cases
3. **Proposes** a plan — lists specific files to change, in what order, with what modifications
4. **Waits for approval** before making changes

This enables **safe exploration** — Claude Code can investigate multiple approaches, consider trade-offs, and surface potential issues before committing to any changes.

### Activating Plan Mode

Use the `shift+tab` key to toggle between plan and act modes during a session. You can also start a prompt with explicit instructions:

```
Plan how to refactor the payment processing module to support
multiple payment providers. Do not make any changes yet.
```

---

## Direct Execution

### When to Use Direct Execution

Direct execution is appropriate when:

| Indicator | Example |
|-----------|---------|
| **Simple, well-scoped change** | "Add null check before accessing user.email" |
| **Clear specification** | "Rename the `getData` function to `fetchUserProfile`" |
| **Single-file modification** | Adding a validation rule to one file |
| **Mechanical transformation** | "Add TypeScript types to all functions in this file" |
| **Known pattern** | "Add a new endpoint following the same pattern as GET /users" |

### What Direct Execution Does

1. **Immediately begins** reading relevant files and making changes
2. **Applies changes** as it goes — file edits, command execution
3. **Reports** what was done

Direct execution is faster and more efficient for tasks where the path forward is obvious.

---

## The Explore Subagent

The Explore subagent is a specialized mode for **isolating verbose discovery** from the main session.

### How It Works

- Launches a **separate sub-agent context** dedicated to exploration
- The sub-agent can read files, search code, and run commands freely
- Only the **summary findings** are returned to the main session
- The intermediate steps (dozens of file reads, grep results) do not clutter the main conversation

### When to Use Explore

- You need to understand a large, unfamiliar module before making changes
- Discovery will involve reading many files and the intermediate output is not useful to keep
- You want to search for patterns across the codebase without filling the context window

### Explore vs. Plan Mode

| Aspect | Plan Mode | Explore Subagent |
|--------|-----------|-----------------|
| **Context** | Same session — all exploration visible | Forked — only summary returns |
| **Token usage** | Exploration consumes main context | Exploration contained in sub-context |
| **Best for** | Planning changes you'll review step by step | Large-scale discovery where details are noise |
| **Output** | Detailed plan with file-by-file changes | Summary findings and recommendations |

---

## Combining Plan Mode and Direct Execution

The most effective workflow for complex tasks often combines both modes:

```
Phase 1: Plan (understand)
  → Explore the codebase
  → Identify affected files
  → Propose approach
  → Get user approval

Phase 2: Execute (implement)
  → Switch to direct execution
  → Apply the approved plan
  → Make changes file by file
```

This "plan then execute" pattern gives you the safety of exploration with the efficiency of direct implementation.

---

## Examples

### Example 1: Plan Mode for Microservice Restructuring

**Scenario:** The team wants to extract the notification system from the monolith into a standalone microservice.

**Prompt (plan mode):**

```
I need to extract the notification system into a separate microservice.
Currently, notifications are handled in:
- src/services/notification.ts
- src/models/notification.ts
- src/routes/notifications.ts
- src/workers/email-worker.ts
- src/workers/push-worker.ts

Plan this extraction. Consider:
- What interfaces need to be defined for the service boundary?
- What database tables need to be migrated?
- How should the monolith communicate with the new service?
- What is the migration strategy (big bang vs. strangler fig)?

Do not make any changes yet.
```

**Why plan mode?** This is a large-scale architectural change with multiple valid approaches (REST vs. gRPC vs. events for communication, big-bang vs. strangler-fig for migration). Making changes without a plan could lead to an inconsistent intermediate state.

**Claude Code's plan output might include:**

```markdown
## Extraction Plan

### Phase 1: Define Service Boundary
- Create `NotificationServiceClient` interface in monolith
- Define gRPC proto: `notification.proto` with SendNotification, GetStatus RPCs
- Files: new `proto/notification.proto`, new `src/clients/notification-client.ts`

### Phase 2: Build Microservice Scaffold
- New repo: `notification-service/`
- Copy models, workers, and business logic
- Set up independent database connection
- Files: 8 new files in new service

### Phase 3: Strangler Fig Migration
- Replace direct calls with client calls behind feature flag
- Run both paths in parallel for 2 weeks
- Monitor for discrepancies
- Files: modify 12 call sites in monolith

### Risks
- Email worker has a hard dependency on user preferences stored in monolith DB
- Push notification tokens are in the users table — need a sync mechanism
```

### Example 2: Direct Execution for Single-File Bug Fix

**Scenario:** Users report a crash when their profile has no email address.

**Prompt (direct execution):**

```
There's a null reference error in src/components/UserProfile.tsx.
When user.email is null, the toLowerCase() call on line 47 crashes.
Add a null check and display "No email provided" as fallback.
```

**Why direct execution?** This is a single-file, well-scoped fix with a clear specification. There is exactly one approach: add a null check. Planning would be wasteful.

**Claude Code immediately edits the file:**

```typescript
// Before
const displayEmail = user.email.toLowerCase();

// After
const displayEmail = user.email ? user.email.toLowerCase() : 'No email provided';
```

### Example 3: Explore Subagent for Verbose Discovery

**Scenario:** You joined a new team and need to understand how the caching layer works before making changes to it.

**Prompt:**

```
Use the explore subagent to investigate the caching layer in this project.
I need to understand:
- What caching libraries/services are used
- Where caches are configured
- What the cache invalidation strategy is
- Whether there are any known issues (check TODOs and comments)

Summarize your findings — I don't need to see every file you read.
```

**Why Explore subagent?** Understanding the caching layer might require reading 30+ files — cache configuration, middleware, decorators, utility functions, tests, and comments. Running this in the main session would flood the context with file contents that are only needed for intermediate understanding.

**The Explore subagent internally:**
1. Searches for `cache`, `redis`, `memcached` across the codebase
2. Reads 25 files containing cache-related code
3. Reads configuration files and environment variables
4. Checks test files for cache behavior specifications
5. Scans for TODO and FIXME comments related to caching

**Returns to main session:**

```markdown
## Caching Layer Summary

**Technology:** Redis via `ioredis`, with an in-memory LRU fallback (node-lru-cache)

**Configuration:** `src/config/cache.ts` — TTL defaults are 5min for API responses,
1hr for user sessions, 24hr for feature flags

**Invalidation Strategy:** Event-driven — the `EventBus` emits entity change events,
and `CacheInvalidator` (src/services/cache-invalidator.ts) subscribes to clear
relevant keys. Pattern: `entity:{type}:{id}`

**Known Issues:**
- TODO in cache-invalidator.ts: "Race condition when bulk updates fire multiple
  invalidation events simultaneously" (line 87)
- FIXME in user-cache.ts: "Stale cache when user updates profile via mobile app
  because mobile doesn't emit the same events" (line 34)
```

### Example 4: Combined Approach — Plan Migration Then Execute

**Scenario:** Migrate the database ORM from Sequelize to Prisma.

**Phase 1 — Plan mode:**

```
Plan the migration from Sequelize to Prisma for this project.
Investigate the current Sequelize usage and propose a migration plan.
Do not make changes yet.
```

Claude Code explores and produces:

```markdown
## Migration Plan: Sequelize → Prisma

### Current State
- 14 Sequelize models in src/models/
- 23 files with direct Sequelize queries
- 8 migration files in db/migrations/
- Custom Sequelize hooks in 3 models

### Migration Order (by dependency)
1. Generate Prisma schema from existing DB (prisma db pull)
2. Migrate models one by one, starting with leaf models (no relations):
   - AuditLog, Setting, FeatureFlag (no FK dependencies)
3. Then dependent models: User, Organization, Team
4. Then junction tables: TeamMember, UserRole
5. Finally: complex models with hooks (Order, Payment, Subscription)

### Risk Areas
- Custom hooks (beforeCreate, afterUpdate) need manual conversion to Prisma middleware
- Raw SQL queries in src/reports/ — need to convert to Prisma.$queryRaw
- Sequelize Op.like / Op.in syntax differences
```

**Phase 2 — Direct execution (after approval):**

```
Good plan. Let's start with step 1 — run prisma db pull to generate the schema.
Then migrate the three leaf models: AuditLog, Setting, FeatureFlag.
```

Claude Code switches to direct execution and begins making changes immediately, following the approved plan.

---

## Decision Framework

```
Is the task well-defined and scoped to 1-3 files?
├── YES → Direct execution
└── NO
    ├── Do I need to explore unfamiliar code first?
    │   ├── YES, and exploration will be verbose → Explore subagent
    │   └── YES, and I want to review the exploration → Plan mode
    └── Are there multiple valid approaches?
        ├── YES → Plan mode (compare approaches first)
        └── NO, but it's a large change → Plan mode (get approval on scope)
```

---

## Exam Tips

- **Plan mode** = complex, multi-file, architectural, multiple approaches
- **Direct execution** = simple, well-scoped, single-file, clear specification
- **Explore subagent** = verbose discovery where intermediate results are noise
- The **combined approach** (plan then execute) is the recommended workflow for large tasks
- Plan mode enables **safe exploration** — you can investigate without committing to changes
- `shift+tab` toggles between plan and act modes

---

## Related Topics

- [[Iterative Refinement Techniques]] — plan mode often leads into iterative refinement
- [[Custom Slash Commands and Skills]] — skills with `context: fork` relate to the Explore subagent concept
- [[CLAUDE.md Configuration Hierarchy]] — CLAUDE.md can include guidance on when to use plan mode for specific project areas
- [[CI/CD Integration]] — CI pipelines typically use direct execution with `-p` flag
