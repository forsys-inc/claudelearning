---
tags:
  - domain-3
  - task-3-6
  - ci-cd
  - automation
  - non-interactive
  - print-flag
  - json-output
  - claude-md
  - code-review
domain: "3 - Claude Code Configuration & Workflows"
task: "3.6"
---

# CI/CD Integration

> **Task 3.6** — Configure Claude Code for non-interactive CI/CD pipelines using the `-p` flag, structured output formats, CLAUDE.md for project context, and session isolation strategies for effective automated review and test generation.

---

## Non-Interactive Mode with `-p`

The `-p` (or `--print`) flag runs Claude Code in **non-interactive mode** — it processes the prompt, produces output, and exits. This is mandatory for CI/CD pipelines where there is no terminal for interactive input.

```bash
# Basic non-interactive invocation
claude -p "Analyze this PR for security issues"

# Without -p, Claude Code opens an interactive session and the CI job hangs
# waiting for user input that will never come.
```

**Key behavior differences in `-p` mode:**
- No interactive prompts or confirmations
- Output goes to stdout (pipeable)
- Process exits with a return code when complete
- No session persistence (each invocation is independent)

---

## Structured Output with `--output-format` and `--json-schema`

For CI pipelines that need to parse Claude's output programmatically, two flags control output structure:

- **`--output-format json`** — Wraps the response in a JSON envelope with metadata (cost, token usage, etc.)
- **`--json-schema <schema>`** — Constrains the response content to match a specific JSON schema, enabling machine-parseable structured findings

```bash
# JSON output with schema for PR review findings
claude -p "Review this diff for issues" \
  --output-format json \
  --json-schema '{"type":"object","properties":{"findings":{"type":"array","items":{"type":"object","properties":{"file":{"type":"string"},"line":{"type":"integer"},"severity":{"type":"string","enum":["critical","high","medium","low"]},"message":{"type":"string"}},"required":["file","line","severity","message"]}}},"required":["findings"]}'
```

---

## CLAUDE.md as CI Context Mechanism

CLAUDE.md files provide project-level context to Claude Code. In CI, this is the primary mechanism for communicating:

- **Testing standards** — What framework to use, coverage expectations, naming conventions
- **Review criteria** — What to flag vs. ignore, project-specific acceptable patterns
- **Architecture constraints** — Patterns that are intentional and should not be flagged

Since CI invocations are stateless (no prior conversation), CLAUDE.md is the *only* way to provide persistent project context.

---

## Session Context Isolation

A critical principle for CI-based review:

> **The same session that generated code is less effective at reviewing its own changes.**

When Claude generates code in one session and then reviews it in the same session, it retains the reasoning context from generation. This makes it less likely to question its own decisions — it "remembers" why it made each choice and is biased toward confirming them.

**For effective CI review, the review session must be independent of the generation session.** This means:
- Separate invocations (each `-p` call is already a fresh session)
- Do not pass the generation conversation history to the reviewer
- The reviewer should see only the code artifact, not the reasoning that produced it

---

## Avoiding Duplicate Review Comments

When CI re-runs a review after new commits, Claude may re-flag issues that were already reported. To prevent noisy duplicate comments:

- **Include prior review findings** in the prompt so Claude can skip already-reported issues
- **Provide existing test files** so test generation does not suggest tests that already exist

---

## Examples

### Example 1 — CI Script Using `-p` Flag to Prevent Interactive Hangs

A GitHub Actions workflow that runs Claude Code as a PR reviewer:

```yaml
# .github/workflows/claude-review.yml
name: Claude PR Review
on:
  pull_request:
    types: [opened, synchronize]

jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Get PR diff
        run: git diff origin/${{ github.base_ref }}...HEAD > /tmp/pr_diff.txt

      - name: Run Claude review
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
        run: |
          claude -p "Review the following PR diff for security issues, \
            performance problems, and correctness bugs. Focus on changes \
            only — do not comment on unchanged surrounding code.

            $(cat /tmp/pr_diff.txt)" \
            --output-format json > /tmp/review_output.json

      - name: Post review comments
        run: python scripts/post_review_comments.py /tmp/review_output.json
```

