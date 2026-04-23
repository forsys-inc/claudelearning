---
tags:
  - domain-5
  - task-5-3
  - error-handling
  - multi-agent
  - error-propagation
  - partial-results
domain: "5 - Context Management & Reliability"
task: "5.3"
---

# Error Propagation in Multi-Agent Systems

> **Task 5.3** — Propagate errors with structured context, distinguish access failures from valid empty results, and enable coordinators to proceed with partial results and coverage annotations.

---

## Structured Error Context

When a subagent or tool call fails, a simple error message like `"Error: search failed"` destroys the information the coordinator needs to make good decisions. Structured error returns should include:

- **Failure type** — Was it a timeout, authentication failure, rate limit, malformed query, or a valid query that returned no results?
- **Attempted query** — What was the agent trying to do? This lets the coordinator retry with different parameters or assign to a different subagent.
- **Partial results** — Did the operation partially succeed before failing? A database query that returned 3 of 5 expected results is more useful than zero.
- **Alternatives** — Can the same information be obtained from a different source or with a different query strategy?

---

## Access Failures vs. Valid Empty Results

This distinction is one of the most commonly tested concepts in the exam. Two scenarios that look identical in a generic error handler:

| Scenario | What happened | Correct interpretation |
|----------|--------------|----------------------|
| **Access failure** | Database timed out after 30s | We don't know if records exist — the search was incomplete |
| **Valid empty result** | Query executed successfully, returned 0 rows | We have confirmed evidence that no matching records exist |

The coordinator's behavior should differ dramatically:

- **Access failure**: Report that coverage is incomplete, suggest retry, or route to an alternative data source.
- **Valid empty result**: Treat as a confirmed finding — "No matching records" is a real answer, not an error.

---

## Generic Error Statuses Hide Valuable Context

When a tool returns `{"status": "error", "message": "search unavailable"}`, the coordinator cannot distinguish between:

- The search API being entirely down (retry later)
- The search API rejecting a malformed query (fix the query)
- The search API rate-limiting this request (wait and retry)
- A network timeout (might work on retry)
- The search index being temporarily stale (results may be incomplete)

Each of these requires a different recovery strategy, but the generic error status collapses them all into one undifferentiated failure.

---

## Anti-Patterns

### 1. Silently suppressing errors
A subagent encounters an error and returns an empty result instead of reporting the failure. The coordinator proceeds as if the subagent found nothing, producing a synthesis with invisible gaps.

### 2. Terminating entire workflows on single failures
A coordinator receives an error from one of four subagents and aborts the entire research task, discarding the three successful results. This is especially wasteful when the failed subagent was researching a supplementary topic.

---

## Subagent Error Handling Strategy

- **Transient errors** (timeouts, rate limits): The subagent should retry locally with backoff. Only propagate to the coordinator if retries are exhausted.
- **Permanent errors** (authentication failure, invalid query): Propagate immediately — retrying will not help.
- **Partial success**: Return whatever results were obtained, clearly annotated with what is missing.

The key principle: **subagents handle what they can locally; they propagate only what the coordinator needs to know about.**

---

## Coverage Annotations in Synthesis Output

When a coordinator synthesizes results from multiple subagents, the final output should include explicit coverage annotations so the consumer knows what was and was not investigated.

---

## Examples

### Example 1 — Structured Error Return from Subagent

```python
# Subagent schema for returning results — success or failure.
# The same structure handles both cases, preventing silent suppression.

def research_subagent_result(
    topic: str,
    status: str,  # "success" | "partial" | "error"
    findings: str | None = None,
    sources: list[str] | None = None,
    error_detail: dict | None = None,
) -> dict:
    """Build a structured result from a research subagent."""
    result = {
        "topic": topic,
        "status": status,
        "timestamp": "2026-04-22T10:30:00Z",
    }
    if status in ("success", "partial"):
        result["findings"] = findings
        result["sources"] = sources or []
    if status in ("partial", "error"):
        result["error_detail"] = error_detail
    return result

# Subagent returning a structured error:
error_result = research_subagent_result(
    topic="Patent filings 2024-2026",
    status="error",
    error_detail={
        "failure_type": "timeout",
        "attempted_query": "USPTO full-text search: assignee='Acme Corp' AND date>=2024",
        "partial_results": None,
        "retry_attempted": True,
        "retries_exhausted": True,
        "alternatives": [
            "Try Google Patents API as fallback",
            "Narrow date range to 2025-2026 to reduce query scope",
        ],
    },
)

# Subagent returning partial success:
partial_result = research_subagent_result(
    topic="Competitor pricing analysis",
    status="partial",
    findings="Retrieved pricing for 3 of 5 competitors: Acme ($299), "
             "Beta ($315), Gamma ($289). Delta and Epsilon APIs timed out.",
    sources=["acme.com/pricing", "beta.io/plans", "gamma.co/pricing"],
    error_detail={
        "failure_type": "partial_timeout",
        "attempted_query": "Pricing pages for [Acme, Beta, Gamma, Delta, Epsilon]",
        "partial_results": "3/5 competitors retrieved",
        "alternatives": ["Retry Delta and Epsilon separately with longer timeout"],
    },
)
```

