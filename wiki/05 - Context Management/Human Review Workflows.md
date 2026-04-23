---
tags:
  - domain-5
  - task-5-5
  - human-review
  - confidence-calibration
  - data-extraction
  - stratified-sampling
domain: "5 - Context Management & Reliability"
task: "5.5"
---

# Human Review Workflows

> **Task 5.5** — Design human-in-the-loop review systems with stratified sampling, field-level confidence calibration, and routing logic that catches accuracy gaps masked by aggregate metrics.

---

## Aggregate Accuracy Masks Poor Performance on Specific Types

The most dangerous metric in automated data extraction is **overall accuracy**. A system reporting 97% accuracy across all documents may be performing excellently on printed forms (99.5%) while catastrophically failing on handwritten documents (60%). If 95% of your documents are printed, the aggregate looks great — but every handwritten document is essentially unprocessed.

This masking effect applies across multiple dimensions:

- **Document type**: Printed vs. handwritten, structured forms vs. free-text letters, PDFs vs. scanned images
- **Field type**: Standard fields (name, date) vs. complex fields (multi-line addresses, monetary amounts with currency symbols)
- **Source quality**: High-resolution scans vs. photographed documents vs. faxed copies

---

## Stratified Random Sampling

To detect masked accuracy problems, sample documents for human review **stratified by the dimensions that matter**, not uniformly at random.

### Why uniform random sampling fails

If 95% of documents are printed invoices and 5% are handwritten receipts, a random sample of 100 documents will contain approximately 95 printed invoices and 5 handwritten receipts. You will be unable to draw statistically meaningful conclusions about handwritten receipt accuracy from 5 samples.

### Stratified approach

Sample a **fixed minimum** from each stratum (document type, source quality, field complexity) to ensure every category gets meaningful coverage.

---

## Field-Level Confidence Scores

Rather than a single document-level confidence score, assign confidence scores to individual extracted fields. This enables fine-grained routing:

- High-confidence fields proceed automatically
- Low-confidence fields are routed to human review
- The human reviewer sees only the specific fields that need attention, not the entire document

### Calibration requirement

Raw model confidence scores (logprobs, self-reported confidence) are not well-calibrated out of the box. They must be calibrated against a labeled validation set:

1. Extract fields from a labeled dataset where ground truth is known
2. Group extractions by confidence score ranges (0.9-1.0, 0.8-0.9, etc.)
3. Measure actual accuracy within each confidence range
4. If the 0.9-1.0 confidence bucket only has 85% actual accuracy, the scores need recalibration

---

## Validating by Document Type and Field Before Automating

Before moving any extraction category to fully automated processing:

1. Measure accuracy **per document type per field** (not aggregate)
2. Require a minimum sample size for each category
3. Set accuracy thresholds per field based on business impact (financial amounts need higher accuracy than descriptive text)
4. Only automate categories that meet thresholds; route the rest to human review

---

## Examples

### Example 1 — 97% Overall Accuracy Masking 60% Accuracy on Handwritten Documents

```python
# Extraction accuracy report — the aggregate looks great
extraction_results = {
    "overall_accuracy": 0.97,
    "total_documents_processed": 10000,
    "total_fields_extracted": 80000,
    "fields_correct": 77600,
}
# A manager sees 97% and approves full automation. But...

# Breakdown by document type reveals the problem:
accuracy_by_type = {
    "printed_invoice": {
        "document_count": 8500,
        "field_accuracy": 0.995,  # Excellent
        "sample_size": 200,
    },
    "typed_letter": {
        "document_count": 1000,
        "field_accuracy": 0.94,  # Good
        "sample_size": 100,
    },
    "handwritten_receipt": {
        "document_count": 400,
        "field_accuracy": 0.60,  # Catastrophic
        "sample_size": 80,
    },
    "faxed_form": {
        "document_count": 100,
        "field_accuracy": 0.72,  # Poor
        "sample_size": 50,
    },
}

# The aggregate is dominated by 8,500 printed invoices at 99.5%.
# 400 handwritten receipts at 60% accuracy = 40% error rate on those docs.
# Business impact: handwritten receipts are expense claims — errors mean
# incorrect reimbursements, compliance issues, and employee frustration.

# Correct decision matrix:
automation_decisions = {
    "printed_invoice": "automate",        # 99.5% exceeds 98% threshold
    "typed_letter": "automate",           # 94% exceeds 90% threshold
    "handwritten_receipt": "human_review", # 60% far below any threshold
    "faxed_form": "human_review",         # 72% below 90% threshold
}
```

