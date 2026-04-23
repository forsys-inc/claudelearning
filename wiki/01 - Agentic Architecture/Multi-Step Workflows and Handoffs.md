---
tags:
  - domain-1
  - task-1-4
  - workflows
  - handoffs
  - programmatic-enforcement
  - prerequisite-gates
domain: "1 - Agentic Architecture & Orchestration"
task: "1.4"
---

# Multi-Step Workflows and Handoffs

> **Task 1.4** — Enforce workflow ordering through programmatic gates and hooks rather than relying solely on prompt instructions; design structured handoff protocols.

---

## The Core Problem

When an agent handles multi-step workflows (e.g., verify customer identity before processing a refund), the ordering of steps is often **critical**. Skipping or reordering steps can cause security violations, compliance failures, or incorrect outcomes.

There are two approaches to enforcing step ordering:

| Approach | Mechanism | Guarantee Level |
|----------|-----------|-----------------|
| **Prompt-based guidance** | System prompt instructions like "Always verify identity before processing refunds" | Probabilistic — works most of the time, but has a non-zero failure rate |
| **Programmatic enforcement** | Code-level gates, hooks, prerequisite checks that block execution of step N until step N-1 is complete | Deterministic — physically impossible to skip |

The exam tests a clear principle: **use programmatic enforcement for critical operations and prompt-based guidance for non-critical preferences.**

---

## Programmatic Enforcement

### Prerequisite Gates

A prerequisite gate is code that runs before a tool executes and blocks it if preconditions are not met. The tool literally cannot run until the gate is satisfied.

### Hooks

Pre-tool hooks can intercept a tool call and reject it based on state. Post-tool hooks can transform or validate results. See [[Hooks and Interception]] for deep coverage.

### State Tracking

The orchestration layer maintains a state object tracking which steps have completed. Tools check this state before executing.

---

## Non-Zero Failure Rate of Prompt Instructions

Even well-crafted prompt instructions fail some percentage of the time. For a refund workflow processing thousands of requests daily, even a 0.1% failure rate means several unauthorized refunds per day.

Factors that increase prompt instruction failure:
- Long conversations where early instructions scroll out of the context window
- Complex multi-step tasks where the model loses track of which step it is on
- Ambiguous user requests that push the model toward shortcutting steps
- Adversarial prompting by users attempting to bypass restrictions

---

## Structured Handoff Protocols

When an agent cannot or should not handle a task (complexity, authorization level, domain mismatch), it performs a **structured handoff** to another agent or a human. A good handoff includes:

1. **Context summary** — What has been done so far and what the user needs
2. **State snapshot** — Verified facts, tool results, customer data already gathered
3. **Reason for handoff** — Why this agent cannot continue
4. **Recommended next steps** — What the receiving agent/human should do first

---

## Examples

### Example 1 — Programmatic Prerequisite Blocking process_refund

```python
class WorkflowState:
    """Tracks which prerequisite steps have been completed."""

    def __init__(self):
        self.completed_steps: set[str] = set()
        self.verified_customer_id: str | None = None
        self.verification_timestamp: float | None = None

    def mark_complete(self, step: str, **metadata):
        self.completed_steps.add(step)
        for key, value in metadata.items():
            setattr(self, key, value)

    def is_complete(self, step: str) -> bool:
        return step in self.completed_steps


# Tool definitions with prerequisite checks built into execution
TOOL_PREREQUISITES = {
    "process_refund": {
        "requires": ["customer_verified"],
        "error_message": "Cannot process refund: customer identity not verified. "
                        "Call get_customer_verification first.",
    },
    "apply_account_credit": {
        "requires": ["customer_verified"],
        "error_message": "Cannot apply credit: customer identity not verified.",
    },
    "escalate_to_manager": {
        "requires": [],  # No prerequisites — always available
        "error_message": None,
    },
    "get_customer_verification": {
        "requires": [],  # This IS the first step
        "error_message": None,
    },
}


def execute_tool(tool_name: str, tool_input: dict, state: WorkflowState) -> str:
    """Execute a tool only if all prerequisites are satisfied."""

    prereqs = TOOL_PREREQUISITES.get(tool_name, {}).get("requires", [])

    for prereq in prereqs:
        if not state.is_complete(prereq):
            # BLOCK execution — return error to the model
            error_msg = TOOL_PREREQUISITES[tool_name]["error_message"]
            return json.dumps({
                "error": "PREREQUISITE_NOT_MET",
                "message": error_msg,
                "missing_step": prereq,
                "completed_steps": list(state.completed_steps),
            })

    # Prerequisites satisfied — execute the tool
    result = TOOL_IMPLEMENTATIONS[tool_name](tool_input)

    # Update state based on what just completed
    if tool_name == "get_customer_verification" and result.get("verified"):
        state.mark_complete(
            "customer_verified",
            verified_customer_id=tool_input["customer_id"],
            verification_timestamp=time.time(),
        )

    return json.dumps(result)
```

If the model tries to call `process_refund` before `get_customer_verification`, the gate returns a structured error. The model sees the error, understands the prerequisite, and calls the verification tool first. This is deterministic — there is no way to process a refund without verification.

---

### Example 2 — Decomposing a Multi-Concern Customer Request

