---
tags:
  - domain-1
  - task-1-6
  - task-decomposition
  - prompt-chaining
  - adaptive-decomposition
  - pipelines
domain: "1 - Agentic Architecture & Orchestration"
task: "1.6"
---

# Task Decomposition Strategies

> **Task 1.6** — Choose between fixed sequential pipelines (prompt chaining) and dynamic adaptive decomposition based on the nature of the task.

---

## Two Approaches to Decomposition

| Approach | Structure | Best For |
|----------|-----------|----------|
| **Fixed Sequential Pipeline (Prompt Chaining)** | Pre-defined stages, each feeding into the next | Predictable, repeatable tasks with known structure |
| **Dynamic Adaptive Decomposition** | Agent decides at runtime how to break down the task | Open-ended investigations, novel problem spaces |

The exam requires you to recognize when each approach is appropriate and to avoid forcing one approach where the other fits better.

---

## Prompt Chaining (Fixed Pipelines)

Prompt chaining breaks a task into a fixed sequence of stages. Each stage has a specific role, receives the output of the previous stage, and produces structured output for the next stage.

```
Stage 1          Stage 2           Stage 3          Stage 4
┌──────────┐    ┌──────────────┐  ┌────────────┐  ┌───────────┐
│ Per-file  │───►│ Cross-file   │─►│ Priority   │─►│ Report    │
│ analysis  │    │ integration  │  │ ranking    │  │ generation│
└──────────┘    └──────────────┘  └────────────┘  └───────────┘
```

### Characteristics
- **Predictable**: You know exactly how many stages will run.
- **Debuggable**: Each stage's output can be inspected independently.
- **Repeatable**: Same input structure always produces same pipeline.
- **Rigid**: Cannot adapt if the task does not fit the pre-defined stages.

### When to Use
- Code reviews (analyze each file, then cross-reference)
- Document processing pipelines (extract, transform, validate, load)
- Multi-aspect evaluations where the aspects are known in advance

---

## Dynamic Adaptive Decomposition

The agent itself decides how to decompose the task at runtime based on what it discovers. The investigation plan evolves as the agent learns more.

### Characteristics
- **Flexible**: Adapts to the shape of the problem.
- **Exploratory**: Can follow unexpected threads.
- **Harder to predict**: Cannot guarantee how many steps or what they will be.
- **Risk of drift**: Agent may go down rabbit holes without guardrails.

### When to Use
- Bug investigation (you do not know where the bug is yet)
- Open-ended research (the dimensions emerge from initial findings)
- Legacy code exploration (structure is unknown until examined)

---

## Examples

### Example 1 — Prompt Chaining: Per-File Analysis Then Cross-File Integration

```python
def code_review_pipeline(changed_files: list[str]) -> dict:
    """Fixed sequential pipeline for code review.

    Stage 1: Analyze each file independently for local issues.
    Stage 2: Cross-file integration to find interaction bugs.
    Stage 3: Prioritize findings by severity.
    Stage 4: Generate formatted review report.
    """

    # Stage 1: Per-file analysis (can run in parallel)
    file_analyses = {}
    for filepath in changed_files:
        file_content = read_file(filepath)
        analysis = call_claude(
            system="You are a code reviewer. Analyze this single file for bugs, "
                   "security issues, and style problems. Return structured JSON.",
            prompt=f"Review this file:\n\nFile: {filepath}\n```\n{file_content}\n```",
        )
        file_analyses[filepath] = json.loads(analysis)

    # Stage 2: Cross-file integration (requires all Stage 1 outputs)
    cross_file_prompt = f"""You have individual file analyses from a code review.
Now identify cross-file issues: breaking interface changes, inconsistent error
handling patterns, missing updates in dependent files.

Individual file analyses:
{json.dumps(file_analyses, indent=2)}

Changed files: {changed_files}

Return only NEW cross-file issues not already caught in individual reviews."""

    cross_file_issues = json.loads(call_claude(
        system="You are a senior code reviewer specializing in cross-cutting concerns.",
        prompt=cross_file_prompt,
    ))

    # Stage 3: Priority ranking (requires Stages 1 + 2)
    all_issues = merge_issues(file_analyses, cross_file_issues)
    ranked_issues = json.loads(call_claude(
        system="You are a technical lead. Rank code review findings by impact.",
        prompt=f"Rank these issues by severity and business impact:\n{json.dumps(all_issues)}",
    ))

    # Stage 4: Report generation (requires Stage 3)
    report = call_claude(
        system="Generate a clear, actionable code review report.",
        prompt=f"Create a PR review comment from these ranked findings:\n{json.dumps(ranked_issues)}",
    )

    return {"ranked_issues": ranked_issues, "report": report}
```

