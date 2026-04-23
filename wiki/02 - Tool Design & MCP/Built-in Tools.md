---
tags:
  - domain-2
  - task-2-5
  - built-in-tools
  - claude-code
  - grep
  - glob
  - edit
domain: "Tool Design & MCP Integration"
task: "2.5"
---

# Built-in Tools (Task 2.5)

> **Core principle:** Claude Code ships with a set of built-in tools for codebase exploration and modification. Knowing when to use each tool — and how to chain them for incremental understanding — is a foundational skill.

---

## Tool Overview

| Tool | Purpose | Input | Best For |
|------|---------|-------|----------|
| **Grep** | Search file *contents* by regex | Pattern + optional path/glob filter | Finding function calls, error messages, imports, string literals |
| **Glob** | Search file *paths* by pattern | Glob pattern (e.g., `**/*.tsx`) | Finding files by name, extension, or directory structure |
| **Read** | Read full file contents | File path + optional line range | Understanding a specific file after locating it |
| **Write** | Create or overwrite an entire file | File path + content | Creating new files or full rewrites |
| **Edit** | Targeted text replacement | File path + old_string + new_string | Modifying specific sections without rewriting the whole file |

---

## Grep: Content Search

Grep searches *inside* files for content matching a regex pattern. It is the primary tool for answering questions like:
- "Where is this function called?"
- "Which files import this module?"
- "Where does this error message originate?"

**Key characteristics:**
- Regex-powered: supports `functionName\(`, `import.*from ['"]module['"]`, etc.
- Can filter by file type or glob pattern.
- Returns matching lines with file paths and line numbers.
- Does NOT search file names — use Glob for that.

**When to use Grep:**
- You know a string (function name, error message, import path) and want to find where it appears.
- You need to trace usage of a symbol across the codebase.
- You are searching for patterns (e.g., all `TODO` comments, all `console.error` calls).

---

## Glob: File Path Pattern Matching

Glob searches file *paths* (names, extensions, directory positions) using glob syntax. It does not look inside files.

**Common patterns:**
- `**/*.test.tsx` — all test files with `.tsx` extension anywhere in the tree
- `src/components/**/*.tsx` — all `.tsx` files under `src/components/`
- `**/index.ts` — all `index.ts` files at any depth
- `src/**/*utils*` — all files with "utils" in the name under `src/`

**When to use Glob:**
- You know the file name or extension but not the directory.
- You want to understand project structure (e.g., "what test files exist?").
- You need to find configuration files (`**/*.config.js`, `**/.env*`).

---

## Read: Full File Access

Read retrieves the contents of a specific file. It supports line ranges for large files.

**When to use Read:**
- After Grep or Glob has identified a file you need to understand.
- You need to see the full context around a Grep match (Grep shows lines, Read shows the file).
- You need to understand a file's structure, imports, and exports.

---

## Write: Full File Operations

Write creates a new file or completely overwrites an existing one.

**When to use Write:**
- Creating a new file that does not exist yet.
- A file needs so many changes that Edit would require too many individual operations.
- **Fallback:** When Edit fails (see below), Read + Write can accomplish the same change.

---

## Edit: Targeted Modifications

Edit replaces a specific string in a file with a new string. It requires the `old_string` to be **unique** within the file — if the same text appears in multiple places, Edit fails.

**When to use Edit:**
- Changing a function signature, fixing a bug, updating a constant.
- Adding an import line, modifying a configuration value.
- Any targeted change where you know the exact text to replace.

**Edit failure modes:**
- `old_string` is not found (typo, wrong indentation, file changed).
- `old_string` appears multiple times (not unique).

**Fallback pattern:** When Edit fails, use Read to get the current file contents, make the modification in memory, and use Write to save the full updated file.

---

## Incremental Codebase Understanding

The most powerful pattern is chaining these tools to build understanding step by step:

```
Grep (find entry points)
  → Read (understand the entry point file)
    → Grep (find what it imports/calls)
      → Read (understand the next file)
        → ... (continue until you understand the flow)
```

