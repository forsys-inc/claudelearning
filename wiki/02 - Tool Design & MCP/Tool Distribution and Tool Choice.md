---
tags:
  - domain-2
  - task-2-3
  - tool-distribution
  - tool-choice
  - agent-design
domain: "Tool Design & MCP Integration"
task: "2.3"
---

# Tool Distribution and Tool Choice (Task 2.3)

> **Core principle:** Every tool you give an agent is a tool it might misuse. Selection reliability degrades as tool count grows. Give each agent only the tools it needs, and use `tool_choice` to constrain selection when the next step is known.

---

## Tool Count and Selection Reliability

Empirically, Claude's tool selection accuracy drops significantly as the number of available tools increases:

- **4-5 tools:** High selection reliability. Claude can hold all descriptions in working memory and differentiate confidently.
- **8-10 tools:** Moderate reliability. Some similar tools may be confused.
- **15-18+ tools:** Significant degradation. Claude starts choosing tools based on superficial name matches rather than careful description analysis.

This is not a hard limit — it depends on how differentiated the descriptions are (see [[Tool Interface Design]]). But as a design heuristic: **if an agent has more than ~8 tools, ask whether it should be split into multiple agents with scoped tool sets.**

---

## Out-of-Specialization Tool Misuse

When an agent has access to tools outside its intended role, it will use them — often incorrectly.

**Example:** A `synthesis_agent` whose job is to combine research from multiple subagents also has access to `fetch_url`, a raw HTTP client. Instead of waiting for the `research_subagent` to return results, the synthesis agent starts fetching URLs itself — bypassing rate limiting, caching, and citation tracking that the research subagent provides.

The synthesis agent is not "wrong" for calling `fetch_url` — the tool is available and the description matches the need. The architectural mistake was giving it access.

---

## Scoped Tool Access

Each agent should receive only the tools appropriate for its role:

| Agent Role | Appropriate Tools | Inappropriate Tools |
|-----------|------------------|---------------------|
| Synthesis | `verify_fact`, `format_citation`, `merge_findings` | `fetch_url`, `query_database`, `write_file` |
| Research | `search_web`, `fetch_url`, `extract_content` | `format_citation`, `send_email` |
| Writer | `read_template`, `write_draft`, `check_grammar` | `query_database`, `deploy_service` |

This is not about security (though it helps) — it is about **reliability**. Fewer tools means better selection accuracy and less chance of the agent going off-script.

---

## The `tool_choice` Parameter

The API provides a `tool_choice` parameter that controls tool selection behavior at the request level:

### `"auto"` (default)
Claude decides whether to call a tool and which one. This is appropriate when the agent genuinely needs to choose among multiple options.

```json
{
  "tool_choice": { "type": "auto" }
}
```

### `"any"`
Claude must call *some* tool — it cannot respond with text only. Useful when you know the next step requires a tool call but any of the available tools could be appropriate.

```json
{
  "tool_choice": { "type": "any" }
}
```

### Forced Selection
Claude must call a *specific* tool. This is the strongest constraint — useful when the next step in a pipeline is deterministic.

```json
{
  "tool_choice": {
    "type": "tool",
    "name": "extract_metadata"
  }
}
```

**When to use forced selection:**
- The first step of a pipeline always requires the same tool (e.g., always extract metadata before analysis).
- A subagent has exactly one tool and should always call it.
- You want to eliminate selection overhead for a known-next-step.

---

## Examples

### Example 1: Restricting Synthesis Agent Tools

**Bad — synthesis agent has all tools:**

```python
synthesis_agent = Agent(
    system_prompt="You synthesize research findings into coherent reports.",
    tools=[
        # Research tools (not its job)
        search_web,
        fetch_url,
        query_database,
        # Synthesis tools (its actual job)
        verify_fact,
        merge_findings,
        format_citation,
        # Output tools (not its job)
        send_email,
        deploy_report,
        # Utility tools (tempting distractions)
        translate_text,
        convert_currency,
        calculate_statistics,
        generate_chart,
        # 13 tools — selection reliability is degraded
    ]
)
```

**Problem:** The synthesis agent sees `fetch_url` and starts doing its own research instead of synthesizing what the research subagent provides. It sees `send_email` and starts emailing partial results. With 13 tools, it frequently misroutes.

**Good — scoped to synthesis-only tools:**

```python
synthesis_agent = Agent(
    system_prompt="You synthesize research findings into coherent reports. "
                  "You receive pre-gathered research from subagents. "
                  "You do not fetch data yourself.",
    tools=[
        verify_fact,       # Check claims against provided sources
        merge_findings,    # Combine results from multiple research tasks
        format_citation,   # Generate proper citations
    ]
)
```

Three tools. Clear role boundaries. High selection reliability.

---

### Example 2: Replacing a Generic Tool with a Constrained Alternative

**Bad — generic `fetch_url` available to the synthesis agent:**

