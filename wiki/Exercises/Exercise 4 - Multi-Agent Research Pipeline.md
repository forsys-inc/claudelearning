---
tags: [exercise, domain/1, domain/2, domain/5, hands-on]
domains: [1, 2, 5]
---

# Exercise 4: Design and Debug a Multi-Agent Research Pipeline

**Objective:** Practice orchestrating subagents, managing context passing, implementing error propagation, and handling synthesis with provenance tracking.

**Domains reinforced:** [[Domain 1 - Agentic Architecture & Orchestration]], [[Domain 2 - Tool Design & MCP Integration]], [[Domain 5 - Context Management & Reliability]]

---

## Step 1: Build a Coordinator Agent with Subagents

Design a coordinator that delegates to web search and document analysis subagents using the Claude Agent SDK.

```python
from claude_agent_sdk import Agent, AgentDefinition, Tool

# Define subagent types
web_search_agent = AgentDefinition(
    name="web_search",
    description="Searches the web for information on a given topic. Returns structured findings with source URLs, publication dates, and relevance scores.",
    system_prompt="""You are a web research specialist. For each query:
1. Search for relevant sources
2. Extract key claims with supporting evidence
3. Return structured findings with source attribution

Output format for each finding:
- claim: The key fact or statistic
- evidence: Direct quote or excerpt supporting the claim
- source_url: URL of the source
- source_name: Publication or website name
- publication_date: Date published (YYYY-MM-DD if available, null if not)
- relevance_score: 0-1 how relevant to the research question
""",
    allowed_tools=["web_search", "fetch_url"]
)

document_analysis_agent = AgentDefinition(
    name="document_analysis",
    description="Analyzes documents and papers in depth. Extracts structured findings with methodology context and claim-source mappings.",
    system_prompt="""You are a document analysis specialist. For each document:
1. Identify key claims, statistics, and conclusions
2. Note methodology and limitations
3. Return structured findings preserving source attribution

Output format for each finding:
- claim: The key fact, statistic, or conclusion
- evidence: Exact quote from the document
- source_document: Document name/title
- page_or_section: Where in the document this was found
- methodology: How the data was gathered (if applicable)
- publication_date: When the document was published
- confidence: How well-supported this claim is within the document
""",
    allowed_tools=["read_document", "search_document"]
)

synthesis_agent = AgentDefinition(
    name="synthesis",
    description="Combines findings from multiple research agents into a coherent, cited report. Preserves source attribution and flags conflicting information.",
    system_prompt="""You are a research synthesis specialist. You receive structured findings from web search and document analysis agents.

Rules:
1. PRESERVE all source attribution — every claim in your output must link to its source
2. When sources CONFLICT, include both values with attribution — do NOT pick one
3. Note temporal differences — statistics from different years are not contradictions
4. Distinguish well-established findings from contested ones
5. Flag coverage gaps — areas where research was incomplete

Output structure:
- Executive summary (key findings)
- Detailed findings by topic (with inline citations)
- Conflicting information (with both sources)
- Coverage gaps and limitations
- Source bibliography
""",
    allowed_tools=["verify_fact"]  # Scoped tool for simple lookups only
)

# Coordinator agent
coordinator = AgentDefinition(
    name="coordinator",
    description="Orchestrates research by decomposing topics, delegating to specialist subagents, evaluating coverage, and triggering refinement.",
    system_prompt="""You are a research coordinator. Your job:
1. Analyze the research question and identify ALL relevant subtopics
2. Delegate to specialist subagents with clear, specific assignments
3. Evaluate results for coverage gaps
4. Trigger refinement rounds if needed
5. Send complete findings to synthesis

IMPORTANT: Decompose broadly — if the topic is "impact of AI on creative industries",
cover ALL creative industries (visual arts, music, writing, film, gaming, etc.),
not just the first few that come to mind.

Spawn subagents in PARALLEL when their tasks are independent.
Pass COMPLETE findings from prior agents to downstream agents.
""",
    allowed_tools=["Task", "web_search", "read_document"]  # Task tool enables spawning
)
```

