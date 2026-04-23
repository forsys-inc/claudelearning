---
tags:
  - domain-4
  - task-4-5
  - batch-api
  - message-batches
  - cost-optimization
  - latency
  - custom-id
  - failure-handling
domain: "4 - Prompt Engineering & Structured Output"
task: "4.5"
---

# Batch Processing

> **Task 4.5** — Use the Message Batches API for cost-efficient, latency-tolerant workloads. Understand batch processing windows, `custom_id` correlation, failure handling, and prompt refinement strategies.

---

## Message Batches API Overview

The Message Batches API processes large volumes of requests at **50% cost savings** compared to standard API pricing. The tradeoff is latency: batches have a processing window of **up to 24 hours** with no guaranteed latency SLA — a batch may complete in minutes or may take the full 24 hours.

| Aspect | Standard API | Message Batches API |
|--------|-------------|---------------------|
| **Cost** | Full price | 50% discount |
| **Latency** | Seconds (predictable) | Up to 24 hours (no SLA) |
| **Use case** | Interactive, blocking workflows | Background, latency-tolerant workloads |
| **Tool use** | Full multi-turn tool calling | Single-turn only (no multi-turn tool calling) |

### Critical Limitation: No Multi-Turn Tool Calling

The Batch API does **NOT** support multi-turn tool calling within a single request. Each batch request is a single message exchange — you send messages in, you get a single response back. If your workflow requires the model to call tools and then continue reasoning based on tool results, you must use the standard API or orchestrate multiple batch submissions.

---

## Appropriate vs. Inappropriate Workloads

### Appropriate (Latency-Tolerant, Non-Blocking)

- **Overnight technical debt reports** — Run nightly, results reviewed next morning
- **Weekly code audit summaries** — Batch-analyze all PRs merged during the week
- **Nightly test generation** — Generate test cases for new code, reviewed before next sprint
- **Documentation generation** — Batch-process source files to generate API docs
- **Bulk data extraction** — Process thousands of documents for metadata extraction

### Inappropriate (Blocking, Latency-Sensitive)

- **Pre-merge CI checks** — Developers are blocked waiting for results; 24-hour SLA is unacceptable
- **Interactive code review** — Reviewers expect feedback within minutes
- **Real-time chat assistants** — User is waiting for a response
- **Deployment gates** — Cannot hold a deployment for up to 24 hours

---

## `custom_id` for Request-Response Correlation

Each request in a batch includes a `custom_id` field that you define. This ID is echoed back in the response, allowing you to correlate which response belongs to which input document. This is essential because batch responses are not guaranteed to return in submission order.

```python
# Each request has a unique custom_id you control
batch_requests = [
    {
        "custom_id": "doc-001",
        "params": {
            "model": "claude-sonnet-4-6-20250514",
            "max_tokens": 4096,
            "messages": [{"role": "user", "content": document_texts["doc-001"]}]
        }
    },
    {
        "custom_id": "doc-002",
        "params": {
            "model": "claude-sonnet-4-6-20250514",
            "max_tokens": 4096,
            "messages": [{"role": "user", "content": document_texts["doc-002"]}]
        }
    },
    # ... up to thousands of requests
]
```

When results come back, use `custom_id` to map responses to their source documents:

```python
for result in batch_results:
    doc_id = result["custom_id"]        # "doc-001", "doc-002", etc.
    status = result["result"]["type"]    # "succeeded" or "errored"
    if status == "succeeded":
        content = result["result"]["message"]["content"]
        save_analysis(doc_id, content)
    else:
        error = result["result"]["error"]
        record_failure(doc_id, error)
```

---

## Batch Submission Frequency and SLA Constraints

Since batch processing can take up to 24 hours, you must plan submission frequency based on your SLA constraints:

**Formula:** `submission_interval = SLA_hours - max_batch_processing_hours - buffer`

