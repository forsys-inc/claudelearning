---
tags:
  - domain-1
  - task-1-5
  - hooks
  - interception
  - PostToolUse
  - compliance
  - data-normalization
domain: "1 - Agentic Architecture & Orchestration"
task: "1.5"
---

# Hooks and Interception

> **Task 1.5** — Use PostToolUse hooks for data normalization, tool call interception for compliance enforcement, and understand when hooks beat prompt instructions.

---

## What Are Hooks?

Hooks are code-level interception points that run before or after tool execution within the agentic loop. They sit between the model's tool call request and the actual tool execution (pre-hooks) or between tool execution and returning the result to the model (post-hooks).

```
Model requests tool_use
        │
        ▼
┌─────────────────┐
│  Pre-Tool Hook  │──► Can BLOCK the call, modify inputs, or redirect
└────────┬────────┘
         ▼
┌─────────────────┐
│  Tool Executes  │
└────────┬────────┘
         ▼
┌─────────────────┐
│  Post-Tool Hook │──► Can TRANSFORM the output, add metadata, normalize data
└────────┬────────┘
         ▼
Result returned to model
```

### Hook Types

| Hook | Timing | Common Uses |
|------|--------|-------------|
| **PreToolUse** | Before tool executes | Validation, authorization, blocking, input sanitization |
| **PostToolUse** | After tool executes | Data normalization, output enrichment, logging, format standardization |

---

## Hooks for Deterministic Guarantees

This is the key exam concept: **hooks provide deterministic guarantees; prompt instructions provide probabilistic compliance.**

- A prompt instruction saying "always convert timestamps to ISO 8601" will work 95-99% of the time. The other 1-5% of the time, the model may forget, especially in long conversations or under complex reasoning loads.
- A PostToolUse hook that converts timestamps will work **100% of the time** because it is code, not a suggestion.

Use hooks when:
- The consequence of failure is a bug, security issue, or compliance violation.
- The transformation is mechanical and does not require judgment.
- Consistency must be absolute, not "usually correct."

Use prompt instructions when:
- The behaviour requires reasoning or judgment (e.g., "choose the most relevant result").
- Flexibility is desired (e.g., "summarize in a friendly tone").
- Failure has low impact (e.g., formatting preferences).

---

## Examples

### Example 1 — PostToolUse Hook Normalizing Unix Timestamps to ISO 8601

```python
import datetime
from typing import Any


def post_tool_hook(tool_name: str, tool_input: dict, tool_result: Any) -> Any:
    """PostToolUse hook that normalizes all Unix timestamps to ISO 8601.

    Runs after every tool execution. Scans the result for numeric values
    in known timestamp fields and converts them to human-readable ISO format.
    """
    TIMESTAMP_FIELDS = {
        "created_at", "updated_at", "timestamp", "date",
        "last_login", "expires_at", "verified_at", "closed_at",
    }

    if isinstance(tool_result, dict):
        return normalize_timestamps(tool_result, TIMESTAMP_FIELDS)
    elif isinstance(tool_result, list):
        return [normalize_timestamps(item, TIMESTAMP_FIELDS)
                for item in tool_result if isinstance(item, dict)]
    return tool_result


def normalize_timestamps(data: dict, fields: set[str]) -> dict:
    """Recursively convert Unix timestamp fields to ISO 8601."""
    normalized = {}
    for key, value in data.items():
        if key in fields and isinstance(value, (int, float)):
            # Convert Unix timestamp to ISO 8601
            normalized[key] = datetime.datetime.fromtimestamp(
                value, tz=datetime.timezone.utc
            ).isoformat()
        elif isinstance(value, dict):
            normalized[key] = normalize_timestamps(value, fields)
        elif isinstance(value, list):
            normalized[key] = [
                normalize_timestamps(v, fields) if isinstance(v, dict) else v
                for v in value
            ]
        else:
            normalized[key] = value
    return normalized


# Before hook:
# {"customer_id": "C-9382", "created_at": 1710489600, "last_login": 1710576000}

# After hook:
# {"customer_id": "C-9382", "created_at": "2024-03-15T08:00:00+00:00",
#  "last_login": "2024-03-16T08:00:00+00:00"}
```

The model never sees raw Unix timestamps. Every tool result is automatically normalized before it enters the conversation history. This eliminates the class of bugs where the model tries to interpret or display Unix timestamps inconsistently.

---

