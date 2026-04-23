---
tags:
  - claude-code
  - configuration
  - path-rules
  - glob-patterns
  - task-3-3
domain: "3 - Claude Code Configuration & Workflows"
task: "3.3"
---

# Path-Specific Rules

> **Task 3.3** — Write path-scoped rules in `.claude/rules/` with YAML frontmatter `paths` fields using glob patterns for conditional activation. Understand when to use path-specific rules vs. directory-level CLAUDE.md files.

---

## Overview

Path-specific rules extend the `.claude/rules/` mechanism (see [[CLAUDE.md Configuration Hierarchy]]) by adding **conditional activation**. Instead of loading for every file in the project, a path-specific rule only activates when Claude Code is working on files that match its glob pattern.

This solves a fundamental problem: in large projects, loading every rule for every task wastes context tokens and can introduce irrelevant — or even conflicting — instructions.

---

## How Path-Specific Rules Work

### File Location and Frontmatter

Path-specific rules live in `.claude/rules/` alongside unconditional rules. The difference is a `paths` field in the YAML frontmatter:

```markdown
---
paths:
  - "src/api/**/*"
  - "src/routes/**/*"
---

# API Handler Conventions

- All handlers must validate request body with zod schemas
- Return responses using the `ApiResponse<T>` envelope type
- Log errors with the structured logger, never `console.error`
```

### Glob Pattern Syntax

The `paths` field accepts standard glob patterns:

| Pattern | Matches |
|---------|---------|
| `src/api/**/*` | All files under `src/api/`, any depth |
| `**/*.test.tsx` | All `.test.tsx` files anywhere in the project |
| `terraform/**/*` | All files under `terraform/` |
| `packages/*/src/**/*.ts` | TypeScript files in any package's `src/` dir |
| `*.config.{js,ts}` | Config files at root with `.js` or `.ts` extension |

### Activation Behavior

- Rules activate when Claude Code reads, writes, or is asked about files matching the pattern
- Multiple `paths` entries are OR'd — the rule activates if **any** pattern matches
- A rule without a `paths` field loads unconditionally (always active)

---

## Why Path-Specific Rules Matter

### Reduced Irrelevant Context

Without path-specific rules, a project with conventions for Terraform, React, API handlers, database migrations, and CI configuration would load **all** of those instructions for every task — even a simple README edit.

```
Without path rules:
  Every task loads: terraform.md + react.md + api.md + db.md + ci.md
  → ~500 lines of instructions, most irrelevant

With path rules:
  Editing src/api/users.ts loads: api.md only
  Editing terraform/main.tf loads: terraform.md only
  → ~100 lines of relevant instructions
```

### Token Savings

Each loaded rule consumes context window tokens. Path-specific rules keep the context lean, leaving more room for actual code and conversation. This is especially important in long sessions or when working with large files.

---

## Path-Specific Rules vs. Directory-Level CLAUDE.md

Both mechanisms provide directory-scoped instructions, but they differ in important ways:

| Aspect | Path-Specific Rules (`.claude/rules/`) | Directory-Level CLAUDE.md |
|--------|----------------------------------------|--------------------------|
| **Location** | Centralized in `.claude/rules/` | Distributed across directories |
| **Scope mechanism** | Glob patterns — can target multiple directories, file types | Physical location — applies to the containing directory and children |
| **Cross-directory** | Can match patterns across unrelated directories (e.g., `**/*.test.tsx`) | Must be duplicated in each directory |
| **Discoverability** | All rules visible in one directory | Scattered throughout the project tree |
| **Best for** | File-type conventions, cross-cutting concerns | Directory-specific overrides in monorepos |

**Rule of thumb:**
- Use **path-specific rules** when the convention is about a **file type** or **cross-cutting pattern** (e.g., "all test files", "all Terraform files")
- Use **directory-level CLAUDE.md** when the convention is about a **specific subtree** with its own identity (e.g., a package in a monorepo that has entirely different tech stack)

---

## Examples

### Example 1: Terraform Infrastructure Conventions

**File:** `.claude/rules/terraform.md`

```yaml
---
paths:
  - "terraform/**/*"
  - "infra/**/*.tf"
---
```

```markdown
# Terraform Conventions

## Module Structure
- Each module must have `main.tf`, `variables.tf`, `outputs.tf`, and `README.md`
- Use `locals.tf` for computed values — never inline complex expressions in resources

## Naming
- Resource names use snake_case: `aws_s3_bucket.data_lake`
- Variable names use snake_case with descriptive prefixes: `vpc_cidr_block`
- Output names match the resource attribute: `output "bucket_arn"`

## State Management
- Never modify state manually — use `terraform state mv` or `terraform import`
- All environments use remote state in S3 with DynamoDB locking
- State file key pattern: `{project}/{environment}/terraform.tfstate`

## Security
- Never hardcode credentials — use `aws_ssm_parameter` data sources
- All S3 buckets must have `server_side_encryption_configuration`
- Security groups must have explicit descriptions on every rule
```

