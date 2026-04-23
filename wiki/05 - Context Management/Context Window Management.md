---
tags:
  - domain-5
  - task-5-1
  - context-window
  - summarization
  - token-management
  - tool-output-trimming
domain: "5 - Context Management & Reliability"
task: "5.1"
---

# Context Window Management

> **Task 5.1** — Manage context window limits through progressive summarization, tool output trimming, position-aware ordering, and structured subagent outputs — while preserving critical transactional facts.

---

## Progressive Summarization Risks

Progressive summarization — condensing earlier conversation turns into shorter summaries — is the primary strategy for keeping long conversations within token limits. But naive summarization destroys the very information that matters most in transactional workflows.

### What gets lost

When a conversation about a customer return is summarized to *"Customer discussed a return issue with their recent order,"* the following critical facts vanish:

- **Numerical values**: refund amount ($142.87), order total ($289.99)
- **Dates**: order date (2026-03-15), return window expiry (2026-04-14)
- **Identifiers**: order number (ORD-2026-8847), tracking number (1Z999AA10123456784)
- **Commitments**: "I'll process your refund within 3 business days"

The model will then confabulate these details or hedge with vague language, destroying customer trust.

### The solution: case facts extraction

Before summarizing, extract transactional facts into a persistent structured block that survives every summarization pass.

---

## "Lost in the Middle" Effect

Models process tokens at the **beginning** (system prompt, initial instructions) and **end** (recent turns) of their context window most reliably. Information placed in the middle of a long context is disproportionately likely to be overlooked or misweighted.

### Implications for design

- Place critical instructions and constraints in the system prompt (beginning)
- Place the most relevant findings and current state near the end of the context
- When aggregating results from multiple sources, lead with a **key findings summary** rather than burying conclusions after pages of raw data

---

## Tool Result Token Accumulation

A single tool call to `get_order_details` might return 40+ fields (billing address, shipping address, item SKUs, warehouse codes, carrier metadata, internal flags). If only 5 fields are relevant to the current task (order status, items, refund eligibility, return window, tracking number), the other 35 fields consume tokens without value — and they accumulate across multiple tool calls.

In a 10-turn conversation with 3 tool calls per turn, verbose tool outputs can consume 60–80% of the context window, leaving little room for reasoning.

---

## Passing Complete Conversation History

Despite the need to trim, the agentic loop requires that **complete conversation history** is passed on every API call. The model has no persistent memory between calls — it sees only the messages array you send. Omitting earlier turns (rather than summarizing them) creates incoherent conversations where the model contradicts its own prior statements.

The tension between "keep everything for coherence" and "trim for token budget" is the core challenge of context management.

---

## Key Skills

### 1. Extracting transactional facts into a persistent "case facts" block
Maintain a structured block at a fixed position in the prompt (typically right after the system prompt or as the first user message) that is updated — never summarized away.

### 2. Trimming verbose tool outputs
Post-process tool results before appending them to the messages array. Strip fields irrelevant to the current task.

### 3. Position-aware input ordering
Structure aggregated inputs so critical information appears at the beginning and end, not buried in the middle.

### 4. Structured outputs from subagents with metadata
Subagents return structured JSON with findings, confidence, sources, and coverage — not free-text paragraphs that the coordinator must parse.

---

## Examples

### Example 1 — "Case Facts" Block Persisted Across Prompts

```python
# Case facts are extracted from tool results and conversation,
# then injected into every subsequent API call as a persistent block.

case_facts = {
    "customer_id": "C-9382",
    "customer_name": "Alice Chen",
    "customer_tier": "premium",
    "order_number": "ORD-2026-8847",
    "order_date": "2026-03-15",
    "order_total": 289.99,
    "items": [
        {"sku": "SHOE-RUN-42", "name": "RunMax Pro 42", "price": 142.87},
        {"sku": "SOCK-CMP-3P", "name": "Compression Socks 3-Pack", "price": 34.99},
    ],
    "return_window_expires": "2026-04-14",
    "refund_amount_approved": 142.87,
    "commitment": "Refund processed within 3 business days",
    "commitment_made_at": "2026-04-01T14:32:00Z",
}

def build_messages(case_facts: dict, conversation_summary: str, recent_turns: list) -> list:
    """Build messages array with case facts block that survives summarization."""
    messages = [
        {
            "role": "user",
            "content": (
                "## Active Case Facts (DO NOT summarize — preserve exactly)\n"
                f"```json\n{json.dumps(case_facts, indent=2)}\n```\n\n"
                "## Conversation Summary\n"
                f"{conversation_summary}\n\n"
                "## Recent Conversation\n"
                "(see following messages)"
            ),
        },
    ]
    # Append recent turns verbatim — these have full fidelity
    messages.extend(recent_turns)
    return messages
```

The case facts block is labeled with an explicit instruction to preserve it. When the conversation is summarized, the summarization logic skips this block and only condenses the conversation narrative.

---

### Example 2 — Trimming Order Lookup from 40 Fields to 5 Return-Relevant Fields

