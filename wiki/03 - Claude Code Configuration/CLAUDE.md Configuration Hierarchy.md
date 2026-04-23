---
tags:
  - claude-code
  - configuration
  - CLAUDE-md
  - memory-files
  - task-3-1
domain: "3 - Claude Code Configuration & Workflows"
task: "3.1"
---

# CLAUDE.md Configuration Hierarchy

> **Task 3.1** — Configure and troubleshoot the CLAUDE.md hierarchy across user, project, and directory levels. Use `@import` for modularity and `.claude/rules/` for topic-specific instructions.

---

## Overview

CLAUDE.md files are the primary mechanism for giving Claude Code persistent instructions — coding standards, architectural conventions, workflow preferences, and project-specific context. They function as "memory files" that are automatically loaded at the start of every session.

The critical concept is the **three-level hierarchy**: instructions can be defined at the user level, project level, or directory level. Each level has different visibility, sharing characteristics, and override behavior.

---

## The Three-Level Hierarchy

```
Priority (highest → lowest):
┌─────────────────────────────────────────────┐
│  3. Directory-level CLAUDE.md               │  ← subdirectory-specific overrides
│     e.g., src/api/CLAUDE.md                 │
├─────────────────────────────────────────────┤
│  2. Project-level CLAUDE.md                 │  ← shared team instructions
│     e.g., .claude/CLAUDE.md or ./CLAUDE.md  │
├─────────────────────────────────────────────┤
│  1. User-level CLAUDE.md                    │  ← personal preferences
│     e.g., ~/.claude/CLAUDE.md               │
└─────────────────────────────────────────────┘
```

### User-Level (`~/.claude/CLAUDE.md`)

- Stored in the user's home directory — **not in version control**
- Contains personal preferences: editor style, preferred language, formatting quirks
- **Not shared with teammates** — this is the key exam point
- Loaded for every project the user works on

### Project-Level (`.claude/CLAUDE.md` or root `CLAUDE.md`)

- Lives in the repository root, either as `./CLAUDE.md` or `.claude/CLAUDE.md`
- **Version controlled** — shared with all team members
- Contains team coding standards, architectural rules, project conventions
- This is the primary location for team-shared instructions

### Directory-Level (subdirectory `CLAUDE.md`)

- Placed in any subdirectory (e.g., `packages/api/CLAUDE.md`)
- Only loaded when Claude Code is working within that directory or its children
- Useful for monorepos where different packages have different conventions
- Overrides or supplements project-level instructions for that subtree

---

## `@import` Syntax

For large projects, a single CLAUDE.md can become unwieldy. The `@import` directive lets you pull in content from other files, keeping your configuration modular.

```markdown
# CLAUDE.md

## General
Follow our TypeScript style guide.

@import docs/coding-standards.md
@import docs/api-conventions.md
@import docs/testing-guidelines.md
```

Key behaviors:
- Paths are **relative to the file containing the `@import`**
- Imported files are inlined at the position of the `@import` directive
- You can import any `.md` file — it does not need to be named `CLAUDE.md`
- Useful for sharing standards documents that are also human-readable in docs/

---

## `.claude/rules/` Directory

Instead of (or in addition to) a monolithic CLAUDE.md, you can place individual rule files in `.claude/rules/`. Each file focuses on a single topic.

```
.claude/
├── CLAUDE.md              # high-level project overview
└── rules/
    ├── typescript.md      # TypeScript coding conventions
    ├── testing.md         # testing standards
    ├── git-workflow.md    # commit message format, branching
    └── api-design.md     # REST API conventions
```

- All `.md` files in `.claude/rules/` are loaded automatically
- No special frontmatter is required for unconditional loading (see [[Path-Specific Rules]] for conditional rules with `paths` frontmatter)
- Easier to maintain than a single large file — each rule file has a clear owner and scope
- Version controlled and shared with the team

---

## Diagnosing Configuration Issues

### The `/memory` Command

Use `/memory` in a Claude Code session to verify which memory files are currently loaded. This command lists:

- All CLAUDE.md files in the hierarchy that were found and loaded
- All `.claude/rules/` files that were loaded
- Any `@import`-ed files

This is the **primary debugging tool** for configuration hierarchy issues.

### Common Issues

| Symptom | Likely Cause |
|---------|-------------|
| Teammate doesn't see team conventions | Instructions are in `~/.claude/CLAUDE.md` (user-level) instead of project-level |
| Rules apply in wrong directory | CLAUDE.md placed at wrong level in the hierarchy |
| Imported file not loading | Wrong relative path in `@import` |
| Too much context loaded | Monolithic CLAUDE.md instead of path-specific rules |

---

## Examples

### Example 1: Project-Level CLAUDE.md with Team Coding Standards