---

### Example 2 — Stratified Sampling Across Document Types

```python
def create_stratified_sample(
    documents: list[dict],
    min_per_stratum: int = 30,
    total_budget: int = 200,
) -> list[dict]:
    """Create a stratified random sample ensuring every document type
    and quality level gets meaningful representation.

    Args:
        documents: All documents with metadata (type, quality, etc.)
        min_per_stratum: Minimum samples per stratum for statistical significance
        total_budget: Total number of documents to sample
    """
    from collections import defaultdict
    import random

    # Group by (document_type, source_quality) — the stratification dimensions
    strata = defaultdict(list)
    for doc in documents:
        key = (doc["document_type"], doc["source_quality"])
        strata[key].append(doc)

    sample = []
    remaining_budget = total_budget

    # Phase 1: Guarantee minimum samples per stratum
    for stratum_key, stratum_docs in strata.items():
        n = min(min_per_stratum, len(stratum_docs))
        stratum_sample = random.sample(stratum_docs, n)
        sample.extend(stratum_sample)
        remaining_budget -= n

    # Phase 2: Allocate remaining budget proportionally
    if remaining_budget > 0:
        already_sampled_ids = {doc["id"] for doc in sample}
        unsampled = [d for d in documents if d["id"] not in already_sampled_ids]
        additional = random.sample(unsampled, min(remaining_budget, len(unsampled)))
        sample.extend(additional)

    return sample

# Example strata and their sample sizes:
# ("printed_invoice", "high_res"):      30 guaranteed (of 7000 total)
# ("printed_invoice", "low_res"):       30 guaranteed (of 1500 total)
# ("handwritten_receipt", "high_res"):   30 guaranteed (of 200 total)
# ("handwritten_receipt", "photo"):      30 guaranteed (of 200 total)
# ("faxed_form", "low_res"):            30 guaranteed (of 100 total)
#
# Without stratification, a random 200-doc sample would contain:
#   ~140 printed_invoice/high_res, ~30 printed_invoice/low_res,
#   ~4 handwritten/high_res, ~4 handwritten/photo, ~2 faxed
# The handwritten and faxed strata would be statistically useless.
```

---

### Example 3 — Field-Level Confidence with Calibrated Thresholds

```python
# Raw extraction output with per-field confidence scores
extraction = {
    "document_id": "DOC-2026-4401",
    "document_type": "insurance_claim_form",
    "fields": {
        "claimant_name": {
            "value": "Margaret Thompson",
            "raw_confidence": 0.98,
            "calibrated_confidence": 0.96,  # Adjusted using validation set
        },
        "claim_date": {
            "value": "2026-03-15",
            "raw_confidence": 0.95,
            "calibrated_confidence": 0.93,
        },
        "claim_amount": {
            "value": "$4,287.50",
            "raw_confidence": 0.88,
            "calibrated_confidence": 0.72,  # Calibration reveals overconfidence
            # Raw model said 88% confident, but validation showed this
            # confidence range is only accurate 72% of the time for amounts
        },
        "policy_number": {
            "value": "POL-2026-??82",  # Partially illegible
            "raw_confidence": 0.45,
            "calibrated_confidence": 0.31,
        },
    },
}

# Calibration table (built from labeled validation set):
# Raw Confidence | Actual Accuracy | Calibrated Score
# 0.95 - 1.00   | 0.96            | 0.96
# 0.90 - 0.95   | 0.93            | 0.93
# 0.85 - 0.90   | 0.72 (!)        | 0.72  ← model overconfident in this range
# 0.80 - 0.85   | 0.68            | 0.68
# 0.40 - 0.50   | 0.31            | 0.31

# Thresholds vary by field type based on business impact:
CONFIDENCE_THRESHOLDS = {
    "claimant_name": 0.85,     # Names: moderate threshold
    "claim_date": 0.90,        # Dates: high threshold (affects deadlines)
    "claim_amount": 0.95,      # Money: very high threshold (financial impact)
    "policy_number": 0.90,     # Identifiers: high threshold (wrong policy = wrong claim)
}

def route_extraction(extraction: dict) -> dict:
    """Route each field to auto-accept or human review based on
    calibrated confidence vs. field-specific thresholds."""
    routing = {}
    for field_name, field_data in extraction["fields"].items():
        threshold = CONFIDENCE_THRESHOLDS.get(field_name, 0.90)
        calibrated = field_data["calibrated_confidence"]
        routing[field_name] = {
            "value": field_data["value"],
            "confidence": calibrated,
            "threshold": threshold,
            "action": "auto_accept" if calibrated >= threshold else "human_review",
        }
    return routing

# Result for this document:
# claimant_name: 0.96 >= 0.85 → auto_accept
# claim_date:    0.93 >= 0.90 → auto_accept
# claim_amount:  0.72 <  0.95 → human_review  ← flagged despite looking "88% confident"
# policy_number: 0.31 <  0.90 → human_review
```