```python
def trim_order_for_return(raw_order: dict) -> dict:
    """Trim verbose order data to only fields relevant for return processing.

    Raw order contains 40+ fields: billing address, shipping address,
    warehouse codes, carrier metadata, internal audit flags, etc.
    For return processing, only 5 fields matter.
    """
    return {
        "order_id": raw_order["order_id"],
        "order_date": raw_order["order_date"],
        "items": [
            {
                "sku": item["sku"],
                "name": item["product_name"],
                "price": item["unit_price"],
                "returnable": item["is_returnable"],
            }
            for item in raw_order["line_items"]
        ],
        "return_window_expires": raw_order["return_policy"]["window_end"],
        "refund_eligible": raw_order["return_policy"]["refund_eligible"],
        "tracking_number": raw_order.get("shipment", {}).get("tracking_number"),
    }

# In the agentic loop, trim before appending to messages:
raw_result = execute_tool("get_order_details", {"order_id": "ORD-2026-8847"})
trimmed_result = trim_order_for_return(json.loads(raw_result))

tool_result_message = {
    "role": "user",
    "content": [{
        "type": "tool_result",
        "tool_use_id": block.id,
        "content": json.dumps(trimmed_result),
    }],
}
messages.append(tool_result_message)
```

This reduces a typical order response from ~2,000 tokens to ~300 tokens — a 6x reduction per tool call.

---

### Example 3 — Key Findings Summary at Beginning of Aggregated Inputs

```python
def build_research_context(subagent_results: list[dict]) -> str:
    """Position-aware ordering: key findings first, details after.

    Exploits the model's tendency to attend more strongly to the
    beginning and end of context (lost-in-the-middle effect).
    """
    # BEGINNING: Executive summary with key findings
    summary_lines = ["## Key Findings (Read First)\n"]
    for result in subagent_results:
        summary_lines.append(
            f"- **{result['topic']}**: {result['key_finding']} "
            f"(confidence: {result['confidence']}, sources: {result['source_count']})"
        )
    summary_lines.append("")

    # MIDDLE: Detailed sections (less critical — model may skim)
    detail_sections = ["\n## Detailed Results by Topic\n"]
    for result in subagent_results:
        detail_sections.append(f"### {result['topic']}")
        detail_sections.append(result["detailed_findings"])
        detail_sections.append(f"*Sources: {', '.join(result['sources'])}*\n")

    # END: Synthesis instructions and coverage gaps (high attention)
    footer = [
        "\n## Coverage & Gaps\n",
        "Topics with incomplete coverage: "
        + ", ".join(r["topic"] for r in subagent_results if r["confidence"] < 0.7),
        "\n**Instruction**: Synthesize findings into a coherent report. "
        "Flag any conflicting data points with source attribution.",
    ]

    return "\n".join(summary_lines + detail_sections + footer)
```

By placing the key findings summary at the top and synthesis instructions at the bottom, the model reliably incorporates both, even when the detailed middle sections span thousands of tokens.

---

### Example 4 — Subagent Structured Output with Metadata for Synthesis

```python
# Subagent system prompt instructs it to return structured JSON, not prose.
SUBAGENT_SYSTEM_PROMPT = """You are a research subagent. Your task is to investigate
a specific topic and return structured findings.

ALWAYS return your response as a JSON object with this schema:
{
  "topic": "string — the topic you investigated",
  "key_finding": "string — one-sentence summary of the most important finding",
  "detailed_findings": "string — full analysis in markdown",
  "confidence": "float 0.0-1.0 — how confident you are in your findings",
  "sources": ["list of source identifiers or URLs"],
  "source_count": "int — number of distinct sources consulted",
  "coverage_gaps": ["list of aspects you could not fully investigate"],
  "conflicts": ["list of conflicting data points found, with source attribution"]
}
"""

# Coordinator collects structured results from multiple subagents:
subagent_results = [
    {
        "topic": "Q1 2026 Revenue",
        "key_finding": "Revenue grew 12% YoY to $4.2B, driven by cloud services.",
        "detailed_findings": "Cloud services revenue increased 23% to $1.8B...",
        "confidence": 0.92,
        "sources": ["10-K filing 2026-03-01", "Earnings call transcript 2026-02-28"],
        "source_count": 2,
        "coverage_gaps": ["Segment-level margins not disclosed until 10-Q"],
        "conflicts": [],
    },
    {
        "topic": "Competitive Landscape",
        "key_finding": "Market share stable at 18%, but two new entrants gained 3% combined.",
        "detailed_findings": "IDC report shows overall market grew 8%...",
        "confidence": 0.74,
        "sources": ["IDC Market Report Q1 2026", "Gartner Peer Insights"],
        "source_count": 2,
        "coverage_gaps": ["Pricing comparison data unavailable"],
        "conflicts": ["IDC reports 18% share vs Gartner's 16.5% estimate"],
    },
]

# The coordinator can now synthesize without re-reading raw research.
# Structured fields enable programmatic aggregation of coverage gaps,
# confidence-weighted conclusions, and explicit conflict handling.
```

Structured subagent output enables the coordinator to synthesize results programmatically — sorting by confidence, aggregating coverage gaps, and surfacing conflicts — without parsing free-text narratives.

---

## Key Takeaways for the Exam

1. **Never summarize away transactional facts** — extract amounts, dates, identifiers, and commitments into a persistent block before compressing conversation history.
2. **Trim tool outputs at the harness level** — strip irrelevant fields before appending tool results to the messages array, not after.
3. **Position matters** — place key findings at the beginning and instructions at the end of aggregated contexts to mitigate the lost-in-the-middle effect.
4. **Subagents should return structured data** — JSON with metadata fields (confidence, sources, gaps) is far more useful to a coordinator than free-text paragraphs.

---

## Related Topics

- [[Domain 5 - Context Management & Reliability]] — Parent domain overview
- [[Agentic Loop Lifecycle]] — How tool results are appended to conversation history
- [[Multi-Agent Orchestration]] — Coordinator-subagent patterns that generate context management challenges
- [[Error Propagation in Multi-Agent Systems]] — Structured error returns follow the same pattern as structured subagent outputs
- [[Information Provenance and Uncertainty]] — Preserving source attribution through summarization