The coordinator can now make informed decisions: retry the patent search with a narrower query, use the 3-of-5 pricing results with an explicit coverage annotation, or escalate the gaps.

---

### Example 2 — Distinguishing Access Failure from Valid Empty Result

```python
def search_database(query: str, timeout_seconds: int = 30) -> dict:
    """Execute a database search and return a structured result that
    distinguishes access failures from valid empty results."""
    try:
        results = db.execute(query, timeout=timeout_seconds)

        if len(results) == 0:
            # VALID EMPTY RESULT — the query succeeded, no records match
            return {
                "status": "success",
                "result_type": "empty",
                "message": "Query executed successfully. No matching records found.",
                "query": query,
                "records": [],
                "is_authoritative": True,  # We KNOW there are no matches
            }
        else:
            return {
                "status": "success",
                "result_type": "data",
                "records": results,
                "record_count": len(results),
                "is_authoritative": True,
            }

    except TimeoutError:
        # ACCESS FAILURE — we don't know if records exist
        return {
            "status": "error",
            "result_type": "access_failure",
            "message": "Database query timed out after 30s.",
            "query": query,
            "records": [],
            "is_authoritative": False,  # We do NOT know if matches exist
            "failure_type": "timeout",
            "suggestion": "Retry with narrower date range or during off-peak hours.",
        }

    except ConnectionError as e:
        return {
            "status": "error",
            "result_type": "access_failure",
            "message": f"Database connection failed: {e}",
            "query": query,
            "records": [],
            "is_authoritative": False,
            "failure_type": "connection_error",
            "suggestion": "Check database connectivity and credentials.",
        }

# The coordinator uses `is_authoritative` to decide how to report:
result = search_database("SELECT * FROM claims WHERE policy_id = 'POL-2026-001'")

if result["is_authoritative"] and result["result_type"] == "empty":
    # Safe to say: "No claims found for this policy."
    pass
elif not result["is_authoritative"]:
    # Must say: "Unable to confirm whether claims exist — search was incomplete."
    pass
```

The `is_authoritative` flag is the key distinction. A valid empty result is authoritative (we searched and confirmed nothing is there). An access failure is non-authoritative (we could not complete the search).

---

### Example 3 — Coordinator Proceeding with Partial Results and Coverage Gap Annotations

```python
def synthesize_research(subagent_results: list[dict]) -> dict:
    """Coordinator synthesizes results from multiple subagents,
    annotating coverage gaps instead of silently omitting failed topics."""

    successful = [r for r in subagent_results if r["status"] == "success"]
    partial = [r for r in subagent_results if r["status"] == "partial"]
    failed = [r for r in subagent_results if r["status"] == "error"]

    # Build synthesis from available results
    synthesis_sections = []
    for result in successful + partial:
        synthesis_sections.append({
            "topic": result["topic"],
            "findings": result["findings"],
            "coverage": "complete" if result["status"] == "success" else "partial",
            "sources": result.get("sources", []),
        })

    # Build explicit coverage gap annotations
    coverage_gaps = []
    for result in failed:
        coverage_gaps.append({
            "topic": result["topic"],
            "reason": result["error_detail"]["failure_type"],
            "impact": f"No data available for {result['topic']}. "
                      f"Synthesis may be incomplete in this area.",
            "suggested_action": result["error_detail"].get("alternatives", []),
        })
    for result in partial:
        coverage_gaps.append({
            "topic": result["topic"],
            "reason": result["error_detail"]["failure_type"],
            "impact": result["error_detail"]["partial_results"],
        })

    return {
        "synthesis": synthesis_sections,
        "coverage_gaps": coverage_gaps,
        "total_topics": len(subagent_results),
        "fully_covered": len(successful),
        "partially_covered": len(partial),
        "not_covered": len(failed),
        "overall_confidence": (
            "high" if len(failed) == 0 and len(partial) == 0
            else "medium" if len(failed) == 0
            else "low"
        ),
    }

# Example output included in the final report:
# {
#   "synthesis": [...findings...],
#   "coverage_gaps": [
#     {
#       "topic": "Patent filings 2024-2026",
#       "reason": "timeout",
#       "impact": "No data available for Patent filings. Synthesis may be incomplete.",
#       "suggested_action": ["Try Google Patents API as fallback"]
#     }
#   ],
#   "total_topics": 4,
#   "fully_covered": 2,
#   "partially_covered": 1,
#   "not_covered": 1,
#   "overall_confidence": "low"
# }
```