### Example 2 — Hook Blocking Refunds Over $500 and Redirecting to Escalation

```python
class ComplianceHook:
    """Pre-tool hook enforcing refund limits and mandatory escalation."""

    REFUND_LIMIT = 500.00

    def pre_tool_use(self, tool_name: str, tool_input: dict) -> dict | None:
        """Intercept tool calls and block or redirect as needed.

        Returns None to allow the call, or a dict to replace the tool result
        (blocking the actual execution).
        """
        if tool_name == "process_refund":
            return self._check_refund_compliance(tool_input)

        if tool_name == "delete_customer_data":
            return self._require_explicit_confirmation(tool_input)

        return None  # Allow all other tools

    def _check_refund_compliance(self, tool_input: dict) -> dict | None:
        amount = tool_input.get("amount", 0)

        if amount > self.REFUND_LIMIT:
            # BLOCK the refund — return an error result to the model
            return {
                "blocked": True,
                "reason": f"Refund amount ${amount:.2f} exceeds automated "
                         f"limit of ${self.REFUND_LIMIT:.2f}.",
                "required_action": "escalate_to_human",
                "escalation_context": {
                    "type": "high_value_refund",
                    "amount": amount,
                    "order_id": tool_input.get("order_id"),
                    "customer_id": tool_input.get("customer_id"),
                },
                "instruction": "Call escalate_to_human with the provided "
                              "escalation_context. Do NOT attempt to split "
                              "the refund into smaller amounts.",
            }

        return None  # Amount within limit — allow execution

    def _require_explicit_confirmation(self, tool_input: dict) -> dict | None:
        if not tool_input.get("customer_confirmed_deletion"):
            return {
                "blocked": True,
                "reason": "Customer must explicitly confirm data deletion.",
                "required_action": "Ask the customer to confirm they want "
                                  "their data permanently deleted.",
            }
        return None


# Integration into the agentic loop:
compliance = ComplianceHook()

def execute_tool_with_hooks(tool_name: str, tool_input: dict) -> str:
    # Pre-hook: check compliance
    blocked_result = compliance.pre_tool_use(tool_name, tool_input)
    if blocked_result is not None:
        return json.dumps(blocked_result)  # Return block message to model

    # Execute the actual tool
    result = TOOL_IMPLEMENTATIONS[tool_name](tool_input)

    # Post-hook: normalize data
    result = post_tool_hook(tool_name, tool_input, result)

    return json.dumps(result)
```

When the model tries to process a $750 refund, the hook blocks execution and returns a structured error telling the model to escalate instead. The model cannot bypass this — the refund tool never executes.

---

### Example 3 — When to Use Hooks vs. Prompts: Decision Matrix

```yaml
# Decision matrix for hooks vs. prompt instructions

hooks_required:
  description: "Use hooks (deterministic) when failure has serious consequences"
  examples:
    - scenario: "Refund amount validation"
      why: "Financial loss if exceeded"
      hook_type: "PreToolUse"
      action: "Block and redirect to escalation"

    - scenario: "Timestamp format normalization"
      why: "Downstream systems require ISO 8601"
      hook_type: "PostToolUse"
      action: "Convert all timestamp fields automatically"

    - scenario: "PII redaction from logs"
      why: "Compliance violation if PII is logged"
      hook_type: "PostToolUse"
      action: "Strip SSN, credit card numbers before logging"

    - scenario: "Identity verification before account changes"
      why: "Security breach if bypassed"
      hook_type: "PreToolUse"
      action: "Block tool if verification step not completed"

prompts_sufficient:
  description: "Use prompt instructions (probabilistic) when failure is tolerable"
  examples:
    - scenario: "Response tone and style"
      why: "Slightly wrong tone is not a security issue"
      prompt: "Respond in a warm, professional tone"

    - scenario: "Result summarization length"
      why: "A longer/shorter summary is an inconvenience, not a bug"
      prompt: "Keep summaries under 3 sentences"

    - scenario: "Tool selection preference"
      why: "Using a less optimal tool path is slower but not harmful"
      prompt: "Prefer get_customer_by_id over search_customers when ID is known"

    - scenario: "Conversation flow suggestions"
      why: "Skipping a courtesy step is suboptimal but not dangerous"
      prompt: "Summarize your understanding before taking action"

combined_approach:
  description: "Use both together for defense in depth"
  example:
    scenario: "Refund processing"
    prompt: "Always verify customer identity before processing refunds"
    hook: "PreToolUse gate blocking process_refund without verification"
    why: "Prompt guides the model to do it right; hook catches the rare failure"
```

