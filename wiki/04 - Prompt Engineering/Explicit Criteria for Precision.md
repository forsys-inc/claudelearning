---
tags:
  - domain-4
  - task-4-1
  - prompt-engineering
  - precision
  - review-criteria
  - false-positives
domain: "4 - Prompt Engineering & Structured Output"
task: "4.1"
---

# Explicit Criteria for Precision

> **Task 4.1** — Write explicit, specific review criteria that reduce false positives, define severity levels with concrete examples, and build developer trust through precision.

---

## Why Explicit Criteria Matter

The single biggest failure mode in AI-assisted code review is **high false-positive rates**. When a review tool flags too many non-issues, developers learn to ignore it — and then miss genuine bugs when they appear. The solution is not "be more careful" (a vague instruction the model cannot act on) but rather **explicit behavioral criteria** that define exactly what to report and what to skip.

### The Core Principle

Explicit criteria define observable behaviors the model can check. Vague instructions ask the model to exercise judgment it cannot calibrate.

| Approach | Instruction | Result |
|----------|-------------|--------|
| **Vague** | "Check that comments are accurate" | Flags stylistic preferences, outdated but harmless comments, and genuine inaccuracies equally |
| **Explicit** | "Flag comments only when the claimed behavior contradicts the actual code behavior" | Focuses on semantic contradictions, skips style issues |
| **Vague** | "Be conservative in your review" | No measurable effect on false-positive rate |
| **Explicit** | "Only flag issues where the code will produce incorrect results, crash, or expose a security vulnerability at runtime" | Measurably reduces false positives to actionable findings |

---

## General Instructions Fail to Improve Precision

Instructions like "be conservative," "only flag important issues," or "use good judgment" do not meaningfully reduce false-positive rates. The model lacks a shared calibration for what "conservative" or "important" means in a given codebase. These instructions feel helpful to write but produce no measurable improvement.

What works instead:

- **Enumerated categories** — List exactly which issue types to report.
- **Concrete behavioral tests** — Define what makes something a finding vs. an acceptable pattern.
- **Severity criteria with code examples** — Show the model what each severity level looks like.

---

## What to Report vs. What to Skip

A well-designed review prompt explicitly partitions issue types:

**Report (high-value findings):**
- Bugs that will produce incorrect results at runtime
- Security vulnerabilities (SQL injection, XSS, path traversal, exposed secrets)
- Concurrency issues (race conditions, deadlocks)
- Resource leaks (unclosed connections, file handles)
- API contract violations that will cause integration failures

**Skip (low-value / high-noise):**
- Minor style deviations already handled by linters
- Variable naming preferences
- Comment formatting
- Import ordering
- Whitespace issues
- TODO/FIXME comments (these are intentional markers, not bugs)

---

## Temporarily Disabling High False-Positive Categories

When a particular category generates excessive false positives, the pragmatic response is to **disable it temporarily** while you refine the criteria, rather than letting it erode trust in the entire system.

Track false-positive rates per category. When a category exceeds a threshold (e.g., >60% false positives), disable it, refine the criteria using the false-positive examples, and re-enable with tighter definitions.

---

## Severity Criteria with Concrete Examples

Abstract severity labels ("high," "medium," "low") are useless without concrete definitions. Each level must include a behavioral description and a code example that unmistakably belongs at that level.

### Severity Framework

| Level | Definition | Behavioral Test |
|-------|-----------|-----------------|
| **Critical** | Will cause data loss, security breach, or system crash in production | Would you wake someone up at 2 AM for this? |
| **High** | Produces incorrect results under normal usage conditions | Would this fail a QA test case? |
| **Medium** | Causes problems under edge cases or specific configurations | Requires a non-obvious scenario to trigger |
| **Low** | Code quality issue that increases future maintenance burden | No runtime impact today, but makes the next change riskier |

---

## Examples

### Example 1 — Vague vs. Explicit Review Criteria Comparison

