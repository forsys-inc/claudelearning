---
tags:
  - domain-4
  - task-4-3
  - prompt-engineering
  - json-schema
  - tool-use
  - structured-output
  - tool-choice
domain: "4 - Prompt Engineering & Structured Output"
task: "4.3"
---

# JSON Schemas and Tool Use

> **Task 4.3** — Leverage `tool_use` with JSON schemas as the most reliable approach for guaranteed schema-compliant output, including `tool_choice` modes, nullable fields, enum patterns, and format normalization.

---

## Why Tool Use Is the Most Reliable Approach

When you need Claude to produce structured output that conforms to a specific schema, **`tool_use` with JSON schemas is the most reliable approach**. It eliminates an entire class of errors:

| Approach | JSON syntax errors | Missing required fields | Wrong field types | Semantic errors |
|----------|-------------------|------------------------|-------------------|-----------------|
| Prompt-based ("respond in JSON") | Common | Common | Common | Common |
| `tool_use` with JSON schema | **Eliminated** | **Eliminated** | **Eliminated** | Still possible |

The critical distinction: **strict JSON schemas eliminate syntax errors but NOT semantic errors**. The model will always produce valid JSON matching the schema, but the *values* in that JSON may still be wrong (e.g., extracting the wrong date from a document, or misclassifying a severity level).

---

## tool_choice Modes

The `tool_choice` parameter controls whether and how the model calls tools:

| Mode | Behavior | Use When |
|------|----------|----------|
| `"auto"` | Model may return text OR call a tool | The model should decide if a tool call is appropriate |
| `"any"` | Model **must** call a tool, but chooses which one | You have multiple tools and the model should pick the right one |
| `{"type": "tool", "name": "extract_metadata"}` | Model **must** call the specifically named tool | You know exactly which tool should be called |

### Key Exam Distinctions

- **`"auto"` can return plain text** — If the model decides no tool is needed, you get text back, not structured JSON. This is a common exam trap.
- **`"any"` forces a tool call** — The model must call *some* tool but gets to choose which one. Use this when you have multiple extraction tools for different document types.
- **Forced selection guarantees a specific schema** — When you force `extract_metadata`, every response will conform to that tool's schema. Use this in pipelines where downstream code depends on a specific schema.

---

## Schema Design Patterns

### Required vs. Optional Fields

Use `required` for fields that must always be present (even if null). Use optional fields only when the field's *existence* is conditional on context.

### Enum with "other" + Detail String

When a field has a known set of common values but might encounter novel ones, use an enum with an `"other"` value paired with a detail string:

```json
{
  "severity": {
    "type": "string",
    "enum": ["critical", "high", "medium", "low", "other"]
  },
  "severity_detail": {
    "type": "string",
    "description": "Required when severity is 'other'. Explain the severity level."
  }
}
```

This prevents the model from being forced to shoehorn novel cases into inappropriate categories while maintaining structured output for the common cases.

### Nullable Fields to Prevent Fabrication

When information may be absent from the source, use nullable types. This signals to the model that `null` is a valid and expected value, reducing the temptation to fabricate plausible-sounding data.

```json
{
  "author": {
    "type": ["string", "null"],
    "description": "Document author. Null if not specified in the source."
  }
}
```

Without nullable fields, the model is forced to produce a string — and will often fabricate one.

### Format Normalization Rules

Strict schemas guarantee structure but not consistency. Pair schemas with normalization rules:

```json
{
  "date": {
    "type": ["string", "null"],
    "description": "Date in ISO 8601 format (YYYY-MM-DD). Convert from any source format. Null if no date present."
  },
  "currency_amount": {
    "type": ["number", "null"],
    "description": "Amount in USD as a decimal number (e.g., 1234.56). Convert from any source format. Null if not specified."
  }
}
```

The schema ensures the field is a string or null; the description ensures the model normalizes to a consistent format.

---

## Examples

### Example 1 — Extraction Tool Definition with JSON Schema

