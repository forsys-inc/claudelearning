---
tags:
  - domain-1
  - task-1-2
  - multi-agent
  - hub-and-spoke
  - orchestration
  - coordinator
domain: "1 - Agentic Architecture & Orchestration"
task: "1.2"
---

# Multi-Agent Orchestration

> **Task 1.2** — Design hub-and-spoke multi-agent systems with isolated subagent context, coordinator-driven task decomposition, and iterative refinement.

---

## Hub-and-Spoke Architecture

In multi-agent systems built on Claude, the standard architecture is **hub-and-spoke**: a single **coordinator** agent manages all communication with **subagents**. Subagents never communicate directly with each other.

```
                    ┌──────────┐
                    │  User    │
                    └────┬─────┘
                         │
                    ┌────▼─────┐
                    │Coordinator│
                    │  (Hub)    │
                    └─┬──┬──┬──┘
                      │  │  │
              ┌───────┘  │  └───────┐
              ▼          ▼          ▼
         ┌────────┐ ┌────────┐ ┌────────┐
         │Subagent│ │Subagent│ │Subagent│
         │   A    │ │   B    │ │   C    │
         └────────┘ └────────┘ └────────┘
```

### Why Hub-and-Spoke?

- **No context bleed** — Each subagent has its own isolated conversation. Subagent A cannot see what subagent B produced unless the coordinator explicitly passes it.
- **Coordinator maintains the big picture** — Only the coordinator holds the overall goal, progress, and aggregated results.
- **Simpler debugging** — All inter-agent communication flows through one point.

---

## Subagent Context Isolation

This is a critical exam concept: **subagents do NOT inherit the coordinator's conversation history.** Each subagent starts with only what the coordinator explicitly provides in the subagent's prompt.

Implications:
- The coordinator must inject all relevant context into every subagent call.
- If subagent B needs results from subagent A, the coordinator must pass those results explicitly.
- Subagents cannot "remember" previous invocations unless the coordinator re-provides that information.

---

## Coordinator Responsibilities

### 1. Task Decomposition
The coordinator analyzes the user's request and breaks it into discrete subtasks suitable for individual subagents.

### 2. Delegation
The coordinator selects which subagent (or subagent type) handles each subtask, passing appropriate context and tool access.

### 3. Result Aggregation
As subagents return results, the coordinator synthesizes them into a coherent response — resolving conflicts, filling gaps, and ensuring consistency.

### 4. Dynamic Subagent Selection
Rather than hard-coding which subagent handles what, the coordinator chooses subagents based on the specific requirements of the current task. This enables handling of novel or hybrid queries.

---

## Risks of Overly Narrow Task Decomposition

Splitting a task into too many small subtasks creates problems:

- **Lost connections** — Important relationships between subtopics are missed because each subagent only sees its narrow slice.
- **Redundant work** — Multiple subagents may cover overlapping ground without coordination.
- **Shallow results** — Each subagent does surface-level work because the subtask is too small to warrant deep investigation.
- **Coordination overhead** — The coordinator spends more tokens managing subagents than the subagents spend on actual work.

The exam tests your ability to find the right granularity: broad enough that each subagent can produce meaningful, self-contained analysis, but narrow enough that the task is tractable.

---

## Iterative Refinement Loops

The coordinator does not always accept subagent results on the first pass. An iterative refinement loop follows this pattern:

1. **Decompose and delegate** — Send subtasks to subagents.
2. **Aggregate and evaluate** — Combine results and check for gaps, contradictions, or insufficient depth.
3. **Re-delegate if needed** — Spawn new subagents (or re-invoke existing patterns) to address gaps.
4. **Finalize** — Return the refined synthesis when quality criteria are met.

---

## Examples

### Example 1 — Coordinator Analyzing Query and Selecting Subagents Dynamically

```python
# The coordinator receives: "Research the environmental and economic impact
# of lithium mining in South America"

# Rather than hard-coding subagent assignments, the coordinator dynamically
# determines what expertise is needed:

coordinator_system_prompt = """You are a research coordinator. When given a
research question:
1. Analyze the query to identify distinct research dimensions.
2. For each dimension, spawn a subagent with the appropriate focus.
3. Aggregate results into a unified synthesis.

Available subagent specializations:
- environmental_analyst: ecological impacts, pollution, habitat effects
- economic_analyst: market dynamics, trade, employment, GDP effects
- regional_specialist: political context, local communities, regulations
- technical_specialist: mining processes, technology, supply chains

Select subagents based on what the query actually requires — not every
specialization is needed for every query."""

# The coordinator might decide this query needs:
# 1. environmental_analyst — for ecological impact of lithium mining
# 2. economic_analyst — for economic impact on South American economies
# 3. regional_specialist — for political/community context in Chile, Argentina, Bolivia

# It would NOT spawn technical_specialist because the query is about
# impact, not mining technology details.
```

---

### Example 2 — Partitioning Research Scope to Minimize Duplication