For example:
- SLA: results must be available within 30 hours of document receipt
- Max batch processing time: 24 hours
- Buffer for post-processing: 2 hours
- **Submission interval: every 4 hours** (30 - 24 - 2 = 4)

This ensures that even in the worst case (batch takes the full 24 hours), results are delivered within the SLA.

---

## Handling Batch Failures

Not all requests in a batch will succeed. Common failure modes:
- **Oversized input** — Document exceeds the model's context window
- **Content policy violations** — Input triggers safety filters
- **Rate-dependent failures** — Temporary capacity issues

The correct pattern is to **resubmit only failed documents by `custom_id`**, potentially with modifications to address the failure cause.

---

## Prompt Refinement Before Large Batches

Processing 5,000 documents with a flawed prompt wastes both money and time. Always refine your prompt on a small sample first:

1. Select a **representative sample** (10-20 documents covering edge cases)
2. Run the sample through the **standard API** (not batch — you need fast iteration)
3. Evaluate outputs against your quality criteria
4. Refine the prompt based on failure modes
5. Re-run the sample until quality is satisfactory
6. Submit the full batch with the refined prompt

---

## Examples

### Example 1 — Batch API for Overnight Technical Debt Reports vs. Real-Time Pre-Merge Checks

**Appropriate use — Overnight batch processing:**

```python
import anthropic
import json

client = anthropic.Anthropic()

# Prepare batch requests for all source files
batch_requests = []
for filepath, content in source_files.items():
    batch_requests.append({
        "custom_id": f"debt-{filepath.replace('/', '_')}",
        "params": {
            "model": "claude-sonnet-4-6-20250514",
            "max_tokens": 4096,
            "messages": [{
                "role": "user",
                "content": f"""Analyze the following source file for technical debt.
Report each instance with:
- Type (code duplication, dead code, missing error handling, outdated pattern, etc.)
- Location (function/class name and approximate line)
- Severity (high/medium/low)
- Suggested remediation

File: {filepath}
```
{content}
```"""
            }]
        }
    })

# Submit the batch — results arrive within 24 hours
batch = client.messages.batches.create(requests=batch_requests)
print(f"Batch submitted: {batch.id}")
# Results will be processed by a cron job that checks batch status
```

**Inappropriate use — Pre-merge CI check (do NOT use batch for this):**

```python
# WRONG: Developer is blocked waiting for this check to pass
# Batch processing could take up to 24 hours — unacceptable for pre-merge

# CORRECT: Use standard API for blocking workflows
response = client.messages.create(
    model="claude-sonnet-4-6-20250514",
    max_tokens=4096,
    messages=[{
        "role": "user",
        "content": f"Review this PR diff for critical issues:\n{diff_content}"
    }]
)
# Response arrives in seconds, CI pipeline can continue
```

---

### Example 2 — Calculating Batch Submission Frequency for a 30-Hour SLA

A compliance team must process incoming documents within 30 hours of receipt. Batch processing can take up to 24 hours.

```
SLA constraint:         30 hours from document receipt to results
Max batch processing:   24 hours (worst case)
Post-processing buffer:  2 hours (parsing, storage, notifications)
───────────────────────────────────────────────────────
Available window:       30 - 24 - 2 = 4 hours

→ Submit a new batch every 4 hours
```

Implementation:

```python
from datetime import datetime, timedelta

SUBMISSION_INTERVAL_HOURS = 4

def should_submit_batch(last_submission_time: datetime) -> bool:
    """Check if it's time to submit a new batch."""
    return datetime.now() - last_submission_time >= timedelta(hours=SUBMISSION_INTERVAL_HOURS)

def collect_and_submit():
    """Collect documents received since last batch and submit."""
    pending_docs = db.query(
        "SELECT id, content FROM documents WHERE batch_id IS NULL"
    )

    if not pending_docs:
        return

    batch_requests = [
        {
            "custom_id": f"compliance-{doc.id}",
            "params": {
                "model": "claude-sonnet-4-6-20250514",
                "max_tokens": 4096,
                "messages": [{
                    "role": "user",
                    "content": f"Analyze this document for regulatory compliance...\n{doc.content}"
                }]
            }
        }
        for doc in pending_docs
    ]

    batch = client.messages.batches.create(requests=batch_requests)

    # Mark documents as submitted
    for doc in pending_docs:
        db.execute(
            "UPDATE documents SET batch_id = ?, submitted_at = NOW() WHERE id = ?",
            (batch.id, doc.id)
        )
```

