---
tags:
  - domain-2
  - task-2-2
  - error-handling
  - mcp
  - tool-design
domain: "Tool Design & MCP Integration"
task: "2.2"
---

# Structured Error Responses (Task 2.2)

> **Core principle:** When a tool fails, the *shape* of the error determines whether the agent can recover, retry, escalate, or explain. A uniform `"Operation failed"` message kills all four options.

---

## The MCP `isError` Flag

In the Model Context Protocol, tool results include an `isError` boolean. When `isError: true`, the model knows the content field describes a failure, not a successful result. This is the first layer of structure — distinguishing "this worked" from "this did not work."

```json
{
  "type": "tool_result",
  "tool_use_id": "toolu_abc123",
  "isError": true,
  "content": "Database connection timed out after 5000ms"
}
```

Without `isError`, Claude may try to interpret an error message as valid data, leading to hallucinated conclusions like *"The customer's name is 'Connection refused.'"*

---

## Error Categories

Not all failures are the same. A well-designed tool system distinguishes at least four categories:

| Category | Description | Retryable? | Example |
|----------|-------------|------------|---------|
| **Transient** | Temporary infrastructure issue | Yes | Timeout, rate limit, network blip |
| **Validation** | Bad input from the caller | No (fix input) | Invalid date format, missing required field |
| **Business** | Valid request, but business rules prevent it | No | Insufficient balance, account frozen |
| **Permission** | Caller lacks access | No (escalate) | Invalid API key, unauthorized role |

### Why This Matters for Agents

- **Transient** errors: the agent should retry (with backoff) before giving up.
- **Validation** errors: the agent should fix the input and retry — not retry the same bad request.
- **Business** errors: the agent should explain the constraint to the user — retrying will always fail.
- **Permission** errors: the agent should escalate or inform the user — it cannot fix this itself.

---

## The Problem with Uniform Errors

```json
{
  "isError": true,
  "content": "Operation failed"
}
```

This tells the agent nothing. It cannot distinguish:
- A transient timeout (retry in 2 seconds)
- A validation failure (fix the date format)
- A business rule violation (the account is frozen)
- A permission denial (the API key is wrong)

The agent's only option is to tell the user "something went wrong" — the worst possible experience.

---

## Structured Error Metadata

A structured error response gives the agent everything it needs to act:

```json
{
  "isError": true,
  "content": {
    "errorCategory": "transient | validation | business | permission",
    "errorCode": "MACHINE_READABLE_CODE",
    "isRetryable": true,
    "retryAfterMs": 2000,
    "message": "Human-readable description of what went wrong",
    "details": {}
  }
}
```

Key fields:
- **`errorCategory`** — tells the agent *what class* of problem this is.
- **`isRetryable`** — a direct boolean signal: should the agent try again?
- **`retryAfterMs`** — if retryable, how long to wait.
- **`message`** — a human-readable string the agent can relay to the user.
- **`details`** — additional context (e.g., which field failed validation, what the limit was).

---

## Local Error Recovery in Subagents

In a multi-agent architecture ([[Multi-Agent Orchestration]]), subagents should handle transient errors locally rather than propagating every failure to the orchestrator.

**Pattern:** A subagent retries transient failures up to N times. Only if all retries are exhausted — or the error is non-transient — does it propagate the error upward.

This prevents the orchestrator from being flooded with transient noise that would have resolved itself.

---

## Access Failures vs Valid Empty Results

A critical distinction that uniform errors destroy:

| Scenario | Tool Response | Meaning |
|----------|---------------|---------|
| User has no orders | `{ "orders": [], "isError": false }` | Success — zero results is valid data |
| Database is unreachable | `{ "isError": true, "errorCategory": "transient" }` | Failure — we don't know if there are orders |
| API key is invalid | `{ "isError": true, "errorCategory": "permission" }` | Failure — we are not authorized to check |

If all three return `"Operation failed"`, the agent might tell the user "You have no orders" when in fact the database was just down.

---

## Examples

### Example 1: Structured Error Response with Category and Retryability

```json
{
  "type": "tool_result",
  "tool_use_id": "toolu_rate_limit",
  "isError": true,
  "content": {
    "errorCategory": "transient",
    "errorCode": "RATE_LIMIT_EXCEEDED",
    "isRetryable": true,
    "retryAfterMs": 3000,
    "message": "API rate limit exceeded. 429 response from payments service. Current limit: 100 requests/minute.",
    "details": {
      "service": "payments-api",
      "currentRate": 112,
      "limit": 100,
      "windowResetAt": "2026-04-22T14:35:00Z"
    }
  }
}
```

**Agent behavior:** Wait 3 seconds, then retry the same call. If it fails again, wait longer (exponential backoff). After 3 retries, propagate to the orchestrator.

---