```xml
<!-- BAD: Vague criteria -->
<system>
You are a code reviewer. Review the following code for issues.
Be thorough but conservative. Only flag important problems.
Check that the code is correct and well-written.
</system>

<!-- GOOD: Explicit criteria -->
<system>
You are a code reviewer for a Python REST API codebase.

REPORT these issue types:
1. SQL injection: any string concatenation or f-string used in SQL queries
   instead of parameterized queries
2. Unvalidated input: request parameters used in file paths, shell commands,
   or database queries without validation
3. Resource leaks: database connections, file handles, or HTTP sessions opened
   without a context manager (with statement) or explicit close in a finally block
4. Null reference: accessing attributes on values that could be None without
   a guard check, especially after .get() calls or optional parameters
5. API contract violations: returning a response shape that differs from the
   OpenAPI spec (missing required fields, wrong types)

DO NOT REPORT:
- Style issues (naming conventions, import ordering, line length)
- Missing type hints (these are tracked separately)
- TODO/FIXME comments (these are intentional tracking markers)
- Suggestions that are "nice to have" but do not affect correctness or security

For each finding, specify:
- The exact line number(s)
- Which of the 5 categories above it falls into
- A one-sentence explanation of the runtime consequence
</system>
```

The explicit version gives the model five concrete categories with observable definitions. The vague version leaves "important" and "conservative" undefined, resulting in inconsistent flagging.

---

### Example 2 — Severity Levels with Code Examples for Each

```xml
<system>
Assign severity using these definitions and reference examples:

CRITICAL — Will cause data loss, security breach, or crash in production.
Example:
```python
# CRITICAL: SQL injection — user input directly interpolated into query
def get_user(username):
    query = f"SELECT * FROM users WHERE name = '{username}'"
    cursor.execute(query)  # Attacker sends: ' OR '1'='1
```

HIGH — Produces incorrect results under normal usage conditions.
Example:
```python
# HIGH: Off-by-one causes last item to be silently dropped
def process_batch(items):
    for i in range(len(items) - 1):  # Should be range(len(items))
        process(items[i])
```

MEDIUM — Causes problems under edge cases or specific configurations.
Example:
```python
# MEDIUM: Fails only when timezone differs from UTC
def is_today(timestamp):
    return timestamp.date() == datetime.now().date()
    # Should use datetime.now(tz=timezone.utc) if timestamps are UTC
```

LOW — Code quality issue increasing future maintenance burden.
Example:
```python
# LOW: Magic number makes threshold changes error-prone
def check_health(response_time):
    if response_time > 2.5:  # Should be a named constant
        alert("slow response")
```
</system>
```

Each severity level has a concrete code example that anchors the model's judgment. Without these examples, "HIGH" and "MEDIUM" blur together and the model's assignments become inconsistent across reviews.

---

### Example 3 — Disabling Unreliable Categories to Restore Trust

```python
# Configuration-driven category management for AI code review
REVIEW_CATEGORIES = {
    "sql_injection": {
        "enabled": True,
        "false_positive_rate": 0.12,  # 12% — acceptable
        "criteria": "Flag string concatenation or f-strings in SQL queries. "
                    "Do NOT flag ORM query builder methods (.filter(), .where()).",
    },
    "null_reference": {
        "enabled": True,
        "false_positive_rate": 0.08,
        "criteria": "Flag attribute access on values that may be None. "
                    "Do NOT flag when a preceding if-check guards the access.",
    },
    "error_handling": {
        "enabled": False,  # Disabled: 72% false positive rate
        "false_positive_rate": 0.72,
        "criteria": "TEMPORARILY DISABLED — criteria being refined. "
                    "Was flagging intentional bare excepts in CLI scripts "
                    "and test fixtures where they are acceptable.",
        "disabled_reason": "Flagging bare excepts in test fixtures and CLI "
                           "entry points where they are intentional patterns.",
    },
    "comment_accuracy": {
        "enabled": True,
        "false_positive_rate": 0.15,
        "criteria": "Flag comments ONLY when the described behavior directly "
                    "contradicts what the code actually does. "
                    "Do NOT flag: outdated parameter names in docstrings, "
                    "missing documentation for new parameters, "
                    "or stylistic phrasing preferences.",
    },
}

def build_review_prompt(categories: dict) -> str:
    """Build a review prompt using only enabled categories."""
    enabled = {k: v for k, v in categories.items() if v["enabled"]}

    criteria_block = "\n".join(
        f"- **{name}**: {cfg['criteria']}"
        for name, cfg in enabled.items()
    )

    return f"""Review the following code for these specific issue types ONLY:

{criteria_block}

Do NOT report issues outside these categories.
For each finding, state the category, line number, and runtime consequence."""
```