**Key points:**
- `allowedTools` includes `"Task"` for the coordinator (required for spawning subagents)
- Each subagent has restricted tools matching its role
- Synthesis agent gets only `verify_fact` (scoped), not full web search tools

---

## Step 2: Implement Parallel Subagent Execution

The coordinator should emit multiple Task tool calls in a **single response** for parallel execution.

```python
# Coordinator's system prompt should guide it to spawn parallel tasks:
coordinator_prompt_addition = """
When researching a topic with independent subtopics, spawn multiple subagents
in a SINGLE response. For example, to research "AI in creative industries":

In ONE response, emit these tool calls:
- Task: web_search agent — "AI impact on visual arts and design"
- Task: web_search agent — "AI impact on music composition and production"
- Task: web_search agent — "AI impact on creative writing and journalism"
- Task: document_analysis agent — [relevant academic papers]

Do NOT wait for one to complete before spawning the next.
"""

# Measuring latency improvement
import time

# Sequential (anti-pattern)
def sequential_research(topics):
    results = []
    start = time.time()
    for topic in topics:
        result = spawn_subagent("web_search", topic)  # Blocks until complete
        results.append(result)
    sequential_time = time.time() - start
    return results, sequential_time

# Parallel (correct pattern) — handled by emitting multiple Task calls
# The Agent SDK executes them concurrently
# Typical improvement: 3 parallel agents finish in ~1x time vs ~3x sequential
```

---

## Step 3: Design Structured Output with Provenance

Ensure content is separated from metadata so attribution survives synthesis.

```python
# Structured finding format (what subagents return)
finding_schema = {
    "type": "object",
    "properties": {
        "claim": {
            "type": "string",
            "description": "The factual claim or statistic"
        },
        "evidence": {
            "type": "string",
            "description": "Direct quote or excerpt supporting the claim"
        },
        "source": {
            "type": "object",
            "properties": {
                "url": {"type": ["string", "null"]},
                "document_name": {"type": ["string", "null"]},
                "author": {"type": ["string", "null"]},
                "publication_date": {
                    "type": ["string", "null"],
                    "description": "YYYY-MM-DD format"
                },
                "page_or_section": {"type": ["string", "null"]}
            }
        },
        "methodology": {
            "type": ["string", "null"],
            "description": "How data was gathered (surveys, analysis, etc.)"
        },
        "confidence": {
            "type": "number",
            "minimum": 0,
            "maximum": 1
        },
        "topic_area": {
            "type": "string",
            "description": "Which subtopic this finding relates to"
        }
    },
    "required": ["claim", "evidence", "source", "topic_area"]
}

# What the coordinator passes to synthesis agent:
synthesis_input = """
## Research Findings to Synthesize

### From Web Search Agent (Visual Arts):
{web_search_findings_json}

### From Web Search Agent (Music):
{web_search_music_json}

### From Document Analysis Agent:
{document_analysis_findings_json}

## Instructions:
- Combine findings into a coherent report
- PRESERVE all source attribution (URLs, document names, dates)
- Flag any conflicting statistics with BOTH sources
- Note coverage gaps (topics we couldn't find data on)
- Include publication dates to contextualize temporal differences
"""
```

---

## Step 4: Implement Error Propagation

Simulate a subagent timeout and verify the coordinator receives actionable error context.

