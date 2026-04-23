---
tags:
  - domain-4
  - task-4-4
  - prompt-engineering
  - validation
  - retry-loops
  - self-correction
  - error-feedback
domain: "4 - Prompt Engineering & Structured Output"
task: "4.4"
---

# Validation and Retry Loops

> **Task 4.4** — Design retry-with-error-feedback loops, distinguish when retries are effective vs. futile, and build self-correction flows with semantic validation.

---

## Retry-with-Error-Feedback

The most effective retry pattern is **appending specific validation errors** to the next request. Rather than simply re-prompting with the same input, you send the original document, the failed extraction, and a precise description of what went wrong. This gives the model targeted information to correct its output.

### When Retries Are Effective vs. Ineffective

This is a critical distinction for the exam:

| Scenario | Retry Effective? | Why |
|----------|-----------------|-----|
| Model output has wrong JSON structure | **No** — use `tool_use` instead | Schema enforcement eliminates structural errors entirely |
| Model misread a date format | **Yes** | The information exists in the source; the model just parsed it wrong |
| Model fabricated a value not in the source | **Partially** | Retry with explicit "this field is not in the source" may help |
| Source document lacks the requested field | **No** | No amount of retrying will extract information that does not exist |
| Model misclassified a severity level | **Yes** | Feedback on the correct criteria can recalibrate |
| Model returned garbled output due to truncation | **Yes** | Retry with higher `max_tokens` or smaller input |

**Key principle**: Retries are effective for **format and interpretation errors** (the information exists but was processed incorrectly). Retries are **ineffective when the information is absent from the source** — the model cannot extract what is not there, and retrying will only produce different fabrications.

---

## Semantic Validation vs. Schema Syntax Errors

When using `tool_use` with JSON schemas, **syntax errors are already eliminated**. The model will always produce valid JSON conforming to the schema. Therefore, your validation layer should focus entirely on **semantic errors**:

| Validation Layer | Handles | Method |
|-----------------|---------|--------|
| JSON schema (via tool_use) | Syntax, structure, types, required fields | Automatic — no code needed |
| Semantic validation (your code) | Wrong values, inconsistencies, fabrication | Custom rules you implement |

Semantic validations to implement:
- **Cross-field consistency** — Does the total match the sum of line items?
- **Date logic** — Is the due date after the invoice date?
- **Referential integrity** — Do referenced IDs exist in the source?
- **Value ranges** — Is the confidence between 0 and 1?
- **Null appropriateness** — Is a field null when the source clearly contains the value?

---

## Feedback Loop Design: detected_pattern Field

For review systems that produce findings (code review, document review), include a `detected_pattern` field in the output schema. This field captures the specific code or text pattern that triggered the finding, enabling downstream **dismissal analysis**.

When developers dismiss a finding, you can analyze the `detected_pattern` to understand *why* it was a false positive. Over time, this builds a dataset for refining review criteria and disabling unreliable categories.

---

## Self-Correction Flows

Self-correction fields are schema fields that the model populates to flag its own internal inconsistencies. These work because the model can detect contradictions *during generation* even when it cannot prevent them.

Common self-correction patterns:
- **`calculated_total` vs. `stated_total`** — Model extracts both the total it calculated from line items and the total stated in the document. If they differ, downstream code flags a discrepancy.
- **`conflict_detected` boolean** — Model sets this to `true` when it notices contradictory information in the source.
- **`confidence` score** — Model self-reports confidence, enabling routing of low-confidence extractions to human review.

---

## Examples

### Example 1 — Follow-Up Request with Original Doc + Failed Extraction + Errors

```python
import anthropic
import json

client = anthropic.Anthropic()

def extract_with_retry(document: str, tool: dict, max_retries: int = 2) -> dict:
    """Extract structured data with validation and error-feedback retries."""

    messages = [{
        "role": "user",
        "content": f"Extract data from this document:\n\n{document}"
    }]

    for attempt in range(max_retries + 1):
        response = client.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=4096,
            tools=[tool],
            tool_choice={"type": "tool", "name": tool["name"]},
            messages=messages,
        )

        tool_block = next(b for b in response.content if b.type == "tool_use")
        extracted = tool_block.input

        # Run semantic validation
        errors = validate_extraction(extracted, document)

        if not errors:
            return extracted  # Valid extraction — done

        if attempt < max_retries:
            # Append the failed attempt and specific errors as feedback
            messages.append({
                "role": "assistant",
                "content": response.content
            })
            messages.append({
                "role": "user",
                "content": (
                    f"The extraction has {len(errors)} validation error(s). "
                    f"Please re-extract, correcting these specific issues:\n\n"
                    + "\n".join(f"- {e}" for e in errors)
                    + "\n\nOriginal document for reference:\n\n" + document
                )
            })
        else:
            # Max retries exhausted — return with error metadata
            extracted["_validation_errors"] = errors
            extracted["_retry_count"] = attempt
            return extracted

    return extracted


def validate_extraction(data: dict, source: str) -> list[str]:
    """Semantic validation — schema compliance is already guaranteed by tool_use."""
    errors = []

    # Cross-field consistency: line items should sum to total
    if data.get("line_items") and data.get("total_amount") is not None:
        calculated = sum(
            item.get("line_total", 0) or 0
            for item in data["line_items"]
        )
        if abs(calculated - data["total_amount"]) > 0.01:
            errors.append(
                f"Line items sum to {calculated:.2f} but total_amount is "
                f"{data['total_amount']:.2f}. Verify which is correct from "
                f"the source document."
            )

    # Date logic: due_date should be after invoice_date
    if data.get("invoice_date") and data.get("due_date"):
        if data["due_date"] < data["invoice_date"]:
            errors.append(
                f"due_date ({data['due_date']}) is before invoice_date "
                f"({data['invoice_date']}). Check if dates were swapped."
            )

    # Fabrication check: verify vendor name appears in source
    if data.get("vendor_name") and data["vendor_name"] not in source:
        errors.append(
            f"vendor_name '{data['vendor_name']}' does not appear in the "
            f"source document. Extract the name exactly as it appears."
        )

    return errors
```