---

### Example 4 — Data Normalization Hook Handling Heterogeneous Formats

```python
def normalize_tool_output(tool_name: str, raw_result: dict) -> dict:
    """PostToolUse hook that standardizes heterogeneous data formats.

    Different backend APIs return data in different formats. This hook
    normalizes everything before it reaches the model, so the model
    always sees consistent structure.
    """

    # Normalize currency: different APIs return different formats
    if "amount" in raw_result:
        raw_amount = raw_result["amount"]
        if isinstance(raw_amount, str):
            # "$1,234.56" or "1234.56 USD" → float
            cleaned = raw_amount.replace("$", "").replace(",", "").split()[0]
            raw_result["amount"] = float(cleaned)
            raw_result["currency"] = "USD"
        elif isinstance(raw_amount, dict):
            # {"value": 1234.56, "currency": "USD"} → flatten
            raw_result["amount"] = raw_amount["value"]
            raw_result["currency"] = raw_amount.get("currency", "USD")

    # Normalize status fields: map backend codes to human-readable strings
    STATUS_MAP = {
        0: "pending", 1: "active", 2: "suspended",
        3: "cancelled", -1: "error",
        "A": "active", "S": "suspended", "P": "pending",
    }
    if "status" in raw_result and raw_result["status"] in STATUS_MAP:
        raw_result["status_code"] = raw_result["status"]  # Preserve original
        raw_result["status"] = STATUS_MAP[raw_result["status"]]

    # Normalize phone numbers to E.164 format
    if "phone" in raw_result and raw_result["phone"]:
        phone = raw_result["phone"]
        digits = "".join(c for c in phone if c.isdigit())
        if len(digits) == 10:  # US number without country code
            raw_result["phone"] = f"+1{digits}"
        elif len(digits) == 11 and digits[0] == "1":
            raw_result["phone"] = f"+{digits}"

    # Normalize boolean-like fields
    BOOL_TRUTHY = {"true", "yes", "1", "Y", "active", "enabled"}
    BOOL_FALSY = {"false", "no", "0", "N", "inactive", "disabled"}
    for key in ["is_active", "auto_renew", "email_verified"]:
        if key in raw_result and isinstance(raw_result[key], str):
            val = raw_result[key].lower().strip()
            if val in BOOL_TRUTHY:
                raw_result[key] = True
            elif val in BOOL_FALSY:
                raw_result[key] = False

    return raw_result


# Example transformation:
# Input from legacy billing API:
# {
#     "customer": "C-9382",
#     "amount": "$1,234.56",
#     "status": "A",
#     "auto_renew": "Y",
#     "created_at": 1710489600,
#     "phone": "(555) 123-4567"
# }

# After normalization hooks (this hook + timestamp hook):
# {
#     "customer": "C-9382",
#     "amount": 1234.56,
#     "currency": "USD",
#     "status": "active",
#     "status_code": "A",
#     "auto_renew": true,
#     "created_at": "2024-03-15T08:00:00+00:00",
#     "phone": "+15551234567"
# }
```

The model always sees clean, consistent data regardless of which backend API the tool called. This eliminates an entire category of errors where the model misinterprets `"A"` as a string value rather than an active status code, or tries to do math on `"$1,234.56"`.

---

## Key Takeaways for the Exam

1. **PostToolUse hooks** transform tool output before the model sees it — ideal for normalization.
2. **PreToolUse hooks** intercept tool calls before execution — ideal for compliance and authorization.
3. **Hooks = deterministic; prompts = probabilistic.** Use hooks for anything where failure is unacceptable.
4. **Defense in depth**: use prompt instructions to guide the model's reasoning AND hooks to catch the cases where instructions fail.
5. Hooks handle **mechanical transformations** (timestamps, currencies, formats); prompts handle **judgment calls** (tone, strategy, prioritization).

---

## Related Topics

- [[Domain 1 - Agentic Architecture & Orchestration]] — Parent domain overview
- [[Multi-Step Workflows and Handoffs]] — Prerequisite gates use the same hook mechanism
- [[Agentic Loop Lifecycle]] — Hooks execute within the tool execution step of the loop
- [[Tool Design Principles]] — Designing tools whose outputs are hook-friendly
- [[Claude Code Configuration]] — Configuring hooks in `.claude/settings.json`