Each stage has a clear contract: what it receives, what it produces, and what it is responsible for. The pipeline is the same regardless of what code is being reviewed.

---

### Example 2 — Dynamic Decomposition for Open-Ended Investigation

```python
def investigate_bug(bug_report: str) -> str:
    """Dynamic adaptive decomposition — the agent decides how to investigate.

    Unlike prompt chaining, there is no fixed sequence. The agent forms
    hypotheses, gathers evidence, and adjusts its plan based on findings.
    """

    investigation_prompt = f"""You are a senior engineer investigating a bug.

Bug report:
{bug_report}

Your approach:
1. Form initial hypotheses based on the bug description.
2. Use available tools to gather evidence (read_file, grep_search, git_log,
   run_test, check_logs).
3. After each piece of evidence, reassess your hypotheses.
4. Narrow down until you identify the root cause.
5. Propose a fix with specific code changes.

Think step by step. Do NOT commit to a fixed investigation plan upfront —
let the evidence guide your next step.

Available tools: read_file, grep_search, git_log, run_test, check_logs"""

    # The agent runs in the standard agentic loop (see Agentic Loop Lifecycle).
    # It might:
    #   1. grep for the error message → finds it in payments/refund.py
    #   2. read_file payments/refund.py → spots a race condition
    #   3. git_log payments/refund.py → sees it was introduced 3 days ago
    #   4. read_file the specific commit → confirms the bug
    #   5. run_test to reproduce → test fails, confirming the hypothesis
    #
    # Or it might:
    #   1. grep for the error message → nothing found
    #   2. check_logs for the timeframe → finds a different error upstream
    #   3. Adjusts hypothesis: the reported error is a symptom, not the cause
    #   4. grep for the upstream error → finds it in auth/session.py
    #   5. read_file auth/session.py → identifies expired token handling bug
    #
    # The investigation path is completely different depending on what
    # each tool call reveals. A fixed pipeline cannot handle this.

    return run_agent(investigation_prompt, tools=INVESTIGATION_TOOLS)
```

The key difference: in prompt chaining, the developer defines the stages. In adaptive decomposition, the model defines its own next step based on what it has learned so far.

---

### Example 3 — Legacy Codebase Test Strategy

```python
# Scenario: You need to add tests to a legacy codebase with no existing tests.
# This is a HYBRID approach — fixed high-level stages with adaptive substeps.

def create_test_strategy(codebase_root: str) -> dict:
    """Hybrid: fixed pipeline stages with adaptive content within each stage."""

    # Stage 1 (Fixed): Discover codebase structure
    # The WHAT is fixed; the HOW adapts to what it finds
    discovery = call_claude(
        system="You are a codebase analyst.",
        prompt=f"""Explore {codebase_root} and produce:
1. Module dependency graph (which modules import which)
2. Entry points (main functions, API handlers, CLI commands)
3. Critical paths (authentication, payment, data persistence)
4. Code complexity hotspots (largest files, deepest nesting)

Use read_file, list_directory, and grep_search to explore.
Adapt your exploration based on what you discover.""",
    )

    # Stage 2 (Fixed): Prioritize what to test first
    prioritization = call_claude(
        system="You are a test strategy consultant.",
        prompt=f"""Given this codebase analysis:
{discovery}

Prioritize modules for testing. Rank by:
- Business criticality (payments > logging)
- Bug likelihood (complex code > simple CRUD)
- Dependency count (heavily-imported modules first)

Return an ordered list of modules with rationale.""",
    )

    # Stage 3 (Adaptive per module): Generate tests for each priority module
    # The approach for each module adapts based on what the module contains
    test_plans = {}
    for module in parse_priority_list(prioritization)[:5]:  # Top 5 modules
        test_plan = call_claude(
            system="You are a test engineer.",
            prompt=f"""Create a test plan for module: {module['name']}

Module analysis: {module['analysis']}
Dependencies: {module['dependencies']}

Decide the right mix of:
- Unit tests (isolated function testing)
- Integration tests (module interaction testing)
- Edge case tests (boundary conditions, error paths)

Adapt your approach to the module's characteristics. A pure-function
utility module needs different tests than a database-backed service.""",
        )
        test_plans[module["name"]] = test_plan

    return {
        "discovery": discovery,
        "prioritization": prioritization,
        "test_plans": test_plans,
    }
```