```python
# Subagent error response structure
def handle_subagent_error(agent_name, query, error, partial_results=None):
    """Structure error information for coordinator decision-making."""
    return {
        "status": "error",
        "agent": agent_name,
        "failure_type": classify_error(error),  # "timeout", "rate_limit", "not_found"
        "attempted_query": query,
        "partial_results": partial_results or [],
        "alternatives": suggest_alternatives(error, query),
        "is_retryable": is_retryable(error)
    }

def classify_error(error):
    if "timeout" in str(error).lower():
        return "timeout"
    elif "rate_limit" in str(error).lower():
        return "rate_limit"
    elif "not_found" in str(error).lower():
        return "not_found"
    return "unknown"

def suggest_alternatives(error, query):
    if classify_error(error) == "timeout":
        return [
            "Retry with narrower query scope",
            "Split into multiple smaller queries",
            "Try alternative search terms"
        ]
    elif classify_error(error) == "not_found":
        return [
            "Try broader search terms",
            "Search in different source databases"
        ]
    return ["Retry with same parameters"]

# Coordinator handling errors
coordinator_error_handling = """
When a subagent returns an error:
1. If retryable (timeout, rate_limit): retry with modified query OR proceed with partial results
2. If not retryable (not_found): proceed with available data, note the gap
3. NEVER silently drop errors — annotate the final output with what couldn't be researched

Example annotation in final report:
"Note: Web search for 'AI in theater and performing arts' timed out.
Partial results (2 of estimated 5 sources) are included.
Coverage of performing arts may be incomplete."
"""

# Anti-patterns to avoid:
# BAD: return {"status": "success", "results": []}  # Suppresses error as empty success
# BAD: raise Exception("Search failed")  # Crashes entire pipeline
# BAD: return {"status": "error", "message": "search unavailable"}  # No context for recovery
```

---

## Step 5: Test with Conflicting Source Data

Create a test case where two credible sources report different statistics.

```python
# Simulated conflicting findings
web_finding_1 = {
    "claim": "AI tools are used by 72% of professional graphic designers",
    "evidence": "Our survey of 2,500 designers found that 72% now use AI tools weekly",
    "source": {
        "url": "https://designsurvey2025.example.com",
        "document_name": "Annual Design Industry Survey 2025",
        "publication_date": "2025-01-15"
    },
    "methodology": "Online survey of 2,500 professional graphic designers in US/EU",
    "topic_area": "AI in visual arts"
}

web_finding_2 = {
    "claim": "Only 45% of designers regularly use AI tools",
    "evidence": "Our research shows 45% of designers have integrated AI into regular workflow",
    "source": {
        "url": "https://creativepro.example.com/report",
        "document_name": "Creative Professional Tools Report 2025",
        "publication_date": "2025-03-01"
    },
    "methodology": "Interviews with 800 designers across all specializations (not just graphic)",
    "topic_area": "AI in visual arts"
}

# CORRECT synthesis output (preserves both, explains difference):
correct_synthesis = """
## AI Adoption Among Designers

AI tool adoption rates vary significantly depending on measurement scope:
- **72%** of *graphic designers* specifically use AI tools weekly
  (Design Industry Survey 2025, n=2,500 graphic designers, Jan 2025)
- **45%** of *designers across all specializations* have integrated AI
  (Creative Professional Tools Report, n=800 all designers, Mar 2025)

The difference likely reflects that graphic designers have higher AI adoption
than designers in other fields (UX, industrial, fashion). The surveys also
differ in scope: "use weekly" vs "integrated into regular workflow."

Sources conflict on the exact adoption rate — both are included for context.
"""

# WRONG synthesis output (arbitrarily picks one):
wrong_synthesis = """
## AI Adoption Among Designers
72% of designers use AI tools weekly. [WRONG: dropped conflicting source]
"""
```

**Verification checklist:**
- [ ] Both values preserved with source attribution
- [ ] Methodological differences noted (sample scope, question wording)
- [ ] Temporal context included (publication dates)
- [ ] Report distinguishes well-established from contested findings

---

## Checklist

- [ ] Coordinator agent with `Task` in `allowedTools`
- [ ] Subagents with role-appropriate tool restrictions
- [ ] Parallel subagent spawning (multiple Task calls in single response)
- [ ] Structured findings with claim-source separation
- [ ] Error propagation with failure type, partial results, alternatives
- [ ] Coordinator proceeds with partial results and annotates gaps
- [ ] Conflicting data preserved with source attribution, not arbitrarily selected

---

## Related Topics

- [[Multi-Agent Orchestration]]
- [[Subagent Configuration and Context Passing]]
- [[Error Propagation in Multi-Agent Systems]]
- [[Information Provenance and Uncertainty]]
- [[Tool Distribution and Tool Choice]]
