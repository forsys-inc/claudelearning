---
tags:
  - domain-5
  - task-5-2
  - escalation
  - ambiguity
  - customer-support
  - human-handoff
domain: "5 - Context Management & Reliability"
task: "5.2"
---

# Escalation and Ambiguity Resolution

> **Task 5.2** — Design escalation triggers and ambiguity resolution strategies: explicit customer requests, policy gaps, multiple-match disambiguation, and few-shot escalation criteria.

---

## Escalation Triggers

Not every difficult situation warrants escalation. The exam expects you to identify the specific triggers that should route a conversation to a human agent, and — critically — the triggers that should **not**.

### Reliable escalation triggers

1. **Explicit customer request** — The customer says "I want to speak to a human" or "transfer me to a manager." This must be honored immediately, regardless of whether the AI could resolve the issue.
2. **Policy exceptions or gaps** — The customer's request falls outside documented policy (e.g., requesting a price match against a competitor, asking for a courtesy credit beyond authorized limits).
3. **Inability to progress** — The agent has attempted resolution, tools have returned errors or insufficient data, and no further automated path exists.
4. **Safety or legal concerns** — The customer describes a safety issue with a product, threatens legal action, or the conversation involves regulated domains requiring licensed human judgment.

### Unreliable escalation signals

- **Sentiment analysis** — Detecting "angry" tone and escalating is unreliable. Customers may use strong language while still wanting a quick resolution, or may be politely frustrated in ways sentiment models miss.
- **Self-reported confidence** — The model saying "I'm not confident about this" is not a calibrated signal. Models are poor at self-assessing confidence on factual claims within conversation flow.

---

## Escalating on Explicit Customer Demand

When a customer explicitly requests a human agent, the correct behavior is **immediate escalation** — not attempting to resolve the issue first.

Common anti-pattern: *"I understand you'd like to speak to a human. Before I transfer you, let me try to help with..."* This creates friction and frustration. The customer has made a clear request; overriding it violates trust.

The one exception: if the request is ambiguous ("Can someone help me?"), it is appropriate to clarify whether they want a human or are simply asking for assistance.

---

## Multiple Customer Matches

When a customer lookup returns multiple matches (e.g., two customers named "John Smith"), the agent must **never** use heuristics to guess which one is correct. Common heuristics that fail:

- Selecting the most recently active account
- Choosing the account with the matching city
- Picking the account with open tickets

Instead, ask the customer for an additional identifier: account number, email address, phone number, or postal code.

---

## Few-Shot Examples for Escalation Criteria

Embedding few-shot examples of correct escalation decisions directly in the system prompt is the most reliable way to calibrate agent behavior. Abstract rules like "escalate when appropriate" are too vague; concrete examples establish clear decision boundaries.

---

## Examples

### Example 1 — Explicit Escalation Criteria with Few-Shot Examples in System Prompt

```python
SYSTEM_PROMPT = """You are a customer support agent for TechMart Electronics.

## Escalation Policy

Escalate to a human agent by calling the `escalate_to_human` tool when ANY
of these conditions are met:

1. Customer explicitly requests a human agent or manager
2. The request requires a policy exception you cannot authorize
3. You cannot resolve the issue after exhausting available tools
4. The situation involves product safety, legal threats, or regulatory concerns

DO NOT escalate based on:
- Your assessment of customer sentiment or frustration
- Your self-assessed confidence level
- The complexity of the question alone (try first)

### Few-Shot Examples

**Example A — Escalate immediately:**
Customer: "I've been going back and forth for 20 minutes. I want a real person."
Action: Call `escalate_to_human` with reason "customer_requested_human".
Do NOT say "Let me try one more thing." Honor the request immediately.

**Example B — Escalate on policy gap:**
Customer: "BestBuy has this for $50 less. Can you match their price?"
Action: Call `escalate_to_human` with reason "policy_exception_required"
and context "Customer requesting competitor price match — not in automated
authorization scope."

**Example C — Do NOT escalate (resolve directly):**
Customer: "This is ridiculous! My order still hasn't shipped!"
Action: Look up the order status. The customer is frustrated but has a
straightforward question. Provide the shipping status and estimated delivery.
Only escalate if the order has a genuine issue you cannot resolve.

**Example D — Do NOT escalate (clarify ambiguity):**
Customer: "Can someone help me with this?"
Action: This is NOT a request for a human. Respond with "I'd be happy to
help! Could you tell me more about what you need?"
"""
```

The few-shot examples establish clear boundaries: Example A vs. Example D shows the difference between an explicit transfer request and a general plea for help. Example B vs. Example C shows when frustration alone is insufficient for escalation.

---

### Example 2 — Customer Asks for Human: Honor Immediately

