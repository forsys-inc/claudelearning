---
tags:
  - exercise
  - domain-1
  - domain-2
  - domain-5
  - agentic-loop
  - tool-design
  - error-handling
  - hooks
---

# Exercise 1 — Multi-Tool Agent

> **Domains covered:** [[Domain 1 - Agentic Architecture & Orchestration]], [[Domain 2 - Tool Design & MCP Integration]], [[Domain 5 - Context Management & Reliability]]

Build a customer support agent with multiple tools, an agentic loop, structured error handling, and a programmatic enforcement hook.

---

## Prerequisites

- Familiarity with the Claude API `tool_use` message flow
- Understanding of [[Agentic Loop Lifecycle]] (stop_reason checking)
- Understanding of [[Tool Interface Design]] (differentiated descriptions)
- Understanding of [[Structured Error Responses]] (errorCategory, isRetryable)

---

## Step 1: Define 3-4 MCP Tools with Differentiated Descriptions

Define tools with detailed, non-overlapping descriptions. Each description should specify input formats, output shapes, use cases, and boundary conditions.

```python
tools = [
    {
        "name": "lookup_customer",
        "description": (
            "Retrieves a customer profile by customer_id or email. "
            "Returns: name, email, account_created_date, subscription_tier, lifetime_spend. "
            "Does NOT return order history — use lookup_order for that. "
            "Input: {customer_id: string} or {email: string}. "
            "Returns 404-style error if no matching customer exists."
        ),
        "input_schema": {
            "type": "object",
            "properties": {
                "customer_id": {"type": "string", "description": "The unique customer identifier"},
                "email": {"type": "string", "description": "Customer email address (alternative to customer_id)"}
            }
        }
    },
    {
        "name": "lookup_order",
        "description": (
            "Retrieves details for a specific order by order_id, or lists recent orders for a customer_id. "
            "Returns: order_id, items[], total_amount, status (pending|shipped|delivered|cancelled), tracking_number. "
            "Input: {order_id: string} or {customer_id: string, limit?: number (default 10)}. "
            "Returns empty array (not error) if customer has no orders."
        ),
        "input_schema": {
            "type": "object",
            "properties": {
                "order_id": {"type": "string"},
                "customer_id": {"type": "string"},
                "limit": {"type": "integer", "default": 10}
            }
        }
    },
    {
        "name": "process_refund",
        "description": (
            "Processes a refund for a specific order. Requires a valid order_id for an order in "
            "'delivered' or 'shipped' status. Returns: refund_id, amount_refunded, estimated_arrival. "
            "PREREQUISITE: lookup_order must be called first to verify the order exists and is eligible. "
            "Input: {order_id: string, reason: string}. "
            "Returns business error if order is not in refundable status."
        ),
        "input_schema": {
            "type": "object",
            "properties": {
                "order_id": {"type": "string", "description": "The order to refund"},
                "reason": {"type": "string", "description": "Reason for the refund"}
            },
            "required": ["order_id", "reason"]
        }
    },
    {
        "name": "escalate_to_human",
        "description": (
            "Transfers the conversation to a human support agent. Use ONLY when: "
            "(1) customer explicitly requests a human, (2) issue involves a policy exception, "
            "(3) refund amount exceeds $500, or (4) customer has expressed frustration 3+ times. "
            "Input: {reason: string, context_summary: string, priority: 'low'|'medium'|'high'}. "
            "Returns: {ticket_id, estimated_wait_minutes}."
        ),
        "input_schema": {
            "type": "object",
            "properties": {
                "reason": {"type": "string"},
                "context_summary": {"type": "string"},
                "priority": {"type": "string", "enum": ["low", "medium", "high"]}
            },
            "required": ["reason", "context_summary", "priority"]
        }
    }
]
```

**Checkpoint:** Review your descriptions. Can you tell *exactly* when to use each tool and what it returns? Are there any ambiguities or overlaps?

---

## Step 2: Implement the Agentic Loop Checking stop_reason

The core loop sends a request to Claude, checks `stop_reason`, and either executes tools or returns the final response.