The calibration step is critical. Without it, the claim amount (raw 0.88) would likely pass a naive 0.85 threshold — but calibration reveals that the 0.85-0.90 raw confidence range only achieves 72% actual accuracy for monetary amounts.

---

### Example 4 — Review Routing Based on Confidence and Document Type

```python
def route_document(extraction: dict, accuracy_registry: dict) -> dict:
    """Route a document for processing based on BOTH field-level confidence
    AND document-type-level accuracy from the accuracy registry.

    Even high-confidence extractions from poorly-performing document types
    get routed to human review.
    """
    doc_type = extraction["document_type"]
    doc_type_accuracy = accuracy_registry.get(doc_type, {})

    # Check if this document type is approved for automation at all
    if doc_type_accuracy.get("automation_approved") is not True:
        return {
            "action": "human_review",
            "reason": f"Document type '{doc_type}' not approved for automation. "
                      f"Current accuracy: {doc_type_accuracy.get('field_accuracy', 'unknown')}",
            "fields_to_review": "all",
        }

    # Document type is approved — route individual fields
    fields_for_review = []
    fields_auto_accepted = []

    for field_name, field_data in extraction["fields"].items():
        threshold = CONFIDENCE_THRESHOLDS.get(field_name, 0.90)
        calibrated = field_data["calibrated_confidence"]

        if calibrated >= threshold:
            fields_auto_accepted.append(field_name)
        else:
            fields_for_review.append({
                "field": field_name,
                "extracted_value": field_data["value"],
                "confidence": calibrated,
                "threshold": threshold,
            })

    if fields_for_review:
        return {
            "action": "partial_human_review",
            "reason": f"{len(fields_for_review)} field(s) below confidence threshold",
            "fields_to_review": fields_for_review,
            "fields_auto_accepted": fields_auto_accepted,
        }

    return {
        "action": "auto_accept",
        "reason": "All fields above confidence thresholds for approved document type",
    }

# Example: A handwritten receipt with high-confidence fields STILL gets
# routed to human review because the document type itself is not approved:
#
# route_document(
#   extraction={"document_type": "handwritten_receipt", "fields": {...}},
#   accuracy_registry={"handwritten_receipt": {"field_accuracy": 0.60, "automation_approved": False}}
# )
# → {"action": "human_review", "reason": "Document type 'handwritten_receipt' not approved..."}
```

This two-level routing (document type approval + field-level confidence) prevents the system from auto-accepting individual high-confidence fields from a document type that is fundamentally unreliable.

---

## Key Takeaways for the Exam

1. **Never trust aggregate accuracy alone** — always break down by document type, field type, and source quality.
2. **Stratified sampling ensures every category gets meaningful coverage** — uniform random sampling is dominated by the most common category.
3. **Calibrate confidence scores against labeled data** — raw model confidence is often overconfident in specific ranges.
4. **Route at two levels** — document type must be approved for automation AND individual fields must meet calibrated confidence thresholds.
5. **Field-specific thresholds reflect business impact** — monetary amounts need higher accuracy than descriptive text.

---

## Related Topics

- [[Domain 5 - Context Management & Reliability]] — Parent domain overview
- [[Escalation and Ambiguity Resolution]] — Routing to humans based on trigger conditions
- [[Information Provenance and Uncertainty]] — Confidence and source attribution in synthesized outputs
- [[Error Propagation in Multi-Agent Systems]] — Structured error context parallels confidence metadata