A standard project-level file that all team members share via version control:

```markdown
# CLAUDE.md

## Project Overview
This is the Acme SaaS platform — a multi-tenant B2B application built with
TypeScript, React, and PostgreSQL.

## Coding Standards
- Use TypeScript strict mode; never use `any` without a justifying comment
- Prefer named exports over default exports
- All public functions must have JSDoc comments
- Use `zod` for runtime validation at API boundaries

## Architecture
- Backend follows hexagonal architecture: domain → application → infrastructure
- Frontend uses feature-based folder structure under src/features/
- All database access goes through repository classes, never direct queries

## Git Conventions
- Commit messages follow Conventional Commits: `feat:`, `fix:`, `chore:`, etc.
- PRs must have a description and link to the relevant issue
```

### Example 2: `@import` for Per-Package Standards in a Monorepo

A root CLAUDE.md that pulls in package-specific standards:

```markdown
# CLAUDE.md (root)

## Monorepo Overview
This monorepo contains three packages: `api`, `web`, and `shared`.

@import packages/api/docs/api-standards.md
@import packages/web/docs/frontend-standards.md
@import packages/shared/docs/shared-types.md
```

Each imported file is a standalone document that developers can also read directly:

```markdown
<!-- packages/api/docs/api-standards.md -->
# API Package Standards

- All endpoints return `{ data, error, metadata }` envelope
- Use `express-async-errors` — never write manual try/catch in handlers
- Rate limiting is applied at the router level via middleware
- All new endpoints require an integration test in __tests__/integration/
```

This keeps standards co-located with their packages while still loading everything into Claude Code's context.

### Example 3: Splitting a Monolithic CLAUDE.md into `.claude/rules/`

**Before** — a 400-line CLAUDE.md that covers everything:

```markdown
# CLAUDE.md
## TypeScript Standards
... (80 lines)
## Testing Standards
... (60 lines)
## API Design
... (70 lines)
## Database Conventions
... (50 lines)
## CI/CD Notes
... (40 lines)
## Security Rules
... (100 lines)
```

**After** — a lean CLAUDE.md with focused rule files:

```markdown
# CLAUDE.md
## Project Overview
Acme SaaS platform. See .claude/rules/ for detailed conventions.

## Quick Reference
- Language: TypeScript (strict)
- Framework: Next.js 14 (App Router)
- Database: PostgreSQL via Prisma
- Testing: Vitest + Playwright
```

```
.claude/rules/
├── typescript.md       # 80 lines of TS conventions
├── testing.md          # 60 lines of testing standards
├── api-design.md       # 70 lines of API rules
├── database.md         # 50 lines of DB conventions
├── ci-cd.md            # 40 lines of pipeline notes
└── security.md         # 100 lines of security rules
```

Each rule file is self-contained. Team members can edit `security.md` without merge conflicts on unrelated testing changes.

### Example 4: Debugging — New Team Member Not Seeing Instructions

**Scenario:** A new developer joins the team. They report that Claude Code doesn't follow the team's commit message format or coding style. Other developers don't have this issue.

**Diagnosis steps:**

1. Ask the developer to run `/memory` in their Claude Code session
2. Check which files are listed in the output

**Root cause:** The senior developer who set up the original conventions put them in `~/.claude/CLAUDE.md` (their personal user-level config) instead of the project-level `.claude/CLAUDE.md`.

**Fix:**

```bash
# Move instructions from user-level to project-level
# On the senior dev's machine:
cat ~/.claude/CLAUDE.md

# Copy relevant team rules to the project
# In the repo:
mkdir -p .claude
cp team-rules.md .claude/CLAUDE.md   # or edit directly

# Commit to version control
git add .claude/CLAUDE.md
git commit -m "chore: move team coding standards to project-level CLAUDE.md"
```

**Key takeaway:** User-level config (`~/.claude/CLAUDE.md`) is never shared. Anything the team needs must be in project-level config (`.claude/CLAUDE.md` or root `CLAUDE.md`) or `.claude/rules/`.

---

## Exam Tips

- If a question describes teammates seeing different behavior, think **user-level vs. project-level**.
- `@import` paths are **relative to the importing file**, not the repo root.
- `/memory` is the command to verify what's loaded — remember this for troubleshooting questions.
- `.claude/rules/` files load automatically without needing `@import` in CLAUDE.md.

---

## Related Topics

- [[Path-Specific Rules]] — adding `paths` frontmatter to `.claude/rules/` files for conditional loading
- [[Custom Slash Commands and Skills]] — another extension point in the `.claude/` directory
- [[CI/CD Integration]] — CLAUDE.md provides context to CI-invoked Claude Code
- [[Domain 5 - Context Management]] — CLAUDE.md content consumes context window tokens
