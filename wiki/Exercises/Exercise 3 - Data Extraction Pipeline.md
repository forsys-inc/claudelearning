---
tags: [exercise, domain/4, domain/5, hands-on]
domains: [4, 5]
---

# Exercise 3: Build a Structured Data Extraction Pipeline

**Objective:** Practice designing JSON schemas, using `tool_use` for structured output, implementing validation-retry loops, and designing batch processing strategies.

**Domains reinforced:** [[Domain 4 - Prompt Engineering & Structured Output]], [[Domain 5 - Context Management & Reliability]]

---

## Step 1: Define an Extraction Tool with JSON Schema

Design a tool schema for extracting invoice data. Include required and optional fields, enums with "other" patterns, and nullable fields.

```python
import anthropic

client = anthropic.Anthropic()

extraction_tool = {
    "name": "extract_invoice",
    "description": "Extract structured data from an invoice document. Call this tool with the extracted fields. Use null for any field not present in the document — do NOT fabricate values.",
    "input_schema": {
        "type": "object",
        "properties": {
            "vendor_name": {
                "type": "string",
                "description": "Company or individual name on the invoice"
            },
            "invoice_number": {
                "type": "string",
                "description": "Unique invoice identifier"
            },
            "invoice_date": {
                "type": ["string", "null"],
                "description": "Date on the invoice in YYYY-MM-DD format. Null if not found."
            },
            "due_date": {
                "type": ["string", "null"],
                "description": "Payment due date in YYYY-MM-DD format. Null if not found."
            },
            "currency": {
                "type": "string",
                "enum": ["USD", "EUR", "GBP", "CAD", "AUD", "other"],
                "description": "Currency code"
            },
            "currency_detail": {
                "type": ["string", "null"],
                "description": "If currency is 'other', specify the actual currency code here"
            },
            "line_items": {
                "type": "array",
                "items": {
                    "type": "object",
                    "properties": {
                        "description": {"type": "string"},
                        "quantity": {"type": ["number", "null"]},
                        "unit_price": {"type": ["number", "null"]},
                        "amount": {"type": "number"}
                    },
                    "required": ["description", "amount"]
                }
            },
            "subtotal": {"type": ["number", "null"]},
            "tax_amount": {"type": ["number", "null"]},
            "stated_total": {
                "type": "number",
                "description": "Total as stated on the invoice"
            },
            "calculated_total": {
                "type": "number",
                "description": "Sum of line item amounts + tax. Used for validation."
            },
            "totals_match": {
                "type": "boolean",
                "description": "True if stated_total equals calculated_total"
            },
            "payment_method": {
                "type": ["string", "null"],
                "enum": ["bank_transfer", "credit_card", "check", "paypal", "other", null],
                "description": "Payment method if specified"
            },
            "confidence_scores": {
                "type": "object",
                "properties": {
                    "vendor_name": {"type": "number", "minimum": 0, "maximum": 1},
                    "line_items": {"type": "number", "minimum": 0, "maximum": 1},
                    "total": {"type": "number", "minimum": 0, "maximum": 1}
                },
                "description": "Field-level confidence scores (0-1)"
            }
        },
        "required": ["vendor_name", "invoice_number", "stated_total",
                      "calculated_total", "totals_match", "line_items",
                      "confidence_scores"]
    }
}
```

**Key design decisions:**
- `invoice_date` and `due_date` are nullable — prevents fabrication when not on document
- `currency` uses enum with "other" + `currency_detail` for extensibility
- `calculated_total` alongside `stated_total` enables self-correction validation
- `confidence_scores` enables downstream human review routing

### Call the extraction

```python
response = client.messages.create(
    model="claude-sonnet-4-6-20250514",
    max_tokens=4096,
    tools=[extraction_tool],
    tool_choice={"type": "tool", "name": "extract_invoice"},  # Force this specific tool
    messages=[{
        "role": "user",
        "content": f"Extract all invoice data from this document:\n\n{document_text}"
    }]
)

# Extract the structured result
tool_use_block = next(b for b in response.content if b.type == "tool_use")
invoice_data = tool_use_block.input
```

**Note:** `tool_choice` forces the specific tool, guaranteeing structured output. Using `"auto"` might return conversational text instead.

---

## Step 2: Implement Validation-Retry Loop

