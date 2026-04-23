---
tags:
  - domain-1
  - task-1-1
  - agentic-loop
  - stop-reason
  - tool-use
domain: "1 - Agentic Architecture & Orchestration"
task: "1.1"
---

# Agentic Loop Lifecycle

> **Task 1.1** — Understand the core agentic loop, how tool results feed back into conversation history, and model-driven decision-making vs. pre-configured decision trees.

---

## The Agentic Loop

The agentic loop is the fundamental execution pattern for Claude-based agents. It operates as a simple state machine:

```
┌─────────────────────────────────┐
│  Send request to Claude API     │
└──────────────┬──────────────────┘
               ▼
┌─────────────────────────────────┐
│  Inspect response.stop_reason   │
└──────────────┬──────────────────┘
               ▼
        ┌──────────────┐
        │ stop_reason?  │
        └──┬────────┬───┘
           │        │
     "tool_use"   "end_turn"
           │        │
           ▼        ▼
     Execute     Return final
     tool(s)     response to
           │     user
           ▼
     Append tool_result
     to messages array
           │
           └──► Loop back to
                send request
```

### Core Mechanics

1. **Send request** — The messages array (conversation history) plus system prompt are sent to the Claude API.
2. **Check `stop_reason`** — The API response includes a `stop_reason` field. This is the *only* reliable signal for controlling the loop.
3. **If `"tool_use"`** — The model wants to call one or more tools. Execute them and append the results back into the messages array as `tool_result` content blocks.
4. **If `"end_turn"`** — The model has decided it is finished. Return the response to the caller.

### How Tool Results Are Appended

Tool results are appended as a new `user` role message containing `tool_result` content blocks. Each `tool_result` must reference the `tool_use_id` from the corresponding tool call. This is how Claude maintains continuity between what it asked for and what it received.

```
messages = [
  { role: "user", content: "..." },
  { role: "assistant", content: [tool_use block] },
  { role: "user", content: [tool_result block] },   ← appended by your code
  { role: "assistant", content: [next response] },   ← from API
]
```

---

## Model-Driven Decision-Making vs. Pre-Configured Decision Trees

| Aspect | Model-Driven | Pre-Configured Decision Trees |
|--------|-------------|-------------------------------|
| **Control flow** | Model chooses which tool to call and when to stop | Developer hard-codes branching logic |
| **Flexibility** | Adapts to novel situations | Only handles anticipated paths |
| **Reliability signal** | `stop_reason` field | Parsing assistant text, counting iterations |
| **Exam stance** | Preferred approach | Anti-pattern when used as primary control |

The exam strongly favours **model-driven** control flow: the agent decides what to do next based on the results it sees, and the `stop_reason` field tells your harness whether the loop should continue.

---

## Anti-Patterns

### 1. Parsing natural language for loop termination
Scanning the assistant's text output for phrases like "I'm done" or "FINAL ANSWER" is fragile. The model may phrase its completion differently, use the phrase mid-reasoning, or omit it entirely.

### 2. Arbitrary iteration caps as primary stopping mechanism
A hard cap like `max_iterations = 5` used as the *primary* way to end the loop means the agent either stops too early (task incomplete) or the cap is set so high it is meaningless. Iteration caps are fine as a **safety net**, but the primary termination signal must be `stop_reason == "end_turn"`.

### 3. Checking assistant text content as completion indicator
Inspecting `response.content[0].text` for keywords to decide loop continuation conflates content with control flow. The `stop_reason` field exists precisely to separate these concerns.

---

## Examples

### Example 1 — Basic Agentic Loop in Python

```python
import anthropic

client = anthropic.Anthropic()

def run_agent(user_message: str, tools: list[dict]) -> str:
    """Core agentic loop checking stop_reason for control flow."""
    messages = [{"role": "user", "content": user_message}]

    while True:
        response = client.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=4096,
            tools=tools,
            messages=messages,
        )

        # Append assistant response to conversation history
        messages.append({"role": "assistant", "content": response.content})

        # PRIMARY CONTROL: check stop_reason
        if response.stop_reason == "end_turn":
            # Model has decided it is finished
            return extract_text(response.content)

        if response.stop_reason == "tool_use":
            # Execute every tool the model requested
            tool_results = []
            for block in response.content:
                if block.type == "tool_use":
                    result = execute_tool(block.name, block.input)
                    tool_results.append({
                        "type": "tool_result",
                        "tool_use_id": block.id,
                        "content": result,
                    })
            # Append results back into conversation history
            messages.append({"role": "user", "content": tool_results})
```

