---
tags:
  - domain-2
  - task-2-1
  - tool-design
  - tool-descriptions
  - llm-tool-selection
domain: "Tool Design & MCP Integration"
task: "2.1"
---

# Tool Interface Design (Task 2.1)

> **Core principle:** The tool description is the *only* information Claude uses to decide which tool to call. The model never sees source code, handler logic, or database schemas. If the description is vague, selection is unreliable.

---

## Why Descriptions Matter

When Claude receives a list of tools, it performs a soft ranking based on:

1. **Tool name** — a short signal, but not sufficient on its own.
2. **Tool description** — the primary decision driver. This is where input formats, intended use cases, edge case behavior, and boundary conditions should live.
3. **Parameter schemas** — types and field descriptions refine the call, but selection happens first.

A minimal description like `"Gets customer info"` gives Claude almost nothing to differentiate it from `"Looks up customer data"` or `"Fetches customer record"`. The result is random-feeling routing that degrades as tool count grows.

---

## Failure Modes

### 1. Ambiguous / Overlapping Descriptions
When two tools have similar names and vague descriptions, Claude cannot reliably choose between them.

**Problem pair:**
- `analyze_content` — "Analyzes content and returns results"
- `analyze_document` — "Analyzes a document and returns results"

Claude will oscillate between these tools because nothing in the descriptions clarifies *what kind* of content, *what kind* of analysis, or *what shape* the results take.

### 2. System Prompt Keyword Sensitivity
If your system prompt says *"You are a web research assistant that analyzes content from the web"*, Claude develops an association between the phrase "analyzes content" and its role. When it sees a tool called `analyze_content`, it will favor that tool even for tasks where another tool is more appropriate — the system prompt keywords create an unintended affinity.

### 3. Generic Names That Invite Misuse
A tool named `fetch_data` with description "Fetches data from a source" will be called for *any* data retrieval task, even when a more specific tool exists.

---

## Design Principles

### Write Differentiated Descriptions
Each description should answer:
- **What** does this tool do specifically?
- **When** should Claude choose it over similar tools?
- **What** are the input formats and constraints?
- **What** does the output look like?
- **What** edge cases or limitations exist?

### Rename to Eliminate Overlap
If two tools have overlapping names, rename them so the names alone signal different purposes.

### Split Generic Tools
A single tool that does many things is harder for Claude to use correctly than several focused tools.

---

## Examples

### Example 1: Bad Minimal Descriptions vs Good Detailed Descriptions

**Bad — minimal, undifferentiated:**

```json
{
  "tools": [
    {
      "name": "get_customer",
      "description": "Gets customer information."
    },
    {
      "name": "lookup_order",
      "description": "Looks up an order."
    }
  ]
}
```

Claude cannot tell whether `get_customer` returns order history, or whether `lookup_order` includes customer details. It may call the wrong tool or call both unnecessarily.

**Good — detailed, differentiated:**

```json
{
  "tools": [
    {
      "name": "get_customer",
      "description": "Retrieves a customer profile by customer_id or email address. Returns: name, email, account_created_date, subscription_tier, and lifetime_spend. Does NOT return order history — use lookup_order for that. Input: {customer_id: string} or {email: string}. Returns 404-style error if no matching customer exists."
    },
    {
      "name": "lookup_order",
      "description": "Retrieves details for a specific order by order_id, or lists recent orders for a customer_id. Returns: order_id, items[], total_amount, status (pending|shipped|delivered|cancelled), and tracking_number if shipped. Input: {order_id: string} or {customer_id: string, limit?: number (default 10)}. Returns empty array (not error) if customer has no orders."
    }
  ]
}
```

Now Claude knows exactly when to use each tool, what inputs are valid, and what to expect in the response — including the difference between "no orders" (empty array) and "no customer" (error).

---

### Example 2: Splitting a Generic Tool into Purpose-Specific Tools

**Before — one overloaded tool:**

```json
{
  "name": "analyze_document",
  "description": "Analyzes a document and returns results."
}
```

This tool is called for extraction, summarization, and fact-checking — three very different tasks with different output shapes. Claude has to guess what "analyze" means in context.

**After — three focused tools:**