```python
from pydantic import BaseModel, field_validator, model_validator
from typing import Optional

class InvoiceExtraction(BaseModel):
    vendor_name: str
    invoice_number: str
    invoice_date: Optional[str] = None
    stated_total: float
    calculated_total: float
    totals_match: bool
    line_items: list
    confidence_scores: dict

    @model_validator(mode='after')
    def validate_totals(self):
        if abs(self.stated_total - self.calculated_total) > 0.01:
            if self.totals_match:
                raise ValueError(
                    f"totals_match is True but stated_total ({self.stated_total}) "
                    f"!= calculated_total ({self.calculated_total}). "
                    f"Re-examine the line items and recalculate."
                )
        return self

    @field_validator('invoice_date')
    @classmethod
    def validate_date_format(cls, v):
        if v is not None:
            import re
            if not re.match(r'\d{4}-\d{2}-\d{2}', v):
                raise ValueError(
                    f"Date '{v}' is not in YYYY-MM-DD format. "
                    f"Convert the date found in the document to YYYY-MM-DD."
                )
        return v


def extract_with_retry(document_text: str, max_retries: int = 2) -> dict:
    """Extract invoice data with validation-retry loop."""
    messages = [{
        "role": "user",
        "content": f"Extract all invoice data from this document:\n\n{document_text}"
    }]

    for attempt in range(max_retries + 1):
        response = client.messages.create(
            model="claude-sonnet-4-6-20250514",
            max_tokens=4096,
            tools=[extraction_tool],
            tool_choice={"type": "tool", "name": "extract_invoice"},
            messages=messages
        )

        tool_use_block = next(b for b in response.content if b.type == "tool_use")
        invoice_data = tool_use_block.input

        try:
            validated = InvoiceExtraction(**invoice_data)
            return validated.model_dump()
        except Exception as e:
            if attempt == max_retries:
                # Return with validation errors noted
                invoice_data["_validation_errors"] = str(e)
                return invoice_data

            # Append the failed attempt and error for retry
            messages.append({"role": "assistant", "content": response.content})
            messages.append({
                "role": "user",
                "content": (
                    f"The extraction had validation errors:\n{str(e)}\n\n"
                    f"Please re-examine the document and correct the extraction. "
                    f"The original document is above."
                )
            })
            print(f"Retry {attempt + 1}: {e}")

    return invoice_data
```