The high-level pipeline (discover, prioritize, generate) is fixed. But within each stage, the agent adapts — discovery explores differently based on codebase structure, and each module gets a test strategy tailored to its characteristics.

---

### Example 4 — Code Review Split Approach

```python
# Choosing the right decomposition for different review scenarios

def review_pull_request(pr_files: list[dict]) -> str:
    """Select decomposition strategy based on PR characteristics."""

    file_count = len(pr_files)
    total_lines = sum(f["lines_changed"] for f in pr_files)
    languages = set(f["language"] for f in pr_files)
    has_tests = any("test" in f["path"].lower() for f in pr_files)

    # SMALL PR (< 5 files, < 200 lines): Single-pass review
    if file_count < 5 and total_lines < 200:
        return single_pass_review(pr_files)

    # LARGE SAME-LANGUAGE PR: Fixed pipeline (prompt chaining)
    # Structure is predictable — per-file then cross-file
    if len(languages) == 1 and file_count >= 5:
        return code_review_pipeline(pr_files)

    # MULTI-LANGUAGE OR ARCHITECTURAL CHANGE: Adaptive decomposition
    # Need to understand the change's intent before deciding how to review
    if len(languages) > 1 or total_lines > 1000:
        return adaptive_review(pr_files)


def adaptive_review(pr_files: list[dict]) -> str:
    """Adaptive decomposition for complex PRs.

    The agent first understands the PR's intent, then decides how to
    decompose the review based on what it discovers.
    """
    return run_agent(
        prompt=f"""Review this pull request. It touches {len(pr_files)} files
across {len(set(f['language'] for f in pr_files))} languages.

Files changed:
{format_file_list(pr_files)}

First, understand the overall intent of this PR by reading key files.
Then decide how to organize your review:
- By feature/concern if the PR implements a feature across layers
- By risk level if the PR mixes risky and safe changes
- By dependency order if changes cascade through the codebase

Explain your chosen review structure before diving in.""",
        tools=REVIEW_TOOLS,
    )


# The decision of WHICH strategy to use is itself a form of adaptive
# decomposition — the orchestrator examines the PR's characteristics
# and routes to the appropriate review pattern.
```

---

## Key Takeaways for the Exam

1. **Prompt chaining** (fixed pipeline) is best for predictable, repeatable tasks where stages are known in advance.
2. **Adaptive decomposition** is best for open-ended investigations where the path depends on what is discovered.
3. **Hybrid approaches** use fixed high-level stages with adaptive content within each stage.
4. The choice between approaches depends on **how predictable the task structure is**, not on task difficulty.
5. Overly rigid pipelines fail on novel problems; overly dynamic approaches waste effort on routine tasks.

---

## Related Topics

- [[Domain 1 - Agentic Architecture & Orchestration]] — Parent domain overview
- [[Multi-Agent Orchestration]] — Decomposition in multi-agent systems
- [[Subagent Configuration and Context Passing]] — Configuring agents for each decomposed subtask
- [[Agentic Loop Lifecycle]] — Each stage or adaptive step runs within the agentic loop
- [[Prompt Engineering Principles]] — Crafting effective prompts for each pipeline stage