When the `error_handling` category produces 72% false positives, it is disabled while the criteria are refined. Developers see only high-precision categories, maintaining trust in the tool. The false-positive examples from the disabled category are used to write tighter criteria before re-enabling.

---

### Example 4 — Specific Criteria for Comment Accuracy Checking

```xml
<system>
Review code comments for accuracy. Apply these specific rules:

FLAG (report as a finding):
- Comment says a function "returns X" but the code returns Y
  Example: "# Returns the user's email" on a function that returns user ID
- Comment says "this loop iterates over X" but the loop iterates over Y
  Example: "# Process each order" on a loop that iterates over customers
- Comment describes a condition as "checks if X" but the condition checks Y
  Example: "# Verify user is admin" on a check for is_authenticated (not is_admin)

DO NOT FLAG (these are not accuracy issues):
- Comments that are correct but could be more detailed
- Docstrings missing newly added parameters (this is a docs task, not accuracy)
- Comments using slightly different terminology than the code
  (e.g., "remove" vs "delete" when the behavior is the same)
- Outdated formatting in docstrings (e.g., old-style :param: vs Google style)
- Comments on obvious code (this is a style preference, not an accuracy issue)

JUDGMENT CALLS — only flag if clearly wrong:
- Comments describing algorithm complexity (flag only if off by more than
  one complexity class, e.g., comment says O(n) but code is O(n^3))
- Comments about thread safety (flag only if the comment claims thread-safe
  but the code uses shared mutable state with no synchronization)
</system>
```

```python
# Example code for the model to review:

def calculate_discount(order_total, customer_tier):
    """Calculate shipping cost based on order weight.  # ← FLAG: says 'shipping cost'
                                                       #   but calculates discount
    Args:
        order_total: The total order amount
        customer_tier: Customer loyalty tier
    """
    # Apply 10% discount for premium customers    # ← DO NOT FLAG: correct description
    if customer_tier == "premium":
        return order_total * 0.10

    # Apply 5% discount for all orders             # ← FLAG: says 'all orders' but
    if customer_tier == "standard":                #   only applies to 'standard' tier
        return order_total * 0.05

    return 0  # No discount for unknown tiers      # ← DO NOT FLAG: accurate comment
```

The criteria separate clear contradictions (docstring says "shipping cost" for a discount function) from acceptable imprecision (slightly informal phrasing). This prevents the model from flagging every comment that could theoretically be improved.

---

## Key Takeaways for the Exam

1. **Explicit behavioral criteria** outperform vague instructions like "be conservative" — the model cannot calibrate abstract adjectives.
2. **Enumerate what to report and what to skip** — partitioning issue types is more effective than asking for "important" issues.
3. **Severity levels need code examples** — labels like "HIGH" and "MEDIUM" are meaningless without concrete anchoring.
4. **Disable high false-positive categories** rather than letting them erode developer trust in the entire review system.
5. **False-positive rate is the metric** — a review tool that is 95% precise on a narrow scope is more valuable than one that is 60% precise on a broad scope.

---

## Related Topics

- [[Domain 4 - Prompt Engineering & Structured Output]] — Parent domain overview
- [[Few-Shot Prompting]] — Use examples to further calibrate review behavior
- [[Validation and Retry Loops]] — Validate review output and handle false-positive feedback
- [[Multi-Instance Review Architectures]] — Use independent instances to cross-check reviews
- [[Hooks and Interception]] — Enforce review criteria through deterministic hooks