This mimics how a developer explores an unfamiliar codebase:
1. Search for a known symbol (Grep).
2. Open the file to understand context (Read).
3. Follow imports and function calls (Grep again).
4. Repeat until the picture is complete.

---

## Tracing Function Usage Across Wrapper Modules

In large codebases, functions are often re-exported through wrapper/barrel modules:

```
src/utils/api.ts         → exports `fetchUser`
src/utils/index.ts       → re-exports from `./api`
src/components/Profile.tsx → imports from `../utils`
```

Grepping for `fetchUser` might only find `api.ts` and `Profile.tsx`, missing the barrel file. To trace the full chain, you may need to:
1. Grep for the function name to find direct references.
2. Read the barrel/index files in the same directory to see re-exports.
3. Grep for the barrel module path to find indirect consumers.

---

## Examples

### Example 1: Using Grep to Find All Callers of a Function

**Scenario:** You need to understand the impact of changing the `validateToken` function's signature.

**Step 1: Find where `validateToken` is defined:**

```
Grep pattern: "function validateToken|const validateToken|validateToken\s*="
```

Result:
```
src/auth/token.ts:24:  export async function validateToken(token: string): Promise<TokenPayload> {
```

**Step 2: Find all callers:**

```
Grep pattern: "validateToken\("
```

Result:
```
src/auth/token.ts:24:    export async function validateToken(token: string): Promise<TokenPayload> {
src/middleware/auth.ts:15:    const payload = await validateToken(req.headers.authorization);
src/api/routes/user.ts:42:    const user = await validateToken(bearerToken);
src/api/routes/admin.ts:18:    const admin = await validateToken(token);
src/tests/auth.test.ts:55:    const result = await validateToken(mockToken);
src/tests/auth.test.ts:72:    await expect(validateToken(expiredToken)).rejects.toThrow();
```

**Step 3: Find who imports it (to understand module boundaries):**

```
Grep pattern: "import.*validateToken"
```

Result:
```
src/middleware/auth.ts:3:    import { validateToken } from '../auth/token';
src/api/routes/user.ts:5:    import { validateToken } from '../../auth/token';
src/api/routes/admin.ts:4:    import { validateToken } from '../../auth/token';
src/tests/auth.test.ts:2:    import { validateToken } from '../auth/token';
```

Now you know exactly which files need updating if the signature changes: 3 production files and 1 test file.

---

### Example 2: Using Glob to Find Test Files

**Scenario:** You need to find all React component test files to add a new testing utility import.

```
Glob pattern: **/*.test.tsx
```

Result:
```
src/components/__tests__/Button.test.tsx
src/components/__tests__/Modal.test.tsx
src/components/__tests__/Sidebar.test.tsx
src/features/auth/__tests__/LoginForm.test.tsx
src/features/auth/__tests__/SignupFlow.test.tsx
src/features/dashboard/__tests__/DashboardLayout.test.tsx
src/features/dashboard/__tests__/MetricsCard.test.tsx
```

Now you can Read each file to check which ones already import the utility, and Edit the ones that do not.

**Follow-up — find which test files already use the utility:**

```
Grep pattern: "import.*renderWithProviders" glob: "**/*.test.tsx"
```

Result:
```
src/components/__tests__/Modal.test.tsx:3:    import { renderWithProviders } from '../../test-utils';
src/features/auth/__tests__/LoginForm.test.tsx:4:    import { renderWithProviders } from '../../../test-utils';
```

Two files already have it. The other five need the import added.

---

### Example 3: Edit Failure into Read + Write Fallback

**Scenario:** You need to change a default timeout from 5000 to 10000 in a config file.

**Attempt 1: Edit**

```
Edit file: src/config/defaults.ts
old_string: "timeout: 5000"
new_string: "timeout: 10000"
```

**Edit fails:** `old_string "timeout: 5000" is not unique — found 3 occurrences.`

The file has multiple timeout values:
```typescript
// src/config/defaults.ts
export const defaults = {
  api: {
    timeout: 5000,       // line 3
    retryTimeout: 5000,  // line 4 — also matches!
  },
  db: {
    timeout: 5000,       // line 7 — also matches!
  }
};
```

**Fallback: Read + Write**

**Step 1: Read the full file:**