**When retries work vs don't:**
- **Works:** Format mismatches (date in wrong format), calculation errors (line items don't sum), structural errors
- **Doesn't work:** Information genuinely absent from document — retrying won't create data that isn't there

---

## Step 3: Add Few-Shot Examples for Varied Formats

```python
few_shot_prompt = """Extract invoice data from documents. Here are examples of handling varied formats:

EXAMPLE 1 — Standard table format:
Document: "Invoice #2024-001 | Acme Corp | Date: March 15, 2024
Item          Qty   Price   Total
Widget A       10   $5.00   $50.00
Widget B        5  $12.00   $60.00
Subtotal: $110.00 | Tax (8%): $8.80 | Total: $118.80"

Extraction: vendor_name="Acme Corp", invoice_number="2024-001",
invoice_date="2024-03-15", line_items=[{description: "Widget A", quantity: 10,
unit_price: 5.00, amount: 50.00}, ...], stated_total=118.80,
calculated_total=118.80, totals_match=true

EXAMPLE 2 — Narrative format (no table):
Document: "Please find attached our invoice (ref: INV-887) for consulting
services provided in Q1. We provided 40 hours of strategy consulting at
$150/hr ($6,000) and 20 hours of implementation support at $200/hr ($4,000).
Total amount due: $10,000."

Extraction: vendor_name=null (not stated — do NOT fabricate),
invoice_number="INV-887", line_items=[{description: "Strategy consulting",
quantity: 40, unit_price: 150.00, amount: 6000.00}, ...],
stated_total=10000.00, calculated_total=10000.00, totals_match=true
NOTE: vendor_name is null because document doesn't identify the vendor.

EXAMPLE 3 — Handwritten/informal with inconsistencies:
Document: "Bob's Repairs — Job #445
Fixed bathroom sink: $85
Replaced kitchen faucet + parts: $220
I make it $310 total (includes $5 disposal fee)"

Extraction: vendor_name="Bob's Repairs", invoice_number="445",
line_items=[{description: "Fixed bathroom sink", amount: 85.00},
{description: "Replaced kitchen faucet + parts", amount: 220.00},
{description: "Disposal fee", amount: 5.00}],
stated_total=310.00, calculated_total=310.00, totals_match=true
NOTE: quantity/unit_price are null for items without explicit breakdown.

Now extract from this document:
{document_text}
"""
```

---

## Step 4: Design Batch Processing Strategy

```python
import json
import time

def create_batch_requests(documents: list[dict]) -> list[dict]:
    """Create batch requests for document extraction."""
    requests = []
    for doc in documents:
        requests.append({
            "custom_id": doc["id"],  # e.g., "invoice-001"
            "params": {
                "model": "claude-sonnet-4-6-20250514",
                "max_tokens": 4096,
                "tools": [extraction_tool],
                "tool_choice": {"type": "tool", "name": "extract_invoice"},
                "messages": [{
                    "role": "user",
                    "content": f"{few_shot_prompt.replace('{document_text}', doc['text'])}"
                }]
            }
        })
    return requests


def process_batch(documents: list[dict]) -> dict:
    """Submit batch, poll for completion, handle failures."""
    requests = create_batch_requests(documents)

    # Submit batch
    batch = client.batches.create(requests=requests)
    print(f"Batch {batch.id} submitted with {len(requests)} requests")

    # Poll for completion (up to 24 hours)
    while batch.processing_status != "ended":
        time.sleep(60)  # Check every minute
        batch = client.batches.retrieve(batch.id)

    # Process results
    results = {}
    failures = []
    for result in client.batches.results(batch.id):
        if result.result.type == "succeeded":
            tool_block = next(
                b for b in result.result.message.content if b.type == "tool_use"
            )
            results[result.custom_id] = tool_block.input
        else:
            failures.append({
                "custom_id": result.custom_id,
                "error": result.result.error
            })

    return {"results": results, "failures": failures}


def handle_failures(failures: list[dict], original_docs: dict) -> list[dict]:
    """Resubmit failed documents with modifications."""
    retry_docs = []
    for failure in failures:
        doc = original_docs[failure["custom_id"]]
        if "context_length" in str(failure.get("error", "")):
            # Chunk oversized document
            chunks = chunk_document(doc["text"], max_chars=50000)
            for i, chunk in enumerate(chunks):
                retry_docs.append({
                    "id": f"{failure['custom_id']}-chunk-{i}",
                    "text": chunk
                })
        else:
            # Retry with simplified prompt
            retry_docs.append({
                "id": failure["custom_id"],
                "text": doc["text"]
            })
    return retry_docs
```

**SLA calculation:**
- Batch processing: up to 24 hours
- If SLA requires results within 30 hours → submit every 4 hours (leaving 6-hour buffer)
- If SLA requires results within 48 hours → submit every 24 hours

**Cost comparison:**
- 100 invoices via synchronous API: ~$X
- 100 invoices via Batch API: ~$X * 0.5 (50% savings)

---

## Step 5: Implement Human Review Routing

```python
def route_for_review(extraction: dict, doc_type: str) -> dict:
    """Route extractions to human review based on confidence and document type."""
    confidence = extraction.get("confidence_scores", {})
    routing = {"needs_review": False, "reasons": []}

    # Low confidence on any critical field
    for field, threshold in [("vendor_name", 0.85), ("line_items", 0.80), ("total", 0.90)]:
        score = confidence.get(field, 0)
        if score < threshold:
            routing["needs_review"] = True
            routing["reasons"].append(
                f"{field} confidence {score:.2f} below threshold {threshold}"
            )

    # Total mismatch
    if not extraction.get("totals_match", True):
        routing["needs_review"] = True
        routing["reasons"].append("Stated total does not match calculated total")

    # Known high-error document types
    high_error_types = ["handwritten", "scanned_fax", "multi_page"]
    if doc_type in high_error_types:
        routing["needs_review"] = True
        routing["reasons"].append(f"Document type '{doc_type}' has elevated error rates")

    return routing


# Stratified sampling for ongoing quality measurement
def stratified_sample(extractions: list[dict], sample_rate: float = 0.05):
    """Sample high-confidence extractions for error rate measurement."""
    import random
    high_confidence = [e for e in extractions if not e["routing"]["needs_review"]]

    # Group by document type
    by_type = {}
    for e in high_confidence:
        doc_type = e.get("doc_type", "unknown")
        by_type.setdefault(doc_type, []).append(e)

    # Sample proportionally from each type
    samples = []
    for doc_type, docs in by_type.items():
        n = max(1, int(len(docs) * sample_rate))
        samples.extend(random.sample(docs, min(n, len(docs))))

    return samples
```

---

## Checklist

- [ ] Extraction tool with JSON schema (required, optional, nullable, enum+"other")
- [ ] Validation-retry loop with specific error feedback
- [ ] Few-shot examples for varied document formats
- [ ] Batch processing with custom_id and failure handling
- [ ] Human review routing with field-level confidence
- [ ] Stratified sampling for ongoing quality measurement

---

## Related Topics

- [[JSON Schemas and Tool Use]]
- [[Validation and Retry Loops]]
- [[Few-Shot Prompting]]
- [[Batch Processing]]
- [[Human Review Workflows]]
