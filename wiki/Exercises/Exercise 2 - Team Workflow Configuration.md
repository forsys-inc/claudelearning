---
tags: [exercise, domain/3, domain/2, hands-on]
domains: [3, 2]
---

# Exercise 2: Configure Claude Code for a Team Development Workflow

**Objective:** Practice configuring CLAUDE.md hierarchies, custom slash commands, path-specific rules, and MCP server integration for a multi-developer project.

**Domains reinforced:** [[Domain 3 - Claude Code Configuration & Workflows]], [[Domain 2 - Tool Design & MCP Integration]]

---

## Step 1: Create a Project-Level CLAUDE.md

Create a project-level CLAUDE.md with universal coding standards and testing conventions that apply to all team members.

```markdown
# CLAUDE.md

## Coding Standards
- Use TypeScript strict mode for all new files
- Prefer named exports over default exports
- Use async/await over raw Promises
- Error handling: always use custom error classes from src/errors/

## Testing Conventions
- Test files live alongside source: `Component.test.tsx` next to `Component.tsx`
- Use `vitest` as the test runner: `npm run test`
- Use `@testing-library/react` for component tests
- Minimum 80% branch coverage for new code
- Available fixtures: see `src/test/fixtures/` for mock data factories

## Running the Project
- `npm run dev` — start development server
- `npm run test` — run all tests
- `npm run test -- --run src/path/to/file.test.tsx` — run single test file
- `npm run lint` — ESLint + Prettier check
```

**Verification:** Have two different team members clone the repo and confirm they both receive these instructions when using Claude Code.

---

## Step 2: Create Path-Specific Rules with Glob Patterns

Create `.claude/rules/` files with YAML frontmatter for different code areas.

### API Handler Rules

```yaml
# .claude/rules/api-conventions.md
---
paths:
  - "src/api/**/*"
  - "src/routes/**/*"
---

## API Handler Conventions
- All handlers use the `asyncHandler` wrapper from `src/middleware/async.ts`
- Return standardized response format: `{ data, error, meta }`
- Validate request bodies with Zod schemas in `src/api/schemas/`
- Log all errors with structured context: `{ endpoint, method, userId, error }`
- Rate limiting applied via middleware, not in handlers
```

### Testing Rules

```yaml
# .claude/rules/testing.md
---
paths:
  - "**/*.test.tsx"
  - "**/*.test.ts"
  - "**/*.spec.ts"
---

## Testing Conventions
- Use `describe` blocks grouped by function/component name
- Use `it` (not `test`) for individual test cases
- Follow Arrange-Act-Assert pattern with blank line separators
- Mock external services using MSW (Mock Service Worker)
- Never mock the database — use test database with transactions
- Available fixtures: `createMockUser()`, `createMockOrder()` from `src/test/fixtures/`
```

### Infrastructure Rules

```yaml
# .claude/rules/terraform.md
---
paths:
  - "terraform/**/*"
  - "infra/**/*"
---

## Infrastructure Conventions
- All resources must have `environment` and `team` tags
- Use modules from `terraform/modules/` — do not create inline resources
- State stored in S3 backend — never modify state manually
- Changes require `terraform plan` output in PR description
```

**Verification:** Edit a file matching `src/api/users.ts` and confirm API conventions load. Edit a `.test.tsx` file and confirm testing conventions load instead.

---

## Step 3: Create a Skill with `context: fork` and `allowed-tools`

Create a project-scoped skill for codebase analysis that runs in isolation.

```markdown
<!-- .claude/skills/analyze-deps/SKILL.md -->
---
context: fork
allowed-tools:
  - Read
  - Glob
  - Grep
  - Bash
argument-hint: "Enter the module or package to analyze (e.g., 'src/auth')"
---

Analyze the dependency graph of the specified module. Report:

1. **Direct dependencies** — what this module imports
2. **Reverse dependencies** — what imports this module
3. **Circular dependencies** — any circular import chains
4. **External dependencies** — third-party packages used

Use Grep to trace imports, Glob to find all files in the module, and Read to examine specific files.

Output a structured summary, not the raw search results.
```

**Key points:**
- `context: fork` prevents verbose exploration output from polluting the main conversation
- `allowed-tools` restricts to read-only operations (no Write, no Edit)
- `argument-hint` prompts the developer when they invoke `/analyze-deps` without arguments

**Verification:** Run `/analyze-deps src/auth` and confirm:
1. Results appear but main conversation context stays clean
2. The skill cannot use Write or Edit tools

---

## Step 4: Configure MCP Servers

### Project-level shared server (`.mcp.json`)

```json
{
  "mcpServers": {
    "jira": {
      "command": "npx",
      "args": ["-y", "@anthropic/mcp-jira"],
      "env": {
        "JIRA_URL": "https://myteam.atlassian.net",
        "JIRA_TOKEN": "${JIRA_API_TOKEN}"
      }
    },
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "${GITHUB_TOKEN}"
      }
    }
  }
}
```

**Key points:**
- `${JIRA_API_TOKEN}` and `${GITHUB_TOKEN}` expand from environment variables — no secrets in version control
- Both servers available to all team members who set the env vars
- Uses community MCP servers rather than custom implementations

### User-level personal server (`~/.claude.json`)

```json
{
  "mcpServers": {
    "sqlite-explorer": {
      "command": "npx",
      "args": ["-y", "@anthropic/mcp-sqlite", "--db", "/Users/me/dev/local.db"]
    }
  }
}
```

**Verification:**
1. Confirm both Jira and GitHub tools appear in Claude Code's available tools
2. Confirm the personal sqlite-explorer is only available to you, not teammates

---

## Step 5: Test Plan Mode vs Direct Execution

Run three tasks of varying complexity and observe when plan mode provides value:

### Task A: Single-file bug fix (Direct Execution)
> "The `validateEmail` function in `src/utils/validation.ts` accepts emails without a TLD. Add a check for at least one dot after the @ symbol."

**Expected:** Direct execution handles this immediately. Plan mode would add unnecessary overhead for a single, clear change.

### Task B: Multi-file library migration (Plan Mode)
> "Migrate from Moment.js to date-fns across the entire codebase. We use Moment in 45+ files for date formatting, relative time display, and timezone conversions."

**Expected:** Plan mode explores the codebase, catalogs all Moment.js usage patterns, identifies edge cases (timezone handling differences), and proposes a migration strategy before changing any files.

### Task C: New feature with multiple approaches (Plan Mode)
> "Add real-time notifications to the app. We could use WebSockets, Server-Sent Events, or a polling approach. Each has different infrastructure requirements."

**Expected:** Plan mode evaluates tradeoffs between approaches given the existing architecture before committing to one.

---

## Checklist

- [ ] Project-level CLAUDE.md created and shared via version control
- [ ] Path-specific rules in `.claude/rules/` with glob patterns
- [ ] Skill created with `context: fork` and `allowed-tools` restrictions
- [ ] MCP server configured in `.mcp.json` with env var expansion
- [ ] Personal MCP server in `~/.claude.json`
- [ ] Plan mode tested on complex task, direct execution on simple task

---

## Related Topics

- [[CLAUDE.md Configuration Hierarchy]]
- [[Path-Specific Rules]]
- [[Custom Slash Commands and Skills]]
- [[MCP Server Configuration]]
- [[Plan Mode vs Direct Execution]]