The consumer of this synthesis knows exactly which topics were fully researched, which are incomplete, and which are missing entirely — instead of receiving a confident-sounding report with invisible holes.

---

### Example 4 — Anti-Pattern: Generic "Search Unavailable" Hiding Context

```python
# BAD — Generic error that hides all useful context
def search_tool_bad(query: str) -> str:
    try:
        results = external_api.search(query)
        return json.dumps(results)
    except Exception:
        return json.dumps({"status": "error", "message": "Search unavailable"})
        # What went wrong? Timeout? Auth failure? Rate limit? Bad query?
        # The coordinator has no idea and cannot take appropriate action.

# The coordinator sees this and can only say:
# "I wasn't able to search for that information."
# No retry strategy, no fallback, no partial results.

# GOOD — Structured error with actionable context
def search_tool_good(query: str) -> str:
    try:
        results = external_api.search(query, timeout=15)
        if not results:
            return json.dumps({
                "status": "success",
                "result_type": "empty",
                "message": "Search completed — no results found.",
                "query": query,
                "is_authoritative": True,
            })
        return json.dumps({
            "status": "success",
            "result_type": "data",
            "results": results,
            "query": query,
        })
    except TimeoutError:
        return json.dumps({
            "status": "error",
            "failure_type": "timeout",
            "message": "Search API timed out after 15 seconds.",
            "query": query,
            "is_authoritative": False,
            "suggestion": "Retry with simpler query or increased timeout.",
        })
    except RateLimitError as e:
        return json.dumps({
            "status": "error",
            "failure_type": "rate_limit",
            "message": f"Rate limit exceeded. Retry after {e.retry_after}s.",
            "query": query,
            "is_authoritative": False,
            "suggestion": f"Wait {e.retry_after} seconds and retry.",
        })
    except AuthenticationError:
        return json.dumps({
            "status": "error",
            "failure_type": "auth_failure",
            "message": "API authentication failed. Check credentials.",
            "query": query,
            "is_authoritative": False,
            "suggestion": "Verify API key is valid and not expired.",
        })
```

The good version enables the coordinator (or the agentic loop) to take differentiated action: wait and retry on rate limits, fix credentials on auth failure, simplify query on timeout. The bad version collapses all these into "search unavailable."

---

## Key Takeaways for the Exam

1. **Access failure ≠ valid empty result** — A timeout means "we don't know"; zero rows from a successful query means "confirmed: nothing there."
2. **Structured error context enables recovery** — Include failure type, attempted query, partial results, and alternatives in every error return.
3. **Never silently suppress errors** — Silent suppression produces confident-sounding syntheses with invisible gaps.
4. **Never abort entire workflows on single failures** — Proceed with partial results and annotate coverage gaps explicitly.
5. **Subagents retry transient errors locally** — Only propagate to the coordinator when retries are exhausted or the error is permanent.

---

## Related Topics

- [[Domain 5 - Context Management & Reliability]] — Parent domain overview
- [[Multi-Agent Orchestration]] — The coordinator-subagent patterns that make error propagation critical
- [[Context Window Management]] — Structured subagent outputs that carry error metadata
- [[Large Codebase Exploration]] — Crash recovery patterns that depend on structured state
- [[Tool Design Principles]] — Designing tool schemas that distinguish error types