```
Read file: src/config/defaults.ts
```

**Step 2: Write with the precise change (only the API timeout):**

```
Write file: src/config/defaults.ts
Content:
export const defaults = {
  api: {
    timeout: 10000,      // Changed from 5000
    retryTimeout: 5000,
  },
  db: {
    timeout: 5000,
  },
};
```

**Alternative fix: Use more context in Edit to make `old_string` unique:**

```
Edit file: src/config/defaults.ts
old_string: "api: {\n    timeout: 5000,"
new_string: "api: {\n    timeout: 10000,"
```

By including surrounding context (`api: {`), the `old_string` becomes unique and Edit succeeds. This is often preferable to a full Read + Write because it is a smaller, more reviewable change.

---

### Example 4: Incremental Exploration — Grep to Read to Trace Flow

**Scenario:** A bug report says "payment processing fails silently." You need to trace the payment flow.

**Step 1: Grep for the entry point:**

```
Grep pattern: "processPayment|handlePayment|submitPayment"
```

Result:
```
src/api/routes/checkout.ts:34:    const result = await processPayment(order);
src/services/payment.ts:12:    export async function processPayment(order: Order): Promise<PaymentResult> {
src/services/payment.ts:45:    async function handlePaymentResponse(response: GatewayResponse) {
```

**Step 2: Read the payment service to understand the flow:**

```
Read file: src/services/payment.ts
```

You discover that `processPayment` calls `gateway.charge()`, then passes the result to `handlePaymentResponse`, which calls `recordTransaction()`.

**Step 3: Grep for `recordTransaction` to find where it lives:**

```
Grep pattern: "export.*recordTransaction|function recordTransaction"
```

Result:
```
src/services/transactions.ts:8:    export async function recordTransaction(payment: PaymentResult): Promise<void> {
```

**Step 4: Read the transaction service:**

```
Read file: src/services/transactions.ts
```

You find the bug: `recordTransaction` has a try/catch that swallows errors:

```typescript
export async function recordTransaction(payment: PaymentResult): Promise<void> {
  try {
    await db.transactions.insert(payment);
  } catch (error) {
    // BUG: error is swallowed silently
    console.log("Transaction recording failed");
    // Should be: throw error; or at minimum: console.error(error);
  }
}
```

**Step 5: Grep to check if this pattern exists elsewhere:**

```
Grep pattern: "catch.*\{[\s]*console\.log"
```

This finds other locations where errors might be silently swallowed, allowing you to fix them all.

**The chain:** Grep (find entry) -> Read (understand flow) -> Grep (trace calls) -> Read (find bug) -> Grep (find similar bugs). Each step narrows the search based on what you learned in the previous step.

---

## Tool Selection Quick Reference

| I want to... | Use |
|--------------|-----|
| Find where a function is called | Grep |
| Find files with a certain extension | Glob |
| Understand a specific file | Read |
| Make a targeted change to one spot | Edit |
| Create a new file | Write |
| Fix an Edit that failed | Read + Write |
| Trace a code path I don't understand | Grep -> Read -> Grep -> Read (chain) |
| Find all config files | Glob (`**/*.config.*`) |
| Find all uses of an error message | Grep |
| Rename a variable in one file | Edit with `replace_all` |

---

## Summary Checklist

- [ ] Use Grep for content search (inside files), Glob for path search (file names)
- [ ] Always Read a file before editing it (Edit requires reading first)
- [ ] When Edit fails due to non-unique text, add more surrounding context or fall back to Read + Write
- [ ] Chain tools incrementally: Grep to find, Read to understand, Grep to trace further
- [ ] Use Glob to understand project structure before diving into individual files
- [ ] When tracing function usage, check for barrel/index re-export files

---

## Related Topics

- [[Domain 2 - Tool Design & MCP Integration]] — domain overview
- [[Tool Interface Design]] — how built-in tool descriptions are structured
- [[MCP Server Configuration]] — MCP tools that complement built-in tools
- [[Tool Distribution and Tool Choice]] — built-in tools always available alongside MCP tools
- [[Domain 5 - Context Management]] — managing context when Read returns large files