**Why `-p` is essential:** Without it, the `claude` invocation opens an interactive REPL. The CI runner has no TTY, so the process either hangs indefinitely or crashes. The `-p` flag ensures Claude processes the prompt and exits cleanly.

```bash
# This hangs in CI (no TTY for interactive input):
claude "Analyze this PR"

# This works in CI (non-interactive, outputs to stdout):
claude -p "Analyze this PR"

# This works and produces structured output:
claude -p "Analyze this PR" --output-format json
```

---

### Example 2 — `--output-format json` with `--json-schema` for Machine-Parseable Findings

A CI pipeline that produces structured findings and posts them as inline PR comments:

```bash
#!/bin/bash
# scripts/structured_review.sh

SCHEMA='{
  "type": "object",
  "properties": {
    "findings": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "file": { "type": "string" },
          "line": { "type": "integer" },
          "severity": {
            "type": "string",
            "enum": ["critical", "high", "medium", "low"]
          },
          "category": {
            "type": "string",
            "enum": ["security", "performance", "correctness", "style"]
          },
          "message": { "type": "string" },
          "suggestion": { "type": "string" }
        },
        "required": ["file", "line", "severity", "category", "message"]
      }
    },
    "summary": { "type": "string" }
  },
  "required": ["findings", "summary"]
}'

claude -p "Review the following diff. Report each finding with file path, \
  line number, severity, category, message, and optional suggestion.

  $(git diff origin/main...HEAD)" \
  --output-format json \
  --json-schema "$SCHEMA" > findings.json
```

The downstream script then parses `findings.json` and uses the GitHub API to post inline comments at the exact file/line for each finding:

```python
# scripts/post_inline_comments.py
import json, os, requests

with open("findings.json") as f:
    data = json.load(f)

for finding in data["result"]["findings"]:
    # Post inline comment via GitHub PR Review API
    requests.post(
        f"https://api.github.com/repos/{os.environ['REPO']}/pulls/{os.environ['PR_NUM']}/comments",
        headers={"Authorization": f"token {os.environ['GITHUB_TOKEN']}"},
        json={
            "body": f"**[{finding['severity'].upper()}] {finding['category']}**: {finding['message']}"
                   + (f"\n\n**Suggestion:** {finding['suggestion']}" if finding.get('suggestion') else ""),
            "commit_id": os.environ["HEAD_SHA"],
            "path": finding["file"],
            "line": finding["line"],
            "side": "RIGHT"
        }
    )
```

The `--json-schema` flag ensures Claude's output is machine-parseable without fragile text parsing. Every finding has a guaranteed structure with required fields.

---

### Example 3 — Including Prior Review Findings When Re-Running After New Commits

When a developer pushes new commits to address review feedback, the CI pipeline re-runs the review. Without context about prior findings, Claude may re-flag issues that were already reported (creating duplicate noise) or re-flag issues the developer intentionally kept unchanged.

```bash
#!/bin/bash
# scripts/incremental_review.sh

# Fetch prior review findings from the PR's existing comments
gh api "repos/${REPO}/pulls/${PR_NUM}/comments" \
  --jq '[.[] | select(.user.login == "claude-bot") | {file: .path, line: .line, message: .body}]' \
  > prior_findings.json

# Include prior findings in the prompt
claude -p "Review the following diff for issues.

IMPORTANT: The following issues were already reported in prior review passes.
Do NOT report these again. Only report NEW issues or issues in NEWLY CHANGED
lines. If a prior finding has been addressed by the new commits, you may note
that it was resolved.

Prior findings (already reported):
$(cat prior_findings.json)

New diff since last review:
$(git diff $LAST_REVIEWED_SHA...HEAD)

Full current diff (for context):
$(git diff origin/main...HEAD)" \
  --output-format json > new_findings.json
```

