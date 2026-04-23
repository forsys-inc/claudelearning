---
tags:
  - domain-4
  - task-4-6
  - multi-instance
  - code-review
  - self-review
  - independent-review
  - confidence-reporting
  - attention-dilution
domain: "4 - Prompt Engineering & Structured Output"
task: "4.6"
---

# Multi-Instance Review Architectures

> **Task 4.6** — Design multi-instance review architectures that overcome self-review limitations, use independent review sessions, multi-pass strategies for large changesets, and confidence-based routing.

---

## The Self-Review Problem

When a model generates code and then reviews it in the same session, it retains the full reasoning context from generation. This creates a systematic bias:

- The model **remembers why** it made each decision during generation.
- When asked to review, it **echoes its own reasoning** rather than challenging it.
- Subtle issues that an independent reviewer would catch are overlooked because the model's reasoning context provides a "justification" for every choice it made.

This is not a hypothetical concern — it is a well-documented limitation. Self-review in the same session produces significantly fewer findings than independent review, particularly for:
- **Logic errors** rationalized during generation ("I chose this approach because...")
- **Missing edge cases** not considered during generation (the model does not know what it did not think about)
- **Architectural concerns** that require a fresh perspective on the code's role in the broader system

**Key exam principle:** Independent review instances (without the generator's reasoning context) are more effective at catching subtle issues than self-review instructions or extended thinking within the same session.

---

## Independent Review Instances

The solution is straightforward: use a **separate Claude instance** that sees only the generated code artifact, not the reasoning that produced it.

```
Generator Session:               Reviewer Session:
┌──────────────────────┐         ┌──────────────────────┐
│ "Generate auth module"│         │ "Review this code for│
│                      │         │  security issues"    │
│ [reasoning context]  │         │                      │
│ [tool calls]         │   code  │ [no prior context]   │
│ [design decisions]   │ ──────► │ [fresh perspective]  │
│ [generated code]     │ only    │ [independent review] │
└──────────────────────┘         └──────────────────────┘
```

The reviewer sees the code as if encountering it for the first time. Without the generator's reasoning polluting its analysis, it is more likely to question assumptions, notice missing error handling, and identify security issues.

---

## Multi-Pass Review for Large Changesets

Large PRs (e.g., 14 files, 2000+ lines changed) create **attention dilution** — when the model processes all changes at once, it may miss issues in later files because its attention is consumed by earlier ones. Additionally, single-pass review of large changesets can produce **contradictory findings** where advice for one file conflicts with advice for another.

The solution is a multi-pass architecture:

### Pass 1: Per-File Local Analysis

Each file is reviewed independently, focusing on file-local issues:
- Correctness within the file
- Error handling completeness
- Internal consistency
- Security issues in the file's code

### Pass 2: Cross-File Integration Analysis

A separate pass reviews the relationships between files:
- Interface compatibility (does the caller match the callee's signature?)
- Consistent error handling strategy across modules
- Data flow correctness across boundaries
- Shared state management

```
Pass 1 (parallel, per-file):
  File A → Review Instance 1 → Findings A
  File B → Review Instance 2 → Findings B
  File C → Review Instance 3 → Findings C
  ...

Pass 2 (integration):
  [All files + Findings A,B,C...] → Integration Instance → Cross-file findings
```

---

## Confidence Self-Reporting

Models can provide calibrated confidence scores alongside their findings. This enables automated routing:

- **High confidence findings** (e.g., SQL injection, null pointer dereference) → auto-comment on PR
- **Medium confidence findings** → queue for human reviewer triage
- **Low confidence findings** → log for analysis but do not surface

Confidence self-reporting works best when the prompt explicitly requests it and provides calibration guidance.

---

## Examples

### Example 1 — Independent Claude Instance Reviewing Generated Code

The generator and reviewer are separate API calls with no shared conversation history:

```python
import anthropic

client = anthropic.Anthropic()

# === Generator Session ===
# Claude generates code with full reasoning context
gen_response = client.messages.create(
    model="claude-sonnet-4-6-20250514",
    max_tokens=4096,
    messages=[{
        "role": "user",
        "content": """Generate a Python authentication module with:
- JWT token creation and validation
- Password hashing with bcrypt
- Rate limiting on login attempts
- Session management with Redis

Follow security best practices."""
    }]
)

generated_code = gen_response.content[0].text

# === Independent Review Session ===
# A completely separate API call — no prior context, no reasoning history.
# The reviewer sees ONLY the code, not the generator's design decisions.
review_response = client.messages.create(
    model="claude-sonnet-4-6-20250514",
    max_tokens=4096,
    messages=[{
        "role": "user",
        "content": f"""Review the following authentication module for security
vulnerabilities, correctness issues, and missing edge cases.

For each finding, report:
- Location (function name, approximate line)
- Severity (critical / high / medium / low)
- Issue description
- Suggested fix

Be thorough — this code handles authentication and must be production-ready.

```python
{generated_code}
```"""
    }]
)

print(review_response.content[0].text)
```

**Why this works better than self-review:**

```python
# LESS EFFECTIVE: Self-review in the same session
messages = [
    {"role": "user", "content": "Generate an auth module..."},
    {"role": "assistant", "content": generated_code},  # Claude's own output
    {"role": "user", "content": "Now review the code you just generated for issues."}
]

# Claude retains its reasoning from generation. When it sees the JWT validation
# function, it thinks "I chose HS256 because..." rather than questioning whether
# HS256 is appropriate. When it sees the rate limiter, it thinks "I designed this
# as a sliding window because..." rather than checking if the window logic is correct.

# RESULT: Fewer findings, especially for design-level issues
```

```python
# MORE EFFECTIVE: Independent review (separate session, code only)
messages = [
    {"role": "user", "content": f"Review this code for security issues:\n{generated_code}"}
]

# Claude encounters the code fresh. It asks "why HS256 instead of RS256?"
# without the generator's justification. It checks the rate limiter logic
# without knowing the intended design, so it evaluates what the code DOES
# rather than what it was MEANT to do.

# RESULT: More findings, especially design-level and assumption-challenging ones
```

---

### Example 2 — Splitting a 14-File PR into Per-File Passes + Integration Pass

A PR modifies 14 files across 3 packages. Reviewing all 14 at once causes attention dilution — issues in files 10-14 get less scrutiny than files 1-3.

```python
import anthropic

client = anthropic.Anthropic()

# The PR's changed files, grouped by package
changed_files = {
    "src/api/routes/users.ts": diff_users,
    "src/api/routes/orders.ts": diff_orders,
    "src/api/middleware/auth.ts": diff_auth_middleware,
    "src/api/middleware/rateLimit.ts": diff_rate_limit,
    "src/services/userService.ts": diff_user_service,
    "src/services/orderService.ts": diff_order_service,
    "src/services/paymentService.ts": diff_payment_service,
    "src/models/user.ts": diff_user_model,
    "src/models/order.ts": diff_order_model,
    "src/utils/validation.ts": diff_validation,
    "src/utils/errors.ts": diff_errors,
    "tests/api/users.test.ts": diff_users_test,
    "tests/services/orderService.test.ts": diff_order_test,
    "tests/services/paymentService.test.ts": diff_payment_test,
}

# === Pass 1: Per-file local analysis (can run in parallel) ===
per_file_findings = {}

for filepath, diff in changed_files.items():
    response = client.messages.create(
        model="claude-sonnet-4-6-20250514",
        max_tokens=2048,
        messages=[{
            "role": "user",
            "content": f"""Review the following file diff for issues LOCAL to this file.
Focus on: correctness, error handling, security, edge cases.
Do NOT comment on cross-file concerns (interface compatibility, etc.)
— that will be handled in a separate pass.

File: {filepath}
```diff
{diff}
```

Report findings as JSON array: [{{"line": N, "severity": "...", "message": "..."}}]
If no issues found, return an empty array."""
        }]
    )
    per_file_findings[filepath] = response.content[0].text

# === Pass 2: Cross-file integration analysis ===
# Provide ALL files plus per-file findings for integration review
all_diffs = "\n\n".join(
    f"=== {fp} ===\n{diff}" for fp, diff in changed_files.items()
)
all_findings = "\n\n".join(
    f"=== {fp} findings ===\n{findings}"
    for fp, findings in per_file_findings.items()
)

integration_response = client.messages.create(
    model="claude-sonnet-4-6-20250514",
    max_tokens=4096,
    messages=[{
        "role": "user",
        "content": f"""You are reviewing a 14-file PR for CROSS-FILE integration issues.
Per-file local issues have already been identified (provided below for context).

Your job is to find issues that span multiple files:
- Interface mismatches (caller passes wrong args, missing required fields)
- Inconsistent error handling across service boundaries
- Data flow issues (field renamed in model but not in service)
- Missing test coverage for new code paths
- Contradictions between files (e.g., validation rules differ between API and service layer)

Do NOT duplicate per-file findings — focus exclusively on cross-file concerns.

ALL DIFFS:
{all_diffs}

PER-FILE FINDINGS (already reported, do not duplicate):
{all_findings}"""
    }]
)
```

**Why multi-pass works:**
- **Pass 1** gives each file full attention — the model's context is focused on one file at a time, so it catches subtle per-file issues that would be missed in a 14-file single pass.
- **Pass 2** focuses exclusively on cross-file relationships, with per-file issues already handled — the model's attention is directed at interfaces and data flow rather than being split across all concern types.

---

### Example 3 — Confidence-Based Review Routing

The reviewer reports a confidence level with each finding, enabling automated routing:

```python
review_response = client.messages.create(
    model="claude-sonnet-4-6-20250514",
    max_tokens=4096,
    messages=[{
        "role": "user",
        "content": f"""Review this code for issues. For each finding, include a
confidence level indicating how certain you are that this is a genuine issue:

- "high": Definitively a bug, security vulnerability, or correctness issue.
  You can explain exactly how it fails and under what conditions.
- "medium": Likely an issue but depends on context you cannot fully verify
  (e.g., concurrency behavior, external service contracts, runtime config).
- "low": Possible concern but could be an intentional pattern, a matter of
  style, or dependent on requirements you do not have.

Return findings as JSON:
[
  {{
    "file": "string",
    "line": number,
    "severity": "critical|high|medium|low",
    "confidence": "high|medium|low",
    "message": "string",
    "reasoning": "string explaining why this confidence level"
  }}
]

Calibration guidance:
- SQL injection with user input → high confidence, critical severity
- Missing null check where caller MIGHT pass null → medium confidence
- Variable naming that COULD be clearer → low confidence
- Potential race condition in async code → medium confidence (depends on
  actual concurrency patterns in production)

```python
{code_to_review}
```"""
    }]
)

# Route findings based on confidence
import json
findings = json.loads(review_response.content[0].text)

auto_comment = [f for f in findings if f["confidence"] == "high"]
human_triage = [f for f in findings if f["confidence"] == "medium"]
logged_only  = [f for f in findings if f["confidence"] == "low"]

# High confidence → post directly as PR comments
for finding in auto_comment:
    post_pr_comment(finding["file"], finding["line"], finding["message"])

# Medium confidence → queue for human reviewer
for finding in human_triage:
    add_to_review_queue(finding)

# Low confidence → log for analysis, do not surface
for finding in logged_only:
    log_finding(finding)

print(f"Auto-commented: {len(auto_comment)}, "
      f"Queued for human: {len(human_triage)}, "
      f"Logged only: {len(logged_only)}")
```

**Why this matters:** Without confidence routing, all findings are treated equally. Low-confidence findings (style nits, possible-but-uncertain issues) create noise that buries critical findings. Confidence-based routing ensures high-severity, high-confidence findings get immediate attention while uncertain findings are appropriately triaged.

---

### Example 4 — Why Self-Review in the Same Session Fails

Consider this concrete scenario. Claude generates a caching module:

```python
# Claude's generated code (in the generation session)
class UserCache:
    def __init__(self, redis_client, ttl=3600):
        self.redis = redis_client
        self.ttl = ttl

    def get_user(self, user_id: str) -> dict | None:
        cached = self.redis.get(f"user:{user_id}")
        if cached:
            return json.loads(cached)
        return None

    def set_user(self, user_id: str, user_data: dict):
        self.redis.set(f"user:{user_id}", json.dumps(user_data), ex=self.ttl)

    def invalidate_user(self, user_id: str):
        self.redis.delete(f"user:{user_id}")
```

**Self-review in the same session** (less effective):

When asked "review the code you just wrote," the model's internal state includes:
- "I chose a simple key-value approach because the requirements did not mention complex queries"
- "I used `json.dumps/loads` for serialization because the data is a plain dict"
- "I set TTL on write because that is the standard Redis pattern"

The model confirms its own decisions: "The code correctly implements a simple Redis cache with TTL-based expiration. The key naming convention is consistent. The serialization approach is appropriate for dict data."

**Independent review** (more effective):

A separate instance, seeing only the code, would likely identify:

1. **No error handling for Redis connection failures.** `self.redis.get()` will throw `redis.ConnectionError` if Redis is down. The caller gets an unhandled exception instead of a cache miss.

2. **No protection against cache stampede.** When a popular user's cache expires, many concurrent requests will all miss the cache simultaneously and hit the database. A lock or probabilistic early expiration is missing.

3. **JSON serialization is fragile.** If `user_data` contains datetime objects or other non-JSON-serializable types, `json.dumps` will raise `TypeError` at write time, but the caller may not handle this.

4. **`invalidate_user` does not verify deletion.** If the key does not exist, `redis.delete` returns 0 but the method silently succeeds. Depending on the use case, the caller may need to know if there was actually cached data to invalidate.

The generator never considered these issues during generation, so its self-review has no reason to raise them — the absence of these patterns was an *implicit decision*, not a conscious tradeoff. An independent reviewer evaluates what the code *does* versus what production code *should do*, without being anchored by the generator's reasoning.

---

## Key Takeaways for the Exam

1. **Self-review in the same session is systematically weaker** than independent review — the model echoes its own reasoning rather than challenging it.
2. **Independent review instances** should see only the code artifact, not the generator's conversation history or reasoning.
3. **Multi-pass review** (per-file + integration) prevents attention dilution on large changesets and avoids contradictory findings.
4. **Confidence self-reporting** enables automated routing — high confidence findings auto-comment, medium findings go to human triage, low findings are logged only.
5. The key insight is about **reasoning context contamination** — the generator's reasoning creates blind spots that persist through self-review, extended thinking, and explicit "review your own work" instructions.

---

## Related Topics

- [[Domain 4 - Prompt Engineering & Structured Output]] — Parent domain overview
- [[CI-CD Integration]] — Multi-instance review in CI/CD pipelines
- [[Explicit Criteria for Precision]] — Providing review criteria to independent reviewer instances
- [[Few-Shot Prompting]] — Calibrating reviewer findings with few-shot examples
- [[Multi-Agent Orchestration]] — Multi-instance review as a form of orchestration
- [[Subagent Configuration and Context Passing]] — Context isolation between generator and reviewer agents
- [[Batch Processing]] — Batch API for running per-file review passes at scale