```json
{
  "tools": [
    {
      "name": "extract_data_points",
      "description": "Extracts structured data points (dates, monetary amounts, named entities, percentages) from a document. Input: {document_id: string, data_types?: string[] (default: all)}. Returns: {data_points: [{type, value, location_in_doc, confidence}]}. Use this when the user needs specific facts or figures pulled out of a document."
    },
    {
      "name": "summarize_content",
      "description": "Generates a concise summary of a document at a specified detail level. Input: {document_id: string, detail_level: 'brief' | 'standard' | 'detailed', focus_topics?: string[]}. Returns: {summary: string, key_points: string[], word_count: number}. Use this when the user wants an overview, not specific data extraction."
    },
    {
      "name": "verify_claim_against_source",
      "description": "Checks whether a specific claim is supported, contradicted, or unaddressed by a source document. Input: {claim: string, source_document_id: string}. Returns: {verdict: 'supported' | 'contradicted' | 'not_addressed', relevant_passages: string[], confidence: number}. Use this for fact-checking and claim verification tasks."
    }
  ]
}
```

Each tool has a clear purpose, distinct input shape, and distinct output shape. Claude can match user intent to the right tool without ambiguity.

---

### Example 3: Renaming to Eliminate Overlap with System Prompt Keywords

**Problem:** The system prompt says *"You help users analyze content from web searches."* A tool named `analyze_content` exists alongside `extract_web_results`. Claude over-selects `analyze_content` because the system prompt phrase "analyze content" creates keyword affinity.

**Before:**

```json
{
  "name": "analyze_content",
  "description": "Analyzes content from various sources."
}
```

**After — renamed with web-specific description:**

```json
{
  "name": "extract_web_results",
  "description": "Parses raw HTML or search engine result pages and extracts structured results: title, URL, snippet, and publication_date for each result. Input: {raw_html: string, max_results?: number (default 10)}. Returns: {results: [{title, url, snippet, date}]}. Only works with HTML input — do not use for PDFs, plain text, or API responses."
}
```

The rename eliminates the keyword overlap with the system prompt. The description makes clear this tool is only for HTML parsing, preventing misuse for other analysis tasks.

---

### Example 4: System Prompt Keyword Issue Creating Unintended Associations

**Scenario:** You have a multi-tool agent with this system prompt:

```
You are a financial compliance assistant. You review transactions,
flag suspicious activity, and generate compliance reports.
```

And these tools:

```json
{
  "tools": [
    {
      "name": "flag_transaction",
      "description": "Flags a transaction for manual review."
    },
    {
      "name": "generate_report",
      "description": "Generates a report."
    },
    {
      "name": "review_customer_profile",
      "description": "Reviews a customer profile for completeness."
    }
  ]
}
```

**Problem:** The system prompt mentions "review transactions" and "flag suspicious activity." Claude develops associations:
- "review" in the system prompt creates affinity with `review_customer_profile`, causing Claude to call it even when the user is asking about *transactions*, not customer profiles.
- "flag" in the system prompt creates affinity with `flag_transaction`, causing it to be called prematurely before evidence gathering is complete.

**Fix:** Rename tools to avoid echoing system prompt verbs, and add boundary descriptions:

```json
{
  "tools": [
    {
      "name": "mark_transaction_suspicious",
      "description": "Marks a specific transaction as suspicious and queues it for manual compliance review. Input: {transaction_id: string, reason: string, severity: 'low'|'medium'|'high'}. Only call this AFTER you have gathered evidence — do not call speculatively. Returns: {case_id: string, status: 'queued'}."
    },
    {
      "name": "build_compliance_report",
      "description": "Compiles a compliance report for a date range. Input: {start_date: string, end_date: string, report_type: 'daily'|'weekly'|'monthly'}. Returns: {report_url: string, flagged_count: number, total_reviewed: number}. This is for generating final reports, not for investigating individual transactions."
    },
    {
      "name": "fetch_customer_kyc",
      "description": "Retrieves Know-Your-Customer profile data for a customer. Input: {customer_id: string}. Returns: {name, id_verified: boolean, risk_tier: string, documents_on_file: string[]}. Use only when you need customer identity data — not for transaction analysis."
    }
  ]
}
```

---

## Summary Checklist

- [ ] Every tool description explains **what**, **when**, **input format**, **output shape**, and **boundaries**
- [ ] No two tools share the same verb in their names
- [ ] Tool names do not echo keywords from the system prompt
- [ ] Generic tools are split into purpose-specific variants
- [ ] Descriptions explicitly state what the tool does NOT do
- [ ] Edge cases (empty results vs errors) are documented in the description

---

## Related Topics

- [[Domain 2 - Tool Design & MCP Integration]] — domain overview
- [[Tool Distribution and Tool Choice]] — controlling which tools an agent sees
- [[Structured Error Responses]] — what tools return when things go wrong
- [[MCP Server Configuration]] — where tool descriptions live in MCP setups
- [[Built-in Tools]] — Claude Code's pre-configured tool set