A document received at hour 0 is included in the next batch at hour 4 (worst case). That batch completes by hour 28 (4 + 24). Post-processing finishes by hour 30. SLA is met even in the worst case.

---

### Example 3 — Failure Handling: Resubmitting by `custom_id` with Chunking for Oversized Documents

When a batch completes, some requests may fail because documents exceeded the context window. The correct approach is to identify failed requests by `custom_id`, apply a fix (e.g., chunking), and resubmit only the failures.

```python
def handle_batch_results(batch_id: str):
    """Process batch results and resubmit failures with modifications."""
    results = client.messages.batches.results(batch_id)

    succeeded = []
    failed_oversized = []
    failed_other = []

    for result in results:
        doc_id = result["custom_id"]

        if result["result"]["type"] == "succeeded":
            succeeded.append(result)
        elif "context_length" in str(result["result"].get("error", "")):
            # Document was too large for the context window
            failed_oversized.append(doc_id)
        else:
            failed_other.append({
                "doc_id": doc_id,
                "error": result["result"]["error"]
            })

    print(f"Succeeded: {len(succeeded)}, Oversized: {len(failed_oversized)}, "
          f"Other failures: {len(failed_other)}")

    # Resubmit oversized documents with chunking
    if failed_oversized:
        resubmit_with_chunking(failed_oversized)


def resubmit_with_chunking(doc_ids: list[str]):
    """Split oversized documents into chunks and resubmit."""
    retry_requests = []

    for doc_id in doc_ids:
        original_content = get_document_content(doc_id)
        chunks = split_into_chunks(original_content, max_tokens=80000)

        for i, chunk in enumerate(chunks):
            retry_requests.append({
                "custom_id": f"{doc_id}_chunk_{i}",
                "params": {
                    "model": "claude-sonnet-4-6-20250514",
                    "max_tokens": 4096,
                    "messages": [{
                        "role": "user",
                        "content": f"""Analyze this document chunk for compliance issues.
This is chunk {i+1} of {len(chunks)} from document {doc_id}.

{chunk}"""
                    }]
                }
            })

    if retry_requests:
        retry_batch = client.messages.batches.create(requests=retry_requests)
        print(f"Retry batch submitted: {retry_batch.id} "
              f"({len(retry_requests)} chunked requests)")
```

Key points:
- **Identify failures by `custom_id`** — Do not resubmit the entire batch, only the failures.
- **Apply targeted fixes** — Oversized documents are chunked; other failure types might need different remediation.
- **Preserve traceability** — Chunked resubmissions use `custom_id` like `doc-001_chunk_0` so results can be reassembled.

---

### Example 4 — Prompt Refinement on a 10-Document Sample Before Processing 5,000 Documents

A team needs to extract structured metadata from 5,000 legal contracts. Before committing to a batch of 5,000 requests, they refine the prompt on a sample.

