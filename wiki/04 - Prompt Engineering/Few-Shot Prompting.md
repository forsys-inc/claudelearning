---
tags:
  - domain-4
  - task-4-2
  - prompt-engineering
  - few-shot
  - examples
  - output-format
  - hallucination-reduction
domain: "4 - Prompt Engineering & Structured Output"
task: "4.2"
---

# Few-Shot Prompting

> **Task 4.2** — Use few-shot examples as the most effective technique for consistently formatted output, ambiguous-case handling, generalization to novel patterns, and hallucination reduction.

---

## Why Few-Shot Examples Are the Most Effective Technique

Instruction-based prompting tells the model what to do. Few-shot prompting **shows** the model what to do. For tasks requiring consistently formatted output, few-shot examples are the single most effective technique because they:

1. **Establish output format implicitly** — The model mirrors the structure of the examples without needing exhaustive format specifications.
2. **Demonstrate ambiguous-case handling** — Show the model how to reason through edge cases where the "right" answer is not obvious.
3. **Enable generalization to novel patterns** — 2-4 well-chosen examples covering different scenarios let the model interpolate for situations not explicitly shown.
4. **Reduce hallucination in extraction tasks** — When examples show the model extracting only what is present (and using null/empty for absent data), the model learns to avoid fabricating information.

### How Many Examples?

- **2-4 examples** is the sweet spot for most tasks. Fewer than 2 may not establish the pattern; more than 4 rarely improves quality and consumes context window.
- **Target ambiguous cases** — Do not waste examples on trivially obvious cases. Show the model how to handle the hard ones.
- **Include reasoning** — When examples show the model's reasoning process (not just input/output), the model produces better reasoning on novel cases.

---

## Demonstrating Ambiguous-Case Handling

The highest-value few-shot examples are those that cover **ambiguous scenarios** where a naive model would guess wrong. These examples teach the model the decision boundary for your specific task.

For example, in a code review context, an ambiguous case might be a pattern that *looks* like a bug but is actually an intentional design choice (e.g., a bare `except` in a CLI entry point).

---

## Reducing Hallucination in Extraction Tasks

In structured data extraction, the model may fabricate plausible values for fields not present in the source. Few-shot examples that explicitly show `null` or `"not_specified"` for absent fields teach the model that **missing data should remain missing**.

---

## Examples

### Example 1 — Few-Shot for Tool Selection in Ambiguous Requests

```xml
<system>
You are a customer support agent with access to these tools:
- refund_order: Process a refund for a specific order
- check_order_status: Look up current status of an order
- escalate_to_human: Transfer to a human agent

Select the appropriate tool based on the customer's request.
Here are examples showing how to handle ambiguous cases:

EXAMPLE 1:
Customer: "I want my money back for order #1234"
Reasoning: Customer explicitly requests money back → this is a refund request.
Tool: refund_order(order_id="1234")

EXAMPLE 2:
Customer: "Where is my order #5678? I've been waiting two weeks."
Reasoning: Customer asks about location/timing → this is a status check.
The frustration does not make it a refund request — they want to know where it is.
Tool: check_order_status(order_id="5678")

EXAMPLE 3 (AMBIGUOUS):
Customer: "Order #9012 still hasn't arrived, this is unacceptable, I want this resolved."
Reasoning: Customer is frustrated and says "resolved" but has NOT explicitly asked for
a refund. "Resolved" could mean they want status, re-shipment, or a refund. When the
desired resolution is unclear, check status first to provide information before assuming
a refund is wanted.
Tool: check_order_status(order_id="9012")

EXAMPLE 4 (AMBIGUOUS):
Customer: "I'm having issues with my account and nothing is working."
Reasoning: No specific order mentioned, problem is vague, no tool can address
an unspecified account issue. This requires human judgment.
Tool: escalate_to_human(reason="Unspecified account issues, customer frustrated")

Now handle the following customer request:
</system>
```

Examples 3 and 4 are the high-value ones — they show the model how to handle ambiguity (frustration without explicit refund request, vague issues without specific orders). Examples 1 and 2 establish baseline behavior but the ambiguous cases are what prevent misfires in production.

---

### Example 2 — Few-Shot Showing Desired Output Format

```xml
<system>
You are a code review assistant. For each issue found, output in this exact format.
Here are examples showing the format and severity calibration:

EXAMPLE INPUT:
```python
def get_user(user_id):
    query = f"SELECT * FROM users WHERE id = {user_id}"
    return db.execute(query)
```

EXAMPLE OUTPUT:
- location: get_user(), line 2
- issue: SQL injection via f-string interpolation
- severity: CRITICAL
- fix: Use parameterized query: `db.execute("SELECT * FROM users WHERE id = ?", (user_id,))`

EXAMPLE INPUT:
```python
def fetch_data(url):
    response = requests.get(url, timeout=30)
    data = response.json()
    return data["results"]
```

EXAMPLE OUTPUT:
- location: fetch_data(), line 3
- issue: No check for HTTP error status before parsing JSON; non-200 responses will
  raise an unhandled exception or return unexpected data
- severity: HIGH
- fix: Add `response.raise_for_status()` before `.json()` call

EXAMPLE INPUT:
```python
def calculate_average(numbers):
    total = sum(numbers)
    return total / len(numbers)
```

EXAMPLE OUTPUT:
- location: calculate_average(), line 3
- issue: Division by zero when empty list is passed
- severity: MEDIUM
- fix: Add guard: `if not numbers: return 0` or raise ValueError for empty input

Now review the following code using this exact format:
</system>
```

The format (location, issue, severity, fix) is demonstrated through examples rather than described in prose. The model mirrors this structure far more reliably than when given a format specification alone.

---