```python
import anthropic

client = anthropic.Anthropic()

extract_invoice_tool = {
    "name": "extract_invoice",
    "description": (
        "Extract structured data from an invoice document. "
        "Use null for any field not present in the source document. "
        "Do not fabricate or infer values that are not explicitly stated."
    ),
    "input_schema": {
        "type": "object",
        "properties": {
            "invoice_number": {
                "type": ["string", "null"],
                "description": "Invoice ID/number as it appears in the document. Null if not present."
            },
            "vendor_name": {
                "type": "string",
                "description": "Name of the vendor/supplier issuing the invoice."
            },
            "invoice_date": {
                "type": ["string", "null"],
                "description": "Invoice date in ISO 8601 format (YYYY-MM-DD). Null if not present."
            },
            "due_date": {
                "type": ["string", "null"],
                "description": "Payment due date in ISO 8601 format (YYYY-MM-DD). Null if not present."
            },
            "total_amount": {
                "type": ["number", "null"],
                "description": "Total invoice amount as a decimal number. Null if not clearly stated."
            },
            "currency": {
                "type": "string",
                "enum": ["USD", "EUR", "GBP", "CAD", "AUD", "other"],
                "description": "Currency code. Use 'other' for currencies not in the enum."
            },
            "currency_detail": {
                "type": ["string", "null"],
                "description": "Required when currency is 'other'. The actual currency code."
            },
            "line_items": {
                "type": "array",
                "items": {
                    "type": "object",
                    "properties": {
                        "description": {"type": "string"},
                        "quantity": {"type": ["number", "null"]},
                        "unit_price": {"type": ["number", "null"]},
                        "line_total": {"type": ["number", "null"]}
                    },
                    "required": ["description"]
                }
            }
        },
        "required": [
            "invoice_number", "vendor_name", "invoice_date",
            "due_date", "total_amount", "currency", "line_items"
        ]
    }
}

# Force the model to call this specific tool
response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=4096,
    tools=[extract_invoice_tool],
    tool_choice={"type": "tool", "name": "extract_invoice"},
    messages=[{
        "role": "user",
        "content": f"Extract data from this invoice:\n\n{invoice_text}"
    }]
)

# Response is guaranteed to be a tool_use block with valid JSON
# matching the schema — no parsing errors possible
tool_block = next(b for b in response.content if b.type == "tool_use")
invoice_data = tool_block.input  # Already a dict matching the schema
```

Key design choices: nullable fields for optional invoice data, `enum` + `"other"` for currency, forced tool selection to guarantee the schema.

---

### Example 2 — tool_choice:"any" for Unknown Document Types

```python
# When you have multiple document types, let the model choose the right tool

tools = [
    {
        "name": "extract_invoice",
        "description": "Extract data from invoice documents (bills, purchase orders).",
        "input_schema": {
            "type": "object",
            "properties": {
                "vendor": {"type": "string"},
                "amount": {"type": "number"},
                "date": {"type": ["string", "null"]},
            },
            "required": ["vendor", "amount", "date"]
        }
    },
    {
        "name": "extract_contract",
        "description": "Extract data from contracts and agreements.",
        "input_schema": {
            "type": "object",
            "properties": {
                "parties": {"type": "array", "items": {"type": "string"}},
                "effective_date": {"type": ["string", "null"]},
                "termination_date": {"type": ["string", "null"]},
                "governing_law": {"type": ["string", "null"]},
            },
            "required": ["parties", "effective_date", "termination_date", "governing_law"]
        }
    },
    {
        "name": "extract_meeting_notes",
        "description": "Extract data from meeting notes and minutes.",
        "input_schema": {
            "type": "object",
            "properties": {
                "date": {"type": ["string", "null"]},
                "attendees": {"type": "array", "items": {"type": "string"}},
                "decisions": {"type": "array", "items": {"type": "string"}},
                "action_items": {"type": "array", "items": {"type": "string"}},
            },
            "required": ["date", "attendees", "decisions", "action_items"]
        }
    },
]

response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=4096,
    tools=tools,
    tool_choice={"type": "any"},  # Must call a tool, but picks which one
    messages=[{
        "role": "user",
        "content": f"Extract structured data from this document:\n\n{document_text}"
    }]
)

# The model will choose the appropriate tool based on document content
tool_block = next(b for b in response.content if b.type == "tool_use")
print(f"Document classified as: {tool_block.name}")
print(f"Extracted data: {tool_block.input}")
```

`tool_choice: {"type": "any"}` is the right choice here because:
- We do not know the document type in advance (so we cannot force a specific tool).
- We need structured output (so `"auto"` is risky — it might return text).
- The model's classification of document type is itself valuable metadata.

---

### Example 3 — Forced Tool Selection for extract_metadata Before Enrichment

```python
# In a pipeline, force extract_metadata first, then use its output for enrichment

# Step 1: Force metadata extraction with a specific schema
metadata_tool = {
    "name": "extract_metadata",
    "description": "Extract document metadata. This is the first step in the pipeline.",
    "input_schema": {
        "type": "object",
        "properties": {
            "title": {"type": ["string", "null"]},
            "author": {"type": ["string", "null"]},
            "date": {
                "type": ["string", "null"],
                "description": "Date in YYYY-MM-DD format. Null if absent."
            },
            "document_type": {
                "type": "string",
                "enum": ["invoice", "contract", "report", "memo", "other"]
            },
            "document_type_detail": {
                "type": ["string", "null"],
                "description": "Explanation when document_type is 'other'."
            },
            "language": {
                "type": "string",
                "enum": ["en", "es", "fr", "de", "other"]
            },
            "confidence": {
                "type": "number",
                "description": "Confidence in extraction accuracy, 0.0 to 1.0."
            }
        },
        "required": [
            "title", "author", "date", "document_type",
            "language", "confidence"
        ]
    }
}

# Force this exact tool — downstream pipeline code expects this schema
response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    tools=[metadata_tool],
    tool_choice={"type": "tool", "name": "extract_metadata"},
    messages=[{
        "role": "user",
        "content": f"Extract metadata:\n\n{document_text}"
    }]
)

metadata = next(b for b in response.content if b.type == "tool_use").input

# Step 2: Use metadata to route to the appropriate enrichment pipeline
if metadata["document_type"] == "invoice":
    enrich_invoice(document_text, metadata)
elif metadata["document_type"] == "contract":
    enrich_contract(document_text, metadata)
else:
    enrich_generic(document_text, metadata)
```