### Example 2: Business Rule Violation — Non-Retryable

```json
{
  "type": "tool_result",
  "tool_use_id": "toolu_transfer",
  "isError": true,
  "content": {
    "errorCategory": "business",
    "errorCode": "INSUFFICIENT_BALANCE",
    "isRetryable": false,
    "message": "Transfer of $500.00 cannot be completed. Account balance is $123.45. The account holder needs to deposit at least $376.55 before this transfer can proceed.",
    "details": {
      "requestedAmount": 500.00,
      "availableBalance": 123.45,
      "shortfall": 376.55,
      "currency": "USD",
      "accountId": "acct_7829"
    }
  }
}
```

**Agent behavior:** Do NOT retry — the same request will always fail. Instead, relay the customer-friendly message to the user: *"The transfer cannot be completed because your balance is $123.45. You would need to deposit at least $376.55 first."* The `details` object gives the agent concrete numbers to work with.

---

### Example 3: Subagent Handling a Transient Failure Locally

Consider an orchestrator that delegates data gathering to a `research_subagent`:

```python
# Inside research_subagent
async def gather_financial_data(company_id: str) -> dict:
    max_retries = 3
    for attempt in range(max_retries):
        result = await call_tool("fetch_financials", {"company_id": company_id})

        if not result.get("isError"):
            return result  # Success — return data to orchestrator

        error = result["content"]

        # Non-retryable errors propagate immediately
        if not error.get("isRetryable", False):
            return {
                "isError": True,
                "content": {
                    "errorCategory": error["errorCategory"],
                    "message": error["message"],
                    "subagent": "research_subagent",
                    "attemptsExhausted": False
                }
            }

        # Transient error — retry locally
        wait_ms = error.get("retryAfterMs", 1000 * (2 ** attempt))
        await asyncio.sleep(wait_ms / 1000)

    # All retries exhausted — now propagate
    return {
        "isError": True,
        "content": {
            "errorCategory": "transient",
            "message": f"Failed to fetch financials after {max_retries} attempts. Last error: {error['message']}",
            "subagent": "research_subagent",
            "attemptsExhausted": True
        }
    }
```

**Key pattern:**
- Transient errors are retried locally (the orchestrator never sees them if retry succeeds).
- Non-retryable errors (validation, business, permission) propagate immediately — no point retrying.
- If all retries are exhausted, the subagent propagates with `attemptsExhausted: true` so the orchestrator knows retrying further is unlikely to help.

---

### Example 4: Access Failure vs Valid Empty Result

**Scenario:** A tool checks whether a customer has any active subscriptions.

**Valid empty result — customer has no subscriptions:**

```json
{
  "type": "tool_result",
  "tool_use_id": "toolu_subs_check",
  "isError": false,
  "content": {
    "subscriptions": [],
    "totalActive": 0,
    "customerId": "cust_4412",
    "checkedAt": "2026-04-22T10:00:00Z"
  }
}
```

**Agent response:** *"You don't have any active subscriptions. Would you like to see our available plans?"*

**Access failure — could not check subscriptions:**

```json
{
  "type": "tool_result",
  "tool_use_id": "toolu_subs_check",
  "isError": true,
  "content": {
    "errorCategory": "permission",
    "errorCode": "SCOPE_INSUFFICIENT",
    "isRetryable": false,
    "message": "API token does not have the 'subscriptions:read' scope. The current token has scopes: ['profile:read', 'orders:read'].",
    "details": {
      "requiredScope": "subscriptions:read",
      "currentScopes": ["profile:read", "orders:read"]
    }
  }
}
```

**Agent response:** *"I'm unable to check your subscriptions right now due to an access configuration issue. Let me connect you with support."*

**The critical difference:** With uniform errors, both scenarios return `"Operation failed"` and the agent might tell the user "You have no subscriptions" when it actually has no idea — it just could not authenticate.

---

## Summary Checklist

- [ ] All tools return `isError: true` for failures (MCP pattern)
- [ ] Errors include `errorCategory` (transient, validation, business, permission)
- [ ] Errors include `isRetryable` boolean
- [ ] Retryable errors include `retryAfterMs` hint
- [ ] Error messages are human-readable and include concrete values
- [ ] Subagents retry transient errors locally before propagating
- [ ] Empty results (`[]`) are returned as successes, not errors
- [ ] Access failures are clearly distinguished from "no data found"

---

## Related Topics

- [[Domain 2 - Tool Design & MCP Integration]] — domain overview
- [[Tool Interface Design]] — how descriptions prevent tool misuse
- [[Tool Distribution and Tool Choice]] — reducing error surface by limiting tool access
- [[Multi-Agent Orchestration]] — error propagation patterns in agent hierarchies
- [[MCP Server Configuration]] — where error handling is configured