### Example 3 — Examples Distinguishing Acceptable Patterns from Genuine Issues

```xml
<system>
Review Python code for exception handling issues. Use these examples to calibrate
what is a genuine issue vs. an acceptable pattern:

GENUINE ISSUE — Flag this:
```python
def process_payment(order):
    try:
        charge = stripe.Charge.create(amount=order.total, currency="usd")
        order.payment_id = charge.id
        order.save()
    except Exception:
        pass  # ← Silent swallow of payment errors — customer charged but order not updated
```
Why: Payment processing errors silently discarded; customer may be charged without
the order being marked as paid. This causes real financial discrepancies.

ACCEPTABLE PATTERN — Do NOT flag:
```python
def main():
    try:
        app.run()
    except KeyboardInterrupt:
        pass  # ← Intentional: suppress traceback on Ctrl+C in CLI tool
```
Why: Bare except for KeyboardInterrupt at the top-level entry point of a CLI
application is a standard pattern to provide clean exit behavior.

ACCEPTABLE PATTERN — Do NOT flag:
```python
def try_parse_date(text):
    for fmt in ["%Y-%m-%d", "%m/%d/%Y", "%d %b %Y"]:
        try:
            return datetime.strptime(text, fmt)
        except ValueError:
            continue  # ← Intentional: trying multiple formats
    return None
```
Why: Catching ValueError in a format-guessing loop is the standard approach.
The function returns None on failure, which is documented behavior.

GENUINE ISSUE — Flag this:
```python
def sync_inventory(items):
    for item in items:
        try:
            update_stock(item)
        except Exception:
            continue  # ← Silently skips failed items with no logging
```
Why: Failed inventory updates are silently skipped. The caller has no way to know
which items failed, leading to inventory drift.

Apply this calibration to the code below. Only flag patterns matching the
"GENUINE ISSUE" examples — not the "ACCEPTABLE PATTERN" ones.
</system>
```

These examples teach the decision boundary. Without them, the model would likely flag the `KeyboardInterrupt` pattern and the format-guessing loop, both of which are intentional and acceptable.

---

### Example 4 — Extraction Examples Handling Varied Document Structures

```xml
<system>
Extract metadata from technical documents. Documents vary in structure — some have
explicit fields, others bury information in prose.

EXAMPLE 1 — Structured document:
Document: """
Title: API Rate Limiting Implementation
Author: Sarah Chen
Date: 2024-03-15
Status: Approved
Summary: Implements token bucket rate limiting for all public API endpoints.
"""
Extraction:
{
  "title": "API Rate Limiting Implementation",
  "author": "Sarah Chen",
  "date": "2024-03-15",
  "status": "approved",
  "summary": "Implements token bucket rate limiting for all public API endpoints."
}

EXAMPLE 2 — Information buried in prose:
Document: """
Meeting Notes — March 20

We discussed Sarah's rate limiting proposal. John suggested we also consider
the caching layer changes before the Q2 release. No decision was made on timeline.
"""
Extraction:
{
  "title": "Meeting Notes — March 20",
  "author": null,
  "date": "March 20",
  "status": null,
  "summary": "Discussion of rate limiting proposal and caching layer changes; no timeline decision made."
}
Note: author is null because meeting notes do not have a single author.
Status is null because no decision status applies.
Date lacks a year — extract what is present, do not fabricate "2024-03-20".

EXAMPLE 3 — Minimal information:
Document: """
TODO: revisit authentication flow after sprint 12
"""
Extraction:
{
  "title": null,
  "author": null,
  "date": null,
  "status": null,
  "summary": "Reminder to revisit authentication flow after sprint 12."
}
Note: Do not fabricate a title. A TODO item does not have title, author, date,
or status fields — all are null.

EXAMPLE 4 — Ambiguous fields:
Document: """
RFC: Migrate to PostgreSQL 16
Prepared by the Infrastructure Team
Last updated February 2024
Draft — pending review

This RFC proposes migrating from PostgreSQL 14 to 16 for improved query
performance and logical replication enhancements.
"""
Extraction:
{
  "title": "RFC: Migrate to PostgreSQL 16",
  "author": "Infrastructure Team",
  "date": "February 2024",
  "status": "draft",
  "summary": "Proposal to migrate from PostgreSQL 14 to 16 for query performance and replication improvements."
}
Note: "Prepared by the Infrastructure Team" maps to author even though it is a team,
not a person. "Draft — pending review" maps to status as "draft".

Now extract metadata from the following document:
</system>
```

Example 2 is critical — it shows `null` for absent fields and partial dates without fabrication. Example 3 shows the model that minimal documents should produce mostly-null output rather than hallucinated values. Without these examples, the model will try to fill in every field.

---

## Key Takeaways for the Exam

1. **Few-shot examples are the most effective technique** for consistently formatted output — more reliable than detailed format specifications alone.
2. **Target ambiguous cases** — Obvious examples are wasted tokens. Show the model how to handle edge cases where the answer is not clear.
3. **2-4 examples is the sweet spot** — Enough to establish patterns, not so many that they consume the context window.
4. **Include reasoning in examples** — When examples show *why* a decision was made, the model produces better reasoning on novel inputs.
5. **Show null/missing values explicitly** — This is the most effective way to reduce hallucination in extraction tasks.

---

## Related Topics

- [[Domain 4 - Prompt Engineering & Structured Output]] — Parent domain overview
- [[Explicit Criteria for Precision]] — Criteria and few-shot examples work together for calibrated review
- [[JSON Schemas and Tool Use]] — Combine few-shot examples with schema enforcement for maximum reliability
- [[Validation and Retry Loops]] — Few-shot examples in retry prompts can correct recurring format errors
- [[Context Management Strategies]] — Token budget considerations when including few-shot examples