```python
# Customer message: "I need a refund for order #1234, and also my account
# email is wrong — it should be alice@newdomain.com, and can you check
# if my premium membership auto-renews next month?"

# The agent must handle three independent concerns, each with its own
# workflow requirements:

workflow_plan = {
    "concern_1_refund": {
        "steps": [
            {"tool": "get_customer_verification", "reason": "Required before any account action"},
            {"tool": "get_order_details", "input": {"order_id": "1234"}},
            {"tool": "check_refund_eligibility", "input": {"order_id": "1234"}},
            {"tool": "process_refund", "input": {"order_id": "1234"}},  # Gate: requires verification
        ],
    },
    "concern_2_email_update": {
        "steps": [
            # Verification already done (shared state) — no need to re-verify
            {"tool": "update_account_email", "input": {"new_email": "alice@newdomain.com"}},
            {"tool": "send_email_confirmation", "input": {"email": "alice@newdomain.com"}},
        ],
    },
    "concern_3_membership_check": {
        "steps": [
            {"tool": "get_membership_details", "input": {}},
            # Read-only query — lower prerequisite bar
        ],
    },
}

# The agent handles these sequentially because verification (concern_1, step_1)
# is a prerequisite shared across concerns 1 and 2.
# Concern 3 is read-only and can proceed with lighter prerequisites.
```

The model decomposes the multi-concern request and addresses each concern with appropriate workflow constraints. The programmatic gate on `process_refund` ensures verification happens regardless of the model's reasoning about step ordering.

---

### Example 3 — Structured Handoff Summary Format

```python
# When the agent determines it must escalate to a human, it produces
# a structured handoff that ensures no context is lost.

handoff_summary = {
    "handoff_type": "escalation_to_human",
    "reason": "Refund amount ($847.50) exceeds automated processing limit ($500)",
    "customer": {
        "id": "C-9382",
        "name": "Alice Chen",
        "tier": "premium",
        "verified": True,
        "verification_method": "email_code",
        "verification_timestamp": "2024-03-15T09:25:00Z",
    },
    "context_summary": (
        "Customer requested refund for order #ORD-5521 (3 items, $847.50). "
        "Order confirmed delivered 2024-03-10. Customer reports items arrived "
        "damaged (photos provided in ticket SUP-2847). Refund eligibility "
        "confirmed — within 30-day window and damage claim is valid."
    ),
    "completed_steps": [
        "customer_verified",
        "order_details_retrieved",
        "refund_eligibility_confirmed",
        "damage_evidence_reviewed",
    ],
    "pending_steps": [
        "process_refund — blocked by $500 automated limit",
    ],
    "recommended_action": (
        "Approve refund of $847.50 to original payment method. All eligibility "
        "checks passed. Customer is a 3-year premium member with clean history."
    ),
    "attachments": ["ticket:SUP-2847", "photos:damage_evidence_3items.zip"],
}

# The human agent receiving this handoff can immediately approve the refund
# without re-verifying the customer or re-checking eligibility.
```

A structured handoff preserves all work already done and tells the receiving agent exactly where to pick up.

---

### Example 4 — Comparison: Prompt-Based vs. Programmatic Enforcement

```python
# PROMPT-BASED (probabilistic) — suitable for non-critical preferences
system_prompt_guidance = """When handling customer requests:
- Greet the customer warmly before diving into the issue.
- Summarize what you understood before taking action.
- Offer a follow-up check before closing the conversation.
These are best practices that improve customer satisfaction."""

# If the model skips the greeting or summary, nothing bad happens.
# The customer experience is slightly worse, but no security or
# compliance violation occurs. Prompt-based is fine here.


# PROGRAMMATIC (deterministic) — required for critical operations
def pre_tool_hook(tool_name: str, tool_input: dict, state: WorkflowState):
    """Blocks critical operations that lack prerequisites."""

    if tool_name == "process_refund":
        if not state.is_complete("customer_verified"):
            raise PrerequisiteError(
                "process_refund blocked: customer not verified"
            )
        if tool_input.get("amount", 0) > 500:
            raise EscalationRequired(
                f"Refund amount ${tool_input['amount']} exceeds $500 limit. "
                "Escalate to human agent."
            )

    if tool_name == "delete_account":
        if not state.is_complete("customer_verified"):
            raise PrerequisiteError("delete_account blocked: customer not verified")
        if not state.is_complete("deletion_confirmed_by_customer"):
            raise PrerequisiteError(
                "delete_account blocked: customer has not explicitly confirmed deletion"
            )

# If the model tries to skip verification, the code stops it.
# No prompt injection or model confusion can bypass this.


# DECISION FRAMEWORK:
# ┌─────────────────────────┬───────────────────────┐
# │ Use Prompt-Based When   │ Use Programmatic When  │
# ├─────────────────────────┼───────────────────────┤
# │ Failure = minor UX issue│ Failure = security/    │
# │                         │ compliance violation   │
# │ Style & tone guidance   │ Financial transactions │
# │ Suggested best practices│ Data deletion          │
# │ Non-critical ordering   │ Identity verification  │
# │ Response formatting     │ Authorization checks   │
# └─────────────────────────┴───────────────────────┘
```

---

## Key Takeaways for the Exam

1. **Prompt instructions have a non-zero failure rate** — never rely on them alone for critical workflow steps.
2. **Programmatic gates** physically prevent out-of-order execution by returning errors to the model.
3. **Structured handoffs** must include context summary, completed steps, pending steps, and recommended next actions.
4. **Use both approaches together**: prompts guide the model's reasoning and tone; gates enforce critical constraints.

---

## Related Topics

- [[Domain 1 - Agentic Architecture & Orchestration]] — Parent domain overview
- [[Hooks and Interception]] — Detailed hook mechanics for enforcement
- [[Agentic Loop Lifecycle]] — How tool execution fits into the loop
- [[Multi-Agent Orchestration]] — Handoffs between agents in multi-agent systems
- [[Prompt Engineering Principles]] — Writing effective prompt-based guidance