Key design decisions:
- The retry message includes the **original document**, the **failed extraction** (via conversation history), and **specific errors** with guidance on what to check.
- Validation focuses on semantics (cross-field consistency, date logic, fabrication) because schema syntax is already guaranteed.
- After max retries, the result is returned with error metadata rather than silently passed downstream.

---

### Example 2 — Identifying When Retries Will Be Ineffective

```python
def should_retry(errors: list[str], extraction: dict, source: str) -> bool:
    """Determine if a retry is likely to improve the extraction.

    Retries are effective for interpretation errors (data exists but
    was misread). Retries are ineffective when data is absent from source.
    """
    retryable_errors = []
    non_retryable_errors = []

    for error in errors:
        if is_missing_data_error(error, extraction, source):
            # Data not in source — retry will just produce different fabrication
            non_retryable_errors.append(error)
        else:
            # Interpretation/format error — retry with feedback can fix this
            retryable_errors.append(error)

    if not retryable_errors:
        # All errors are non-retryable — retry would be wasteful
        return False

    return True


def is_missing_data_error(error: str, extraction: dict, source: str) -> bool:
    """Check if an error is due to absent data rather than misinterpretation."""

    # If the field is null and we're flagging it, check if the data actually
    # exists in the source
    missing_indicators = [
        "not found in source",
        "not present in document",
        "field is null but expected",
    ]

    # If validation flagged a null field, verify the data truly isn't there
    # This prevents retrying when the model correctly returned null
    for indicator in missing_indicators:
        if indicator in error.lower():
            return True

    return False


# Usage in the retry loop:
errors = validate_extraction(extracted, document)
if errors and should_retry(errors, extracted, document):
    # Retry with error feedback — interpretation errors can be corrected
    pass
elif errors:
    # Non-retryable errors — flag for human review instead of retrying
    flag_for_human_review(extracted, errors, document)
else:
    # No errors — proceed
    pass
```

This prevents wasting API calls retrying extractions where the source document genuinely lacks the requested information. The model returned `null` correctly; retrying will only produce fabricated values.

---

### Example 3 — detected_pattern Field for Analyzing False Positives

```python
# Tool schema that includes detected_pattern for dismissal analysis
code_review_tool = {
    "name": "report_finding",
    "description": "Report a code review finding with the detected pattern for tracking.",
    "input_schema": {
        "type": "object",
        "properties": {
            "file_path": {"type": "string"},
            "line_number": {"type": "integer"},
            "category": {
                "type": "string",
                "enum": [
                    "sql_injection", "null_reference", "resource_leak",
                    "error_handling", "concurrency", "other"
                ]
            },
            "severity": {
                "type": "string",
                "enum": ["critical", "high", "medium", "low"]
            },
            "description": {
                "type": "string",
                "description": "What the issue is and its runtime consequence."
            },
            "detected_pattern": {
                "type": "string",
                "description": (
                    "The specific code pattern that triggered this finding. "
                    "Copy the relevant code snippet exactly as it appears. "
                    "This field is used for false-positive analysis — be precise."
                )
            },
            "suggested_fix": {"type": "string"},
        },
        "required": [
            "file_path", "line_number", "category", "severity",
            "description", "detected_pattern", "suggested_fix"
        ]
    }
}

# When a developer dismisses a finding, analyze the detected_pattern:
def analyze_dismissals(dismissed_findings: list[dict]) -> dict:
    """Analyze dismissed findings to identify categories for refinement."""
    pattern_groups = {}

    for finding in dismissed_findings:
        category = finding["category"]
        pattern = finding["detected_pattern"]

        if category not in pattern_groups:
            pattern_groups[category] = []
        pattern_groups[category].append(pattern)

    # Report categories with high dismissal rates
    report = {}
    for category, patterns in pattern_groups.items():
        report[category] = {
            "dismissal_count": len(patterns),
            "common_patterns": find_common_patterns(patterns),
            "recommendation": (
                "Disable category" if len(patterns) > 10
                else "Refine criteria with these false-positive examples"
            )
        }

    return report

# Example output:
# {
#   "error_handling": {
#     "dismissal_count": 14,
#     "common_patterns": ["except KeyboardInterrupt: pass", "except ValueError: continue"],
#     "recommendation": "Disable category"
#   },
#   "null_reference": {
#     "dismissal_count": 3,
#     "common_patterns": ["obj.get('key').strip()"],
#     "recommendation": "Refine criteria with these false-positive examples"
#   }
# }
```