This pattern ensures:
1. Already-reported issues are not duplicated
2. Claude can note when prior findings have been resolved
3. Review focuses on new changes, reducing noise

---

### Example 4 — CLAUDE.md Testing Standards Section for CI-Invoked Test Generation

A project's CLAUDE.md includes a testing standards section that provides context to CI-invoked test generation:

```markdown
# CLAUDE.md

## Testing Standards

### Framework and Structure
- Use `vitest` for all unit and integration tests
- Test files live alongside source: `src/foo.ts` → `src/foo.test.ts`
- Use `describe` blocks grouped by method name
- Use `it` with descriptive names: `it("returns 404 when user not found")`

### Coverage Requirements
- All new public functions must have tests for: happy path, error case,
  edge case (null/undefined/empty input)
- Database interactions must use the test transaction wrapper from
  `tests/helpers/db.ts` — never hit the real database

### Patterns to Follow
- Use `vi.mock()` for external service dependencies
- Use factory functions from `tests/factories/` for test data — do not
  inline large object literals
- Assertions should test behavior, not implementation — do not assert
  on internal method calls unless testing side effects

### Patterns to Avoid
- Do not generate tests for private/internal functions
- Do not use `any` type in test code
- Do not use snapshot tests for API responses (they are brittle)

### Existing Test Utilities
- `tests/helpers/db.ts` — Transaction wrapper, test database setup
- `tests/helpers/auth.ts` — Mock authentication tokens and sessions
- `tests/factories/user.ts` — User factory with sensible defaults
- `tests/factories/order.ts` — Order factory with related entities
```

When CI invokes Claude for test generation:

```bash
# CI test generation script
claude -p "Generate tests for the following new file. Follow the project's
testing standards in CLAUDE.md. Use existing test factories and helpers
where applicable. Do not duplicate tests that already exist.

New source file:
$(cat src/payments/refund.ts)

Existing tests in this area (do not duplicate):
$(cat src/payments/payment.test.ts)

Available factories:
$(cat tests/factories/order.ts)" \
  --output-format json > generated_tests.json
```

**Why CLAUDE.md matters for CI:** Without it, Claude would use generic testing patterns — possibly `jest` instead of `vitest`, inline test data instead of factories, and snapshot tests that the team has explicitly banned. CLAUDE.md provides the project-specific constraints that make generated tests actually mergeable.

**Why existing tests are included:** Without seeing `payment.test.ts`, Claude might generate tests that overlap with existing coverage. Providing existing tests prevents duplicate test suggestions and lets Claude focus on genuinely missing coverage.

---

## Key Takeaways for the Exam

1. **`-p` / `--print` is mandatory for CI** — Without it, Claude Code opens an interactive session that hangs in headless environments.
2. **`--output-format json` + `--json-schema`** produce machine-parseable output that downstream tooling can consume without fragile text parsing.
3. **CLAUDE.md is the CI context mechanism** — It provides project standards, review criteria, and architecture context to stateless CI invocations.
4. **Session isolation matters for review** — The same session that generated code is biased toward confirming its own decisions. Independent review sessions catch more issues.
5. **Include prior findings and existing tests** to prevent duplicate comments and duplicate test suggestions in incremental CI runs.

---

## Related Topics

- [[Domain 3 - Claude Code Configuration & Workflows]] — Parent domain overview
- [[CLAUDE.md Configuration Hierarchy]] — How CLAUDE.md files are loaded and merged across directory levels
- [[Plan Mode vs Direct Execution]] — Plan mode considerations in CI vs. interactive contexts
- [[Multi-Instance Review Architectures]] — Using separate generation and review sessions for quality
- [[JSON Schemas and Tool Use]] — Schema enforcement for structured CI output
- [[Iterative Refinement Techniques]] — Iterative patterns applicable to CI review workflows