This rule only loads when working on files under `terraform/` or `.tf` files under `infra/`. It does not consume context when editing React components or API handlers.

### Example 2: Testing Conventions for `.test.tsx` Files

**File:** `.claude/rules/testing.md`

```yaml
---
paths:
  - "**/*.test.ts"
  - "**/*.test.tsx"
  - "**/*.spec.ts"
  - "**/*.spec.tsx"
  - "__tests__/**/*"
---
```

```markdown
# Testing Conventions

## Framework
- Use Vitest for unit/integration tests, Playwright for E2E
- Prefer `describe`/`it` blocks with clear hierarchy

## Structure
- File naming: `ComponentName.test.tsx` or `module-name.test.ts`
- Group related tests in `describe` blocks named after the unit under test
- Use `beforeEach` for shared setup; avoid `beforeAll` unless dealing with expensive fixtures

## Assertions
- Prefer specific matchers: `toHaveBeenCalledWith()` over `toHaveBeenCalled()`
- Use `toMatchInlineSnapshot()` for complex object assertions
- Never use `toBeTruthy()` / `toBeFalsy()` — use `toBe(true)` or type-safe alternatives

## Mocking
- Mock at module boundaries, not internal functions
- Use `vi.mock()` at the top of the file for module mocks
- Reset mocks in `afterEach`: `vi.restoreAllMocks()`
- Prefer dependency injection over module mocking where possible

## Coverage
- New features require >80% branch coverage
- Bug fixes require a regression test that would have caught the bug
```

This rule activates for any test file in the project, regardless of directory. A glob pattern is the natural fit here — a directory-level CLAUDE.md would not work because test files are co-located with source files throughout the tree.

### Example 3: API Handler Rules

**File:** `.claude/rules/api-handlers.md`

```yaml
---
paths:
  - "src/api/**/*"
  - "src/routes/**/*"
  - "src/middleware/**/*"
---
```

```markdown
# API Handler Conventions

## Request Handling
- Every endpoint must validate the request body using a zod schema
- Extract the schema into a named constant: `const CreateUserSchema = z.object({...})`
- Use the shared `validate()` middleware rather than inline `.parse()` calls

## Response Format
All responses must use the standard envelope:
```typescript
interface ApiResponse<T> {
  data: T | null;
  error: { code: string; message: string } | null;
  metadata: { requestId: string; timestamp: string };
}
```

## Error Handling
- Use `AppError` class for operational errors (inherits from Error, includes statusCode)
- Let the global error handler format the response — do not catch and format in individual handlers
- Log errors with structured logger: `logger.error({ err, requestId, path })`

## Authentication
- All routes except `/health` and `/auth/*` require the `requireAuth` middleware
- Use `req.user` (set by auth middleware) — never decode JWT manually in handlers
- Role-based access uses `requireRole('admin')` middleware
```

### Example 4: Comparison — Path-Specific Rules vs. Subdirectory CLAUDE.md

**Scenario:** A monorepo has `packages/web/` (React) and `packages/api/` (Express). You also want consistent testing conventions across both.

**Approach A: Directory-level CLAUDE.md only**

```
packages/
├── web/
│   └── CLAUDE.md     # React conventions + testing conventions
└── api/
    └── CLAUDE.md     # Express conventions + testing conventions (duplicated!)
```

Problem: Testing conventions are **duplicated** in both CLAUDE.md files. When you update the testing standards, you must update both files.

**Approach B: Path-specific rules + directory-level CLAUDE.md (recommended)**

```
.claude/rules/
├── testing.md          # paths: ["**/*.test.*"] — shared testing conventions
├── react.md            # paths: ["packages/web/**/*"] — React-specific rules
└── express.md          # paths: ["packages/api/**/*"] — Express-specific rules

packages/
├── web/
│   └── CLAUDE.md       # Web package overview, unique local context only
└── api/
    └── CLAUDE.md       # API package overview, unique local context only
```

Benefits:
- Testing conventions are defined **once** and apply to all test files
- React rules only load when working in `packages/web/`
- Express rules only load when working in `packages/api/`
- Each package's CLAUDE.md only contains info unique to that package (local ports, env vars, build commands)
- No duplication, easy to maintain

---

## Exam Tips

- Path-specific rules use `paths` in YAML frontmatter with **glob patterns**
- They live in `.claude/rules/` alongside unconditional rules — the only difference is the presence of `paths`
- The main benefit is **reducing irrelevant context** and **saving tokens**
- For cross-cutting concerns (file-type rules like `**/*.test.tsx`), path-specific rules are superior to directory-level CLAUDE.md
- For package-specific overrides in monorepos, directory-level CLAUDE.md is appropriate
- The two approaches complement each other — use both when it makes sense

---

## Related Topics

- [[CLAUDE.md Configuration Hierarchy]] — the broader configuration system that path-specific rules extend
- [[Custom Slash Commands and Skills]] — another way to scope behavior to specific contexts
- [[Domain 5 - Context Management]] — path-specific rules help manage context window usage
- [[Iterative Refinement Techniques]] — rules can encode the patterns you discover through iteration