```python
# Step 1: Select representative sample (10 documents spanning edge cases)
sample_docs = [
    contracts["simple_nda"],           # Short, well-structured
    contracts["multi_party_agreement"], # Complex, multiple parties
    contracts["amendment_v3"],          # Amendment referencing prior versions
    contracts["foreign_language_mix"],  # Mix of English and Spanish clauses
    contracts["scan_with_ocr_errors"],  # OCR artifacts, missing characters
    contracts["very_long_master_svc"],  # Long document (edge of context window)
    contracts["template_unfilled"],     # Template with placeholder values
    contracts["handwritten_addendum"],  # OCR of handwritten notes
    contracts["terminated_contract"],   # Contract with termination clause exercised
    contracts["no_dates_specified"],    # Missing standard fields
]

# Step 2: Iterate on prompt using standard API (fast feedback loop)
prompt_v1 = """Extract the following fields from this contract:
- parties, effective_date, termination_date, governing_law, total_value
Return JSON."""

for doc in sample_docs:
    response = client.messages.create(
        model="claude-sonnet-4-6-20250514",
        max_tokens=2048,
        messages=[{"role": "user", "content": f"{prompt_v1}\n\n{doc['content']}"}]
    )
    print(f"{doc['name']}: {response.content[0].text[:200]}")

# Step 3: Evaluate and refine
# Found issues:
# - v1 hallucinated dates for the "no_dates" contract
# - v1 picked only the first 2 parties in multi-party agreement
# - v1 returned "N/A" inconsistently (sometimes null, sometimes "N/A")

prompt_v2 = """Extract the following fields from this contract.
Return a JSON object with these exact keys:

- "parties": array of ALL party names (there may be more than 2)
- "effective_date": ISO date string, or null if not specified (do NOT guess)
- "termination_date": ISO date string, or null if not specified (do NOT guess)
- "governing_law": jurisdiction string, or null if not specified
- "total_value": numeric value in USD, or null if not specified

IMPORTANT:
- Use null (not "N/A" or "unknown") for any field not explicitly stated
- Include ALL parties, not just the first two
- Dates must be explicitly stated in the document — do not infer from context
- If the document has OCR errors, extract the best interpretation but flag
  uncertainty by adding an "ocr_uncertain": true field

Contract:
"""

# Step 4: Re-run sample with refined prompt, verify quality

# Step 5: Once quality is satisfactory, submit full batch
batch_requests = [
    {
        "custom_id": f"contract-{contract_id}",
        "params": {
            "model": "claude-sonnet-4-6-20250514",
            "max_tokens": 2048,
            "messages": [{"role": "user", "content": f"{prompt_v2}\n\n{content}"}]
        }
    }
    for contract_id, content in all_5000_contracts.items()
]

batch = client.messages.batches.create(requests=batch_requests)
print(f"Full batch submitted: {batch.id} ({len(batch_requests)} contracts)")
```

**Why sample first?**
- 5,000 requests at the standard API rate would cost X dollars. At batch pricing (50% off), it is X/2. But if the prompt is wrong, you waste X/2 dollars and 24 hours.
- 10 sample requests via standard API cost negligible amounts and return in seconds, enabling rapid iteration.
- The sample revealed 3 issues that would have affected hundreds of contracts in the full batch.

---

## Key Takeaways for the Exam

1. **Message Batches API offers 50% cost savings** with a tradeoff of up to 24-hour processing time and no latency SLA.
2. **Batch API does NOT support multi-turn tool calling** — each request is a single message exchange.
3. **Use batch for non-blocking workloads** (overnight reports, weekly audits) — **never for blocking workflows** (pre-merge checks, interactive review).
4. **`custom_id` is essential** for correlating requests with responses and for targeted failure resubmission.
5. **Calculate submission frequency** as `SLA - max_processing_time - buffer` to guarantee SLA compliance.
6. **Resubmit only failed requests** by `custom_id`, applying targeted fixes (e.g., chunking oversized documents).
7. **Always refine prompts on a small sample** before committing to large batch processing — saves cost, time, and prevents systematic errors.

---

## Related Topics

- [[Domain 4 - Prompt Engineering & Structured Output]] — Parent domain overview
- [[JSON Schemas and Tool Use]] — Structuring batch request prompts for consistent output
- [[Validation and Retry Loops]] — Retry patterns applicable to batch failure handling
- [[CI-CD Integration]] — Batch processing as complement to real-time CI checks
- [[Context Window Management]] — Managing document size relative to context limits in batch requests
- [[Multi-Instance Review Architectures]] — Batch processing as input to multi-instance review pipelines