```python
# In the agentic loop, the model calls escalate_to_human.
# The tool definition makes the handoff seamless.

ESCALATION_TOOL = {
    "name": "escalate_to_human",
    "description": (
        "Transfer this conversation to a human agent. Use when the customer "
        "explicitly requests a human, when a policy exception is needed, or "
        "when you cannot resolve the issue with available tools."
    ),
    "input_schema": {
        "type": "object",
        "properties": {
            "reason": {
                "type": "string",
                "enum": [
                    "customer_requested_human",
                    "policy_exception_required",
                    "resolution_blocked",
                    "safety_or_legal",
                ],
                "description": "The category of escalation trigger.",
            },
            "context_summary": {
                "type": "string",
                "description": (
                    "Brief summary of the issue, steps already taken, "
                    "and any relevant case facts (order numbers, amounts) "
                    "for the human agent."
                ),
            },
            "case_facts": {
                "type": "object",
                "description": "Structured case facts to pass to the human agent.",
            },
        },
        "required": ["reason", "context_summary"],
    },
}

# Correct model behavior when customer says "Just let me talk to a person":
# Assistant response:
# "I'll connect you with a human agent right away."
# [tool_use: escalate_to_human]
#   reason: "customer_requested_human"
#   context_summary: "Customer Alice Chen (C-9382) has a return request for
#     order ORD-2026-8847 ($142.87 RunMax Pro shoes). Return window expired
#     2026-04-14. Customer requested human transfer."
#   case_facts: { order_number: "ORD-2026-8847", amount: 142.87, ... }
```

The `context_summary` and `case_facts` fields ensure the human agent has full context without re-asking the customer for information they already provided.

---

### Example 3 — Policy Gap Scenario: Competitor Price Matching

```python
# Conversation flow demonstrating correct escalation on policy gap.

# Turn 1 — Customer asks about price matching
# Customer: "I just saw the XPS-500 monitor is $80 cheaper on Amazon.
#            Can you match that price?"

# Turn 2 — Agent checks available policies
# The agent's system prompt and available tools include:
#   - apply_discount: max 15% for loyalty members
#   - process_return: standard return processing
#   - get_product_info: product details and pricing
# None of these tools authorize competitor price matching.

# Turn 3 — Agent escalates correctly
# Assistant: "I understand you'd like us to match Amazon's price on the
#   XPS-500 monitor. Competitor price matching requires authorization
#   from our team — let me connect you with someone who can help with that."
# [tool_use: escalate_to_human]
#   reason: "policy_exception_required"
#   context_summary: "Customer requesting competitor price match: XPS-500
#     monitor is $80 cheaper on Amazon. No automated price-match tool
#     available. Customer is loyalty tier Gold."
#   case_facts: {
#     product: "XPS-500 Monitor",
#     our_price: 449.99,
#     competitor_price: 369.99,
#     competitor: "Amazon",
#     customer_tier: "Gold"
#   }

# WRONG approach — trying to "resolve" it:
# "I can offer you a 15% loyalty discount, bringing it to $382.49.
#  That's close to Amazon's price!"
# This is wrong because:
# 1. The customer asked for price matching, not a loyalty discount
# 2. Creatively combining tools to approximate a policy exception is risky
# 3. If the discount is later audited, there's no authorized reason for it
```

---

### Example 4 — Multiple Customer Matches: Ask for Additional Identifiers

```python
# Customer says: "Hi, I need to check on my order. Name is John Smith."
# Agent calls: get_customer(name="John Smith")

# Tool returns multiple matches:
tool_result = {
    "matches": [
        {"customer_id": "C-4401", "name": "John Smith", "city": "Portland, OR"},
        {"customer_id": "C-7823", "name": "John Smith", "city": "Portland, ME"},
        {"customer_id": "C-9102", "name": "Jonathan Smith", "city": "Austin, TX"},
    ],
    "match_count": 3,
}

# CORRECT response — ask for additional identifier:
# "I found a few accounts under that name. To make sure I pull up the
#  right one, could you provide your email address or account number?"

# WRONG — using heuristics:
# "I found your account in Portland, OR!" (guessed based on... what?)
# "I see your most recent order..." (picked the most recently active one)
# "Did you mean the Portland, Oregon account?" (leaking other customers' data)

# The system prompt should include explicit instructions:
DISAMBIGUATION_INSTRUCTION = """
When a customer lookup returns multiple matches:
1. NEVER select a match using heuristics (most recent, closest city, etc.)
2. NEVER reveal details of other customers' accounts
3. Ask for ONE additional identifier: email, account number, phone, or postal code
4. If the customer cannot provide any identifier, escalate to a human agent
"""
```

Heuristic selection is dangerous: picking the wrong account means the agent might disclose another customer's order history, process a refund to the wrong person, or make changes to the wrong account. The only safe approach is disambiguation through additional identifiers provided by the customer.

---

## Key Takeaways for the Exam

1. **Honor explicit human requests immediately** — never try to resolve the issue first when the customer has asked for a person.
2. **Sentiment and self-confidence are unreliable escalation signals** — use objective triggers: explicit request, policy gap, tool failure, safety concern.
3. **Few-shot examples in the system prompt** are the most effective way to calibrate escalation decisions — abstract rules alone are insufficient.
4. **Never guess on ambiguous customer matches** — always ask for an additional identifier rather than applying heuristics.

---

## Related Topics

- [[Domain 5 - Context Management & Reliability]] — Parent domain overview
- [[Context Window Management]] — Passing case facts through escalation for human agent context
- [[Human Review Workflows]] — Routing decisions based on confidence and document type
- [[Prompt Engineering Fundamentals]] — Few-shot examples as a prompting technique
- [[Tool Design Principles]] — Designing escalation tools with structured context fields