```python
import anthropic

client = anthropic.Anthropic()

def run_agent(user_message: str, conversation_history: list = None):
    """Core agentic loop that checks stop_reason to decide whether to continue."""

    if conversation_history is None:
        conversation_history = []

    # Add user message
    conversation_history.append({"role": "user", "content": user_message})

    system_prompt = (
        "You are a customer support agent for an e-commerce company. "
        "You help customers with order inquiries, refunds, and account questions. "
        "Always look up relevant data before making decisions. "
        "Be concise and helpful."
    )

    while True:
        # Call Claude
        response = client.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=1024,
            system=system_prompt,
            tools=tools,
            messages=conversation_history,
        )

        # Check stop_reason — this is the key decision point
        if response.stop_reason == "end_turn":
            # Model is done — extract and return the text response
            assistant_text = ""
            for block in response.content:
                if block.type == "text":
                    assistant_text += block.text
            conversation_history.append({"role": "assistant", "content": response.content})
            return assistant_text, conversation_history

        elif response.stop_reason == "tool_use":
            # Model wants to call a tool — execute it and feed the result back
            conversation_history.append({"role": "assistant", "content": response.content})

            tool_results = []
            for block in response.content:
                if block.type == "tool_use":
                    # Execute the tool (Step 3 adds error handling here)
                    result = execute_tool(block.name, block.input)
                    tool_results.append({
                        "type": "tool_result",
                        "tool_use_id": block.id,
                        "content": result
                    })

            conversation_history.append({"role": "user", "content": tool_results})
            # Loop continues — Claude will process the tool results

        else:
            # Unexpected stop_reason (e.g., max_tokens) — handle gracefully
            raise RuntimeError(f"Unexpected stop_reason: {response.stop_reason}")
```

**Checkpoint:** Trace through the loop mentally. What happens if Claude calls two tools in one turn? What happens if `stop_reason` is `max_tokens`?

---

## Step 3: Add Structured Error Responses

Replace simple error strings with structured objects containing `errorCategory` and `isRetryable`.

```python
def execute_tool(tool_name: str, tool_input: dict) -> str:
    """Execute a tool and return a structured response (success or error)."""
    import json

    try:
        if tool_name == "lookup_customer":
            result = _lookup_customer(tool_input)
        elif tool_name == "lookup_order":
            result = _lookup_order(tool_input)
        elif tool_name == "process_refund":
            result = _process_refund(tool_input)
        elif tool_name == "escalate_to_human":
            result = _escalate_to_human(tool_input)
        else:
            return json.dumps({
                "isError": True,
                "errorCategory": "validation",
                "errorCode": "UNKNOWN_TOOL",
                "isRetryable": False,
                "message": f"Unknown tool: {tool_name}"
            })

        return json.dumps(result)

    except ConnectionError:
        # Transient — agent should retry
        return json.dumps({
            "isError": True,
            "errorCategory": "transient",
            "errorCode": "CONNECTION_FAILED",
            "isRetryable": True,
            "retryAfterMs": 2000,
            "message": "Database connection timed out. Retry in 2 seconds."
        })

    except PermissionError:
        # Permission — agent should escalate
        return json.dumps({
            "isError": True,
            "errorCategory": "permission",
            "errorCode": "INSUFFICIENT_SCOPE",
            "isRetryable": False,
            "message": "This operation requires elevated permissions."
        })


def _lookup_customer(input: dict) -> dict:
    """Stub — replace with real implementation."""
    # TODO: Connect to your customer database
    # Return structured data on success:
    return {
        "isError": False,
        "name": "Jane Smith",
        "email": "jane@example.com",
        "subscription_tier": "premium",
        "lifetime_spend": 2450.00
    }


def _lookup_order(input: dict) -> dict:
    """Stub — replace with real implementation."""
    # NOTE: Empty list is a valid success, not an error
    return {
        "isError": False,
        "orders": [
            {
                "order_id": "ord_123",
                "items": [{"name": "Widget", "qty": 2}],
                "total_amount": 49.99,
                "status": "delivered"
            }
        ]
    }


def _process_refund(input: dict) -> dict:
    """Stub — replace with real implementation."""
    # Business rule example: non-refundable status
    order_status = "pending"  # Simulated — fetch from DB in practice
    if order_status not in ("delivered", "shipped"):
        return {
            "isError": True,
            "errorCategory": "business",
            "errorCode": "ORDER_NOT_REFUNDABLE",
            "isRetryable": False,
            "message": f"Order {input['order_id']} is in '{order_status}' status and cannot be refunded. "
                       f"Only 'delivered' or 'shipped' orders are eligible."
        }
    return {
        "isError": False,
        "refund_id": "ref_456",
        "amount_refunded": 49.99,
        "estimated_arrival": "3-5 business days"
    }


def _escalate_to_human(input: dict) -> dict:
    """Stub — replace with real implementation."""
    return {
        "isError": False,
        "ticket_id": "tkt_789",
        "estimated_wait_minutes": 5
    }
```

**Checkpoint:** Does your `process_refund` stub distinguish business errors from transient errors? Can the agent tell the difference and act accordingly?

---

## Step 4: Implement a Programmatic Hook for Business Rule Enforcement

Add a PostToolUse hook that enforces: `process_refund` cannot execute unless `lookup_order` was called first with the same order_id.

