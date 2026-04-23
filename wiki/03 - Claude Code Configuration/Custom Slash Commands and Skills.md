---
tags:
  - claude-code
  - slash-commands
  - skills
  - extensibility
  - task-3-2
domain: "3 - Claude Code Configuration & Workflows"
task: "3.2"
---

# Custom Slash Commands and Skills

> **Task 3.2** — Create project-scoped and user-scoped slash commands. Build skills with `context: fork`, `allowed-tools`, and `argument-hint` frontmatter.

---

## Overview

Claude Code supports two extensibility mechanisms beyond CLAUDE.md:

1. **Slash Commands** — markdown templates invoked with `/command-name`, providing reusable prompts
2. **Skills** — enhanced markdown files in `.claude/skills/` with SKILL.md frontmatter, supporting isolated execution and tool restrictions

Both allow you to encode complex, multi-step workflows into reusable actions.

---

## Slash Commands

### Project-Scoped Commands (`.claude/commands/`)

- Stored in `.claude/commands/` in the repository
- **Version controlled** — shared with the entire team
- Invoked as `/project:command-name` or simply `/command-name`
- Each command is a single `.md` file whose content becomes the prompt

```
.claude/commands/
├── review.md          → /review
├── add-tests.md       → /add-tests
├── migrate-db.md      → /migrate-db
└── deploy-check.md    → /deploy-check
```

### User-Scoped Commands (`~/.claude/commands/`)

- Stored in the user's home directory
- **Not version controlled** — personal to the individual developer
- Invoked the same way but only available to that user
- Good for personal workflows: `/my-standup`, `/scratch-pad`

### Command File Format

A command file is plain markdown. The entire content is sent as the prompt when the command is invoked. You can use a `$ARGUMENTS` placeholder to accept user input.

```markdown
<!-- .claude/commands/review.md -->
Review the current git diff for:
1. Logic errors or bugs
2. Missing error handling
3. Performance issues
4. Security vulnerabilities

Focus on the changes only — do not review unchanged code.
If you find issues, rate each as critical/warning/info.

$ARGUMENTS
```

Usage: `/review focus on the authentication changes`

---

## Skills

Skills are an evolution of slash commands with additional capabilities defined through YAML frontmatter in a `SKILL.md` file pattern.

### Location

```
.claude/skills/
├── analyze-codebase/
│   └── SKILL.md
├── generate-migration/
│   └── SKILL.md
└── security-audit/
    └── SKILL.md
```

### Frontmatter Fields

| Field | Purpose |
|-------|---------|
| `context: fork` | Runs the skill in an **isolated sub-agent context** — the skill's conversation does not pollute the main session |
| `allowed-tools` | Restricts which tools the skill can use during execution (e.g., only file reads, no shell commands) |
| `argument-hint` | Provides a prompt description shown to the user when invoking the skill, indicating what arguments are expected |

### `context: fork`

When a skill runs with `context: fork`, it executes in a **separate sub-agent**. This means:

- The skill's internal chain-of-thought and intermediate outputs do not appear in the main conversation
- The main session only sees the skill's final result
- Useful for verbose, exploratory tasks where intermediate steps would clutter the session
- The forked context has its own tool call history

### `allowed-tools`

Restricts the tools available during skill execution. This is a security and focus mechanism:

- Prevents a skill from making unintended changes (e.g., a read-only analysis skill cannot modify files)
- Limits blast radius — a skill meant to write to one location cannot accidentally execute shell commands
- Specified as a list of tool names in the frontmatter

### `argument-hint`

Tells the user what input the skill expects. When the user invokes the skill, this hint is displayed as a prompt placeholder.

---

## Skills vs. CLAUDE.md: When to Use Each

| Characteristic | Skills (on-demand) | CLAUDE.md (always-loaded) |
|---------------|-------------------|--------------------------|
| **Loading** | Only when invoked | Every session, automatically |
| **Context cost** | Zero until invoked | Always consumes tokens |
| **Use case** | Specific workflows, audits, generators | General coding standards, architecture rules |
| **Isolation** | Can fork into sub-agent | Part of main context |
| **Frequency** | Occasional, targeted tasks | Every interaction |

**Rule of thumb:** If the instruction applies to *every* interaction (e.g., "use TypeScript strict mode"), put it in CLAUDE.md or `.claude/rules/`. If it's a *specific workflow* triggered on demand (e.g., "audit this module for security issues"), make it a skill.

---

## Examples

### Example 1: Creating a `/review` Command

**File:** `.claude/commands/review.md`

```markdown
You are performing a code review on the current changes. Follow these steps:

1. Run `git diff --cached` (or `git diff` if nothing is staged) to see the changes
2. For each changed file, evaluate:
   - **Correctness**: Are there logic errors, off-by-one errors, or incorrect assumptions?
   - **Error handling**: Are errors caught and handled appropriately? Are edge cases covered?
   - **Performance**: Are there N+1 queries, unnecessary re-renders, or O(n^2) operations?
   - **Security**: Is user input validated? Are there injection vectors?
3. Output a structured review:

## Review Summary
- **Files reviewed**: (list)
- **Critical issues**: (count)
- **Warnings**: (count)

## Issues Found
(For each issue, include file, line, severity, and suggested fix)

$ARGUMENTS
```