```json
{
  "name": "fetch_url",
  "description": "Fetches content from any URL. Returns raw HTML or JSON."
}
```

This tool is too powerful for a synthesis agent. It enables the agent to bypass the research pipeline entirely.

**Good — constrained `load_document` scoped to pre-approved sources:**

```json
{
  "name": "load_document",
  "description": "Loads a document from the internal research cache by document_id. Only documents previously fetched by the research subagent are available. Input: {document_id: string}. Returns: {title, content, source_url, fetched_at, citations[]}. Returns error if document_id is not in cache — do NOT attempt to fetch external URLs."
}
```

The synthesis agent can reference research results but cannot initiate new fetches. The tool name and description reinforce this boundary.

---

### Example 3: Scoped `verify_fact` Tool for Synthesis Agent

The synthesis agent needs to check whether claims in the merged report are supported by source documents. A properly scoped tool:

```json
{
  "name": "verify_fact",
  "description": "Checks whether a specific factual claim is supported by one or more source documents in the research cache. Input: {claim: string, source_document_ids: string[]}. Returns: {verdict: 'supported'|'contradicted'|'unaddressed', supporting_passages: [{doc_id, excerpt, page}], confidence: number}. This tool only checks against already-cached documents. It does not fetch new sources. If a source_document_id is not in the cache, it returns an error for that ID."
}
```

**Key design decisions:**
- The tool only works against cached documents (no external access).
- It returns structured verdicts the agent can use to decide whether to include a claim.
- Invalid document IDs produce per-ID errors, not a blanket failure.

```python
# Usage in synthesis pipeline
result = await synthesis_agent.run(
    messages=[{
        "role": "user",
        "content": f"Synthesize these research findings into a report:\n\n{research_results}"
    }],
    tools=[verify_fact, merge_findings, format_citation]
)
```

---

### Example 4: Forced `tool_choice` for Pipeline First Step

**Scenario:** A document processing pipeline always starts by extracting metadata (title, author, date, type) before any analysis. Without `tool_choice`, Claude might skip extraction and jump straight to analysis based on the raw text.

```python
# Step 1: Force metadata extraction
extraction_response = await client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    system="You are a document processing agent. Always extract metadata first.",
    tools=[
        {
            "name": "extract_metadata",
            "description": "Extracts structured metadata from a document: title, author, publication_date, document_type, language, page_count. Input: {document_text: string}. This must be called before any analysis tools."
        },
        {
            "name": "analyze_sentiment",
            "description": "Analyzes the sentiment and tone of a document. Input: {document_text: string, metadata: object}. Requires metadata from extract_metadata."
        },
        {
            "name": "extract_key_findings",
            "description": "Extracts key findings and conclusions. Input: {document_text: string, metadata: object}. Requires metadata from extract_metadata."
        }
    ],
    tool_choice={"type": "tool", "name": "extract_metadata"},  # Forced
    messages=[{"role": "user", "content": f"Process this document:\n\n{doc_text}"}]
)

# Step 2: Now let Claude choose which analysis tool(s) to use
metadata = parse_tool_result(extraction_response)

analysis_response = await client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=2048,
    system="You are a document processing agent.",
    tools=[analyze_sentiment, extract_key_findings],
    tool_choice={"type": "auto"},  # Claude chooses
    messages=[
        {"role": "user", "content": f"Process this document:\n\n{doc_text}"},
        extraction_response,  # Include the forced extraction step
        {"role": "user", "content": f"Metadata extracted: {metadata}. Now analyze the document."}
    ]
)
```

**Pattern:** Force the first step with `tool_choice: {type: "tool", name: "..."}`, then switch to `auto` for subsequent steps where Claude's judgment adds value.

---

## Decision Framework

```
How many tools does this agent need?
├─ 1 tool → Use forced tool_choice
├─ 2-5 tools → Use "auto" with good descriptions
├─ 6-10 tools → Review: can this agent be split?
└─ 10+ tools → Almost certainly needs splitting

Is the next step deterministic?
├─ Yes → Use forced tool_choice
├─ No, but a tool must be called → Use "any"
└─ No, and text response is valid → Use "auto"
```

---

## Summary Checklist

- [ ] Each agent has 4-5 tools maximum (strong preference)
- [ ] No agent has tools outside its defined role
- [ ] Generic tools (like `fetch_url`) are replaced with scoped alternatives
- [ ] `tool_choice` is set to forced when the next step is deterministic
- [ ] `tool_choice: "any"` is used when a tool call is required but the choice varies
- [ ] `tool_choice: "auto"` is the default for genuinely open-ended steps

---

## Related Topics

- [[Domain 2 - Tool Design & MCP Integration]] — domain overview
- [[Tool Interface Design]] — how descriptions affect selection within a tool set
- [[Structured Error Responses]] — error handling for the tools each agent uses
- [[Multi-Agent Orchestration]] — splitting agents to scope tools
- [[MCP Server Configuration]] — controlling which tools are discovered