Key points: the loop only terminates on `stop_reason == "end_turn"`, tool results carry the `tool_use_id`, and both assistant and tool-result messages are appended to maintain full history.

---

### Example 2 — Tool Result Appending to Messages Array

```python
# After the model returns a tool_use block like:
# {
#   "type": "tool_use",
#   "id": "toolu_01ABC123",
#   "name": "get_customer",
#   "input": {"customer_id": "C-9382"}
# }

# Your code executes the tool and builds a tool_result:
tool_result_message = {
    "role": "user",
    "content": [
        {
            "type": "tool_result",
            "tool_use_id": "toolu_01ABC123",  # Must match the tool_use id
            "content": json.dumps({
                "customer_id": "C-9382",
                "name": "Alice Chen",
                "tier": "premium",
                "account_status": "active",
                "open_tickets": 2,
            }),
        }
    ],
}

# Append to messages — this is how the model sees the tool output
messages.append(tool_result_message)

# The messages array now looks like:
# [
#   {"role": "user",      "content": "Look up customer C-9382"},
#   {"role": "assistant", "content": [tool_use block]},
#   {"role": "user",      "content": [tool_result block]},  ← just appended
# ]
# The next API call sends this entire array so Claude has full context.
```

---

### Example 3 — Anti-Pattern: Iteration Cap as Primary Stopping Mechanism

```python
# BAD — iteration cap is the primary termination mechanism
def run_agent_bad(user_message: str, tools: list[dict], max_iterations: int = 5):
    messages = [{"role": "user", "content": user_message}]

    for i in range(max_iterations):  # ← arbitrary cap drives termination
        response = client.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=4096,
            tools=tools,
            messages=messages,
        )
        messages.append({"role": "assistant", "content": response.content})

        # Problem 1: stops even if model is mid-task at iteration 5
        # Problem 2: never checks stop_reason — doesn't know if model is done
        # Problem 3: if task needs 6 steps, user gets an incomplete result

        if response.stop_reason == "tool_use":
            tool_results = process_tool_calls(response.content)
            messages.append({"role": "user", "content": tool_results})
        else:
            # Falls through here on end_turn but also on hitting the cap
            break

    # May return incomplete results if cap was hit mid-task
    return extract_text(messages[-1])
```

Why this is wrong: the `for i in range(max_iterations)` loop means that a complex task requiring 6+ tool calls silently returns incomplete results. The iteration cap is masquerading as a control-flow mechanism when it should only be a safety net.

---

### Example 4 — Correct Implementation with Safety-Net Cap

```python
def run_agent_correct(user_message: str, tools: list[dict]) -> str:
    """Agentic loop with stop_reason as primary control and safety-net cap."""
    messages = [{"role": "user", "content": user_message}]
    MAX_ITERATIONS = 50  # Safety net only — not expected to trigger

    for _ in range(MAX_ITERATIONS):
        response = client.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=4096,
            tools=tools,
            messages=messages,
        )
        messages.append({"role": "assistant", "content": response.content})

        # PRIMARY CONTROL: model decides when it is done
        if response.stop_reason == "end_turn":
            return extract_text(response.content)

        if response.stop_reason == "tool_use":
            tool_results = []
            for block in response.content:
                if block.type == "tool_use":
                    result = execute_tool(block.name, block.input)
                    tool_results.append({
                        "type": "tool_result",
                        "tool_use_id": block.id,
                        "content": result,
                    })
            messages.append({"role": "user", "content": tool_results})

    # Safety net: if we somehow reach here, log a warning and return what we have
    logging.warning("Agent hit safety-net iteration cap — investigate task complexity")
    return extract_text(messages[-1].get("content", []))
```

The difference: `stop_reason == "end_turn"` is the primary exit condition. The iteration cap (`MAX_ITERATIONS = 50`) is deliberately high and only exists as a safety net. If it ever triggers, it is treated as an anomaly to investigate, not normal operation.

---

## Key Takeaways for the Exam

1. **`stop_reason` is the single source of truth** for loop control — not text parsing, not iteration counts.
2. Tool results must include the matching `tool_use_id` and are appended as `user` role messages.
3. Iteration caps are acceptable as **safety nets** but never as primary termination logic.
4. The model drives the decision of *what to do next* — your harness just executes tools and feeds results back.

---

## Related Topics

- [[Domain 1 - Agentic Architecture & Orchestration]] — Parent domain overview
- [[Multi-Agent Orchestration]] — Extending the loop to multi-agent systems
- [[Hooks and Interception]] — Adding deterministic enforcement within the loop
- [[Tool Design Principles]] — Designing tools the model can effectively invoke
- [[Session Management]] — Persisting and resuming agentic loop state