Forced tool selection is essential here because the enrichment step depends on the exact schema of `extract_metadata`. If the model returned text (via `"auto"`), the pipeline would break. The `confidence` field enables downstream routing decisions (e.g., flag low-confidence extractions for human review).

---

### Example 4 — Schema with Nullable Fields, Enum + "other" Pattern

```python
# Complete schema demonstrating nullable fields and enum + "other" pattern
# for a CI/CD pipeline that classifies test failures

classify_failure_tool = {
    "name": "classify_test_failure",
    "description": (
        "Classify a CI/CD test failure. Use null for fields where the information "
        "cannot be determined from the test output. Do not guess."
    ),
    "input_schema": {
        "type": "object",
        "properties": {
            "test_name": {
                "type": "string",
                "description": "Fully qualified test name (e.g., 'tests.auth.test_login.test_expired_token')."
            },
            "failure_category": {
                "type": "string",
                "enum": [
                    "assertion_error",
                    "timeout",
                    "connection_error",
                    "permission_denied",
                    "resource_exhaustion",
                    "flaky_test",
                    "environment_issue",
                    "other"
                ]
            },
            "failure_category_detail": {
                "type": ["string", "null"],
                "description": "Required when failure_category is 'other'. Describe the failure type."
            },
            "root_cause_file": {
                "type": ["string", "null"],
                "description": "File path most likely responsible. Null if not determinable from test output."
            },
            "root_cause_line": {
                "type": ["integer", "null"],
                "description": "Line number of root cause. Null if not determinable."
            },
            "is_flaky": {
                "type": ["boolean", "null"],
                "description": "Whether this appears to be a flaky test. Null if insufficient information."
            },
            "related_failures": {
                "type": "array",
                "items": {"type": "string"},
                "description": "Other test names that likely share the same root cause. Empty array if none."
            },
            "suggested_action": {
                "type": "string",
                "enum": ["fix_code", "fix_test", "retry", "investigate", "skip_known_flaky", "other"]
            },
            "suggested_action_detail": {
                "type": ["string", "null"],
                "description": "Explanation of suggested action. Required when suggested_action is 'other'."
            },
            "confidence": {
                "type": "number",
                "description": "Confidence in classification, 0.0 to 1.0."
            }
        },
        "required": [
            "test_name", "failure_category", "root_cause_file",
            "root_cause_line", "is_flaky", "related_failures",
            "suggested_action", "confidence"
        ]
    }
}

# Example output the model might produce:
# {
#   "test_name": "tests.payments.test_checkout.test_stripe_timeout",
#   "failure_category": "timeout",
#   "failure_category_detail": null,
#   "root_cause_file": "src/payments/stripe_client.py",
#   "root_cause_line": 47,
#   "is_flaky": true,
#   "related_failures": ["tests.payments.test_checkout.test_stripe_webhook"],
#   "suggested_action": "retry",
#   "suggested_action_detail": null,
#   "confidence": 0.75
# }
```

Design notes:
- **Nullable `root_cause_file` and `root_cause_line`**: The model cannot always determine the root cause from test output alone. Nullable prevents fabrication.
- **`is_flaky` is nullable boolean**: Three states — `true` (appears flaky), `false` (appears deterministic), `null` (insufficient info to judge).
- **Enum + "other" on both `failure_category` and `suggested_action`**: Covers common cases structurally while allowing novel situations.
- **`confidence` field**: Enables downstream routing — low-confidence classifications can be flagged for human review.

---

## Key Takeaways for the Exam

1. **`tool_use` with JSON schemas eliminates syntax errors** — but not semantic errors. The model will always produce valid JSON, but the values may still be wrong.
2. **`tool_choice: "auto"` may return text** instead of a tool call — this is a common exam trap.
3. **`tool_choice: "any"` forces a tool call** but lets the model choose which one — ideal for multi-tool routing.
4. **Forced tool selection** guarantees a specific schema — use in pipelines where downstream code depends on it.
5. **Nullable fields prevent fabrication** — signal that `null` is expected and acceptable.
6. **Enum + "other" + detail string** — structured output for common cases, flexible handling for novel ones.

---

## Related Topics

- [[Domain 4 - Prompt Engineering & Structured Output]] — Parent domain overview
- [[Few-Shot Prompting]] — Combine few-shot examples with schema enforcement for best results
- [[Validation and Retry Loops]] — Semantic validation catches errors that schema compliance cannot
- [[Tool Design Principles]] — Tool definition patterns and best practices
- [[Batch Processing]] — Tool use schemas in batch processing contexts