```python
# BAD partitioning — overlapping scopes cause redundancy:
# Subagent A: "Research lithium mining impact in Chile"
# Subagent B: "Research lithium mining impact in Argentina"
# Subagent C: "Research environmental impact of lithium mining in South America"
# → Subagent C overlaps heavily with A and B

# GOOD partitioning — clear boundaries with minimal overlap:
subagent_tasks = {
    "environmental": {
        "prompt": """Analyze the environmental impact of lithium mining across
South America. Cover: water table depletion in salt flats, chemical runoff
from extraction, habitat disruption in the Lithium Triangle.
Do NOT cover economic or political dimensions — another analyst handles those.""",
        "tools": ["web_search", "read_document"],
    },
    "economic": {
        "prompt": """Analyze the economic impact of lithium mining on South
American economies. Cover: GDP contribution, employment, export revenue,
foreign investment, comparison to other mining sectors.
Do NOT cover environmental effects — another analyst handles those.""",
        "tools": ["web_search", "read_document", "data_analysis"],
    },
    "policy": {
        "prompt": """Analyze the regulatory and community dimensions of lithium
mining in South America. Cover: nationalization efforts (Bolivia, Mexico),
indigenous community impacts, water rights disputes, evolving regulations.
Do NOT cover raw economic data or ecological science — other analysts handle those.""",
        "tools": ["web_search", "read_document"],
    },
}

# Each subagent is told explicitly what NOT to cover, preventing overlap.
# The coordinator will synthesize cross-cutting themes in the aggregation step.
```

---

### Example 3 — Iterative Refinement Loop

```python
def coordinate_research(query: str) -> str:
    """Coordinator runs subagents, evaluates synthesis, and refines if needed."""

    # Phase 1: Decompose and delegate
    subtasks = decompose_query(query)
    results = {}
    for task_id, task_config in subtasks.items():
        results[task_id] = run_subagent(
            prompt=task_config["prompt"],
            tools=task_config["tools"],
        )

    # Phase 2: Synthesize and evaluate
    synthesis = aggregate_results(results)

    # Phase 3: Iterative refinement — coordinator evaluates its own synthesis
    evaluation_prompt = f"""Review this research synthesis for completeness:

{synthesis}

Original query: {query}

Identify:
1. Topics mentioned in the query but missing from the synthesis
2. Claims that lack supporting evidence
3. Contradictions between sections
4. Areas where depth is insufficient

If gaps exist, specify exactly what additional research is needed.
If the synthesis is adequate, respond with SYNTHESIS_COMPLETE."""

    evaluation = call_claude(evaluation_prompt)

    if "SYNTHESIS_COMPLETE" not in evaluation:
        # Phase 4: Targeted follow-up for identified gaps
        gap_results = run_gap_filling_subagents(evaluation)
        synthesis = merge_results(synthesis, gap_results)

    return synthesis
```

The coordinator does not blindly accept initial results. It evaluates the synthesis against the original query and spawns follow-up subagents only for specific gaps — not a full re-run.

---

### Example 4 — Overly Narrow Decomposition (Anti-Pattern)

```python
# User query: "How does remote work affect employee productivity?"

# BAD — overly narrow decomposition:
subtasks_bad = [
    "Research how remote work affects morning productivity",
    "Research how remote work affects afternoon productivity",
    "Research how remote work affects productivity on Mondays",
    "Research how remote work affects productivity on Fridays",
    "Research how remote work affects productivity for engineers",
    "Research how remote work affects productivity for salespeople",
    "Research how remote work affects productivity for managers",
]
# Problems:
# - 7 subagents producing shallow, repetitive results
# - Cross-cutting themes (isolation, flexibility, commute time) are fragmented
# - Coordination overhead exceeds the value of parallelization
# - Key factors like "tooling quality" or "management style" are missed entirely
#   because the decomposition is organized by time/role, not by causal factor

# GOOD — appropriately scoped decomposition:
subtasks_good = [
    {
        "id": "quantitative",
        "prompt": """Analyze quantitative research on remote work and
productivity. Cover: output metrics, hours worked, task completion rates.
Include meta-analyses where available. Note methodological limitations.""",
    },
    {
        "id": "qualitative",
        "prompt": """Analyze qualitative factors affecting remote worker
productivity. Cover: communication patterns, isolation effects, work-life
boundary challenges, tooling and infrastructure, management practices.""",
    },
    {
        "id": "contextual",
        "prompt": """Analyze how context moderates the remote work-productivity
relationship. Cover: industry type, role seniority, team distribution,
organizational culture, individual preferences. Identify where remote work
helps vs. hurts productivity.""",
    },
]
# 3 subagents with meaningful, non-overlapping scopes that cover the topic.
```

---

## Key Takeaways for the Exam

1. **Hub-and-spoke is the standard architecture** — the coordinator is the only agent that communicates with all subagents.
2. **Subagent context is isolated** — they receive only what the coordinator explicitly provides.
3. **Dynamic subagent selection** beats hard-coded routing for handling diverse queries.
4. **Decomposition granularity matters** — too narrow fragments reasoning, too broad overwhelms individual subagents.
5. **Iterative refinement** means the coordinator evaluates and re-delegates, not just fires-and-forgets.

---

## Related Topics

- [[Domain 1 - Agentic Architecture & Orchestration]] — Parent domain overview
- [[Agentic Loop Lifecycle]] — The loop that each subagent runs internally
- [[Subagent Configuration and Context Passing]] — How to configure and invoke subagents
- [[Task Decomposition Strategies]] — Fixed vs. adaptive decomposition approaches
- [[Context Window Management]] — Managing token budgets across coordinator and subagents