```python
class ToolCallTracker:
    """Tracks which tools have been called with which arguments."""

    def __init__(self):
        self.call_history = []  # List of (tool_name, tool_input) tuples

    def record_call(self, tool_name: str, tool_input: dict):
        self.call_history.append((tool_name, tool_input))

    def was_called_with(self, tool_name: str, **kwargs) -> bool:
        """Check if a tool was previously called with specific arguments."""
        for name, input_data in self.call_history:
            if name == tool_name:
                if all(input_data.get(k) == v for k, v in kwargs.items()):
                    return True
        return False


def pre_tool_hook(tracker: ToolCallTracker, tool_name: str, tool_input: dict) -> dict | None:
    """
    Programmatic gate that runs BEFORE tool execution.
    Returns None to allow execution, or an error dict to block it.
    """
    if tool_name == "process_refund":
        order_id = tool_input.get("order_id")

        # Enforce prerequisite: lookup_order must have been called for this order
        if not tracker.was_called_with("lookup_order", order_id=order_id):
            return {
                "isError": True,
                "errorCategory": "validation",
                "errorCode": "PREREQUISITE_NOT_MET",
                "isRetryable": False,
                "message": (
                    f"Cannot process refund for order {order_id}: "
                    f"lookup_order must be called first to verify order eligibility. "
                    f"Please look up the order before attempting a refund."
                )
            }

    return None  # Allow execution


# Updated execute_tool with hook integration
def execute_tool_with_hooks(
    tracker: ToolCallTracker,
    tool_name: str,
    tool_input: dict
) -> str:
    """Execute a tool with pre-execution hook checking."""
    import json

    # Run pre-tool hook
    hook_result = pre_tool_hook(tracker, tool_name, tool_input)
    if hook_result is not None:
        return json.dumps(hook_result)

    # Execute the tool
    result = execute_tool(tool_name, tool_input)

    # Record the call (only on success)
    parsed = json.loads(result)
    if not parsed.get("isError", False):
        tracker.record_call(tool_name, tool_input)

    return result
```

**Checkpoint:** What happens if the agent tries to call `process_refund` without `lookup_order`? Does the error message guide the agent toward the correct behavior?

---

## Step 5: Test with Multi-Concern Messages

Test with messages that require the agent to use multiple tools and handle various scenarios.

```python
# Test cases — run each through run_agent() and verify behavior

test_cases = [
    # Test 1: Simple order lookup (should call lookup_customer + lookup_order)
    "I'd like to check the status of my order. My email is jane@example.com and the order is ord_123.",

    # Test 2: Refund request (should call lookup_order THEN process_refund)
    "I want a refund for order ord_123. The product arrived damaged.",

    # Test 3: Refund without order context (hook should block, then agent should look up order first)
    "Process a refund for ord_999 immediately.",

    # Test 4: Escalation trigger (customer requests human)
    "I've explained this three times now and I'm very frustrated. Please connect me to a real person.",

    # Test 5: Transient error handling (simulate connection failure)
    "Look up my account — my customer ID is cust_timeout_test.",

    # Test 6: Multi-concern message (should handle both questions)
    "Can you check my account status and also tell me about my recent orders? Email: jane@example.com",
]

# Expected behaviors:
# Test 1: Two tool calls, both succeed, agent synthesizes response
# Test 2: lookup_order → process_refund (in order, hook passes)
# Test 3: Hook blocks process_refund → agent calls lookup_order → then retries process_refund
# Test 4: Agent calls escalate_to_human with appropriate priority
# Test 5: Transient error → agent sees isRetryable:true → retries
# Test 6: Agent calls lookup_customer AND lookup_order (possibly in parallel)
```

**Checkpoint:** For each test case, predict which tools will be called and in what order. Run the tests and compare against your predictions.

---

## Verification Checklist

- [ ] All tool descriptions are detailed and non-overlapping
- [ ] Agentic loop correctly checks `stop_reason` on every iteration
- [ ] Loop handles both `end_turn` and `tool_use` stop reasons
- [ ] Error responses include `errorCategory`, `isRetryable`, and `message`
- [ ] Business errors are distinguished from transient errors
- [ ] Empty results are returned as successes, not errors
- [ ] PreToolUse hook enforces `lookup_order` before `process_refund`
- [ ] Hook returns a helpful error message that guides the agent
- [ ] Multi-concern messages trigger appropriate parallel or sequential tool calls

---

## Related Topics

- [[Agentic Loop Lifecycle]] — the stop_reason-driven loop pattern
- [[Tool Interface Design]] — writing differentiated descriptions
- [[Structured Error Responses]] — errorCategory and isRetryable patterns
- [[Hooks and Interception]] — programmatic enforcement hooks
- [[Multi-Step Workflows and Handoffs]] — ordering constraints
- [[Sample Questions]] — Q1, Q2, Q3 test these concepts