The `detected_pattern` field creates a feedback loop: dismissed findings reveal which patterns the model incorrectly flags, and these patterns are used to refine or disable categories. See [[Explicit Criteria for Precision]] for how to update criteria based on this data.

---

### Example 4 — Self-Correction with calculated_total/stated_total Discrepancy

```python
# Schema with self-correction fields
invoice_extraction_tool = {
    "name": "extract_invoice",
    "description": (
        "Extract invoice data with self-correction fields. "
        "Calculate the total from line items independently, then also "
        "extract the stated total. Flag any discrepancy."
    ),
    "input_schema": {
        "type": "object",
        "properties": {
            "vendor_name": {"type": "string"},
            "invoice_number": {"type": ["string", "null"]},
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
            },
            "stated_total": {
                "type": ["number", "null"],
                "description": (
                    "The total amount as explicitly stated in the invoice. "
                    "Extract the exact number printed on the document."
                )
            },
            "calculated_total": {
                "type": ["number", "null"],
                "description": (
                    "Your independent calculation: sum of all line_total values. "
                    "Calculate this yourself from the line items above."
                )
            },
            "totals_match": {
                "type": "boolean",
                "description": "True if stated_total equals calculated_total (within $0.01)."
            },
            "conflict_detected": {
                "type": "boolean",
                "description": (
                    "True if you noticed ANY contradictory information in the "
                    "document (e.g., dates that conflict, names that don't match, "
                    "line items that don't add up to the stated total)."
                )
            },
            "conflict_details": {
                "type": ["string", "null"],
                "description": "Description of the conflict, if conflict_detected is true."
            },
            "confidence": {
                "type": "number",
                "description": "Overall confidence in extraction accuracy, 0.0 to 1.0."
            }
        },
        "required": [
            "vendor_name", "line_items", "stated_total",
            "calculated_total", "totals_match", "conflict_detected",
            "confidence"
        ]
    }
}

# Downstream processing uses self-correction fields for routing
def process_extraction(result: dict) -> str:
    """Route extraction based on self-correction signals."""

    if not result["totals_match"]:
        # Discrepancy between stated and calculated totals
        return f"REVIEW_REQUIRED: Total mismatch — stated={result['stated_total']}, " \
               f"calculated={result['calculated_total']}"

    if result["conflict_detected"]:
        # Model found internal contradictions in the document
        return f"REVIEW_REQUIRED: {result['conflict_details']}"

    if result["confidence"] < 0.8:
        return "REVIEW_REQUIRED: Low confidence extraction"

    return "AUTO_APPROVED"

# Example model output:
# {
#   "vendor_name": "Acme Cloud Services",
#   "line_items": [
#     {"description": "Compute (m5.xlarge x 3)", "quantity": 3, "unit_price": 142.50, "line_total": 427.50},
#     {"description": "Storage (500GB S3)", "quantity": 1, "unit_price": 11.50, "line_total": 11.50},
#     {"description": "Data transfer (2TB)", "quantity": 1, "unit_price": 180.00, "line_total": 180.00}
#   ],
#   "stated_total": 629.00,
#   "calculated_total": 619.00,
#   "totals_match": false,                 ← Model caught the discrepancy
#   "conflict_detected": true,
#   "conflict_details": "Stated total ($629.00) exceeds sum of line items ($619.00) by $10.00. Possible missing line item or rounding issue.",
#   "confidence": 0.7
# }
```

Self-correction works because the model performs two independent extraction paths:
1. Extract the stated total from the document text.
2. Calculate the total from the extracted line items.

When these disagree, the model sets `totals_match: false` and `conflict_detected: true`, routing the extraction to human review. This catches both source document errors (incorrect totals printed on the invoice) and extraction errors (model misread a line item).

---

## Key Takeaways for the Exam

1. **Retry with specific error feedback** — Do not simply re-prompt. Append the failed extraction and precise validation errors.
2. **Retries are ineffective when data is absent** — No amount of retrying will extract information that is not in the source. Route to human review instead.
3. **Schema validation is already handled by tool_use** — Focus your validation code on semantic consistency, not JSON syntax.
4. **`detected_pattern` enables feedback loops** — Track what patterns trigger findings, and use dismissal data to refine criteria.
5. **Self-correction fields** (`calculated_total` vs. `stated_total`, `conflict_detected`) let the model flag its own uncertainty for downstream routing.

---

## Related Topics

- [[Domain 4 - Prompt Engineering & Structured Output]] — Parent domain overview
- [[JSON Schemas and Tool Use]] — Schema enforcement eliminates the need for syntax validation
- [[Explicit Criteria for Precision]] — Dismissal analysis feeds back into criteria refinement
- [[Multi-Instance Review Architectures]] — Independent review as an alternative to self-correction
- [[Batch Processing]] — Validation and retry patterns in batch contexts