**Usage:**

```
> /review only look at the backend changes
```

This command is version controlled in `.claude/commands/` so every team member can use `/review` with the same standards.

### Example 2: Skill with `context: fork` for Codebase Analysis

**File:** `.claude/skills/analyze-codebase/SKILL.md`

```yaml
---
context: fork
argument-hint: "Describe the aspect of the codebase to analyze (e.g., 'dependency graph', 'error handling patterns')"
---
```

```markdown
# Codebase Analysis Skill

Perform a deep analysis of the codebase based on the user's request.

## Process
1. Use `find` and `grep` to discover all relevant files
2. Read key files to understand patterns and architecture
3. Build a mental model of the component relationships
4. Identify patterns, anti-patterns, and inconsistencies

## Output Format
Produce a concise report with:
- **Architecture overview**: How components are organized
- **Key patterns identified**: Common patterns across the codebase
- **Inconsistencies**: Places where conventions are not followed
- **Recommendations**: Specific, actionable improvements

Be thorough in your discovery — read as many files as needed.
Since this runs in a forked context, verbose exploration will not clutter
the user's main session.

$ARGUMENTS
```

**Why `context: fork`?** This skill reads dozens of files and performs verbose discovery. Running it in a forked sub-agent means the main conversation only sees the final report, not the hundreds of intermediate file reads and grep results.

### Example 3: Skill with `allowed-tools` Restricting to File Write Only

**File:** `.claude/skills/generate-migration/SKILL.md`

```yaml
---
allowed-tools:
  - Read
  - Write
  - Glob
argument-hint: "Describe the database schema change (e.g., 'add email_verified boolean column to users table')"
---
```

```markdown
# Database Migration Generator

Generate a database migration file based on the requested schema change.

## Process
1. Read the current schema from `prisma/schema.prisma` (or equivalent)
2. Read recent migrations in `prisma/migrations/` to understand naming conventions
3. Generate a new migration file following the project's conventions

## Constraints
- Generate ONLY the migration file — do not run it
- Follow the existing naming pattern (timestamp prefix)
- Include both `up` and `down` migrations
- Add a comment block explaining what the migration does

$ARGUMENTS
```

**Why `allowed-tools`?** This skill is restricted to `Read`, `Write`, and `Glob`. It **cannot** execute shell commands (`Bash`), so it cannot accidentally run `prisma migrate` or modify the database. It can only read the schema, read existing migrations, and write a new migration file. This limits the blast radius to file operations only.

### Example 4: `argument-hint` Usage

**File:** `.claude/skills/security-audit/SKILL.md`

```yaml
---
context: fork
allowed-tools:
  - Read
  - Grep
  - Glob
  - Bash
argument-hint: "Path to the module or directory to audit (e.g., 'src/auth/' or 'packages/api/src/middleware/')"
---
```

```markdown
# Security Audit Skill

Perform a focused security audit on the specified module or directory.

## Checks
1. **Authentication**: Verify auth middleware is applied to all protected routes
2. **Input validation**: Check that all user inputs are validated with zod/joi
3. **SQL injection**: Look for raw SQL queries without parameterization
4. **XSS**: Check for `dangerouslySetInnerHTML` or unescaped user content
5. **Secrets**: Scan for hardcoded API keys, passwords, or tokens
6. **Dependencies**: Check for known vulnerable packages in the module's imports

## Output
Produce a security report:
- **Critical**: Must fix before merge
- **High**: Fix in current sprint
- **Medium**: Track in backlog
- **Low**: Informational

$ARGUMENTS
```

When the user types `/security-audit`, Claude Code displays the hint:

```
> /security-audit
  Path to the module or directory to audit (e.g., 'src/auth/' or 'packages/api/src/middleware/')
> /security-audit src/auth/
```

The `argument-hint` guides the user to provide the right input without needing to read the skill's documentation.

---

## Exam Tips

- **Project-scoped** = `.claude/commands/` (version controlled, team-shared). **User-scoped** = `~/.claude/commands/` (personal).
- `context: fork` isolates a skill into a sub-agent — the main session only sees the final output.
- `allowed-tools` is a **restriction** mechanism — it limits what tools the skill can access, not what tools it gains.
- Skills live in `.claude/skills/` with a `SKILL.md` file; slash commands live in `.claude/commands/` as plain `.md` files.
- The key decision: "Does every session need this?" → CLAUDE.md. "Is this an on-demand workflow?" → Skill or command.

---

## Related Topics

- [[CLAUDE.md Configuration Hierarchy]] — the always-loaded instructions that complement on-demand skills
- [[Path-Specific Rules]] — another way to conditionally load instructions
- [[Plan Mode vs Direct Execution]] — skills with `context: fork` relate to the Explore subagent concept
- [[Iterative Refinement Techniques]] — skills can encode iterative workflows as reusable templates
