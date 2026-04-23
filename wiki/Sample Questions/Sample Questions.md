---
tags:
  - sample-questions
  - exam-prep
  - all-domains
  - practice
---

# Sample Questions

> All 12 sample questions from the Claude Certified Architect -- Foundations exam guide, with full explanations and distractor analysis.

---

## How to Use This Page

1. **First pass:** Cover the answer and try to answer each question yourself.
2. **Second pass:** Read the explanation and understand *why* each distractor is wrong.
3. **Third pass:** Follow the topic links to review any areas where you were uncertain.

Each question is tagged with the domain(s) and topic(s) it tests.

---

## Question 1 — Programmatic Prerequisite for Tool Ordering

**Scenario:** Customer Support Resolution Agent

> You are building a customer support agent using the Claude Agent SDK. When the agent handles a refund request, it must first look up the order details before it can process the refund. During testing, you notice that sometimes the agent tries to call `process_refund` before calling `lookup_order`, leading to errors.

**What is the most reliable way to ensure the agent always looks up the order before processing a refund?**

- **A) Add a programmatic check in a PostToolUse hook that verifies `lookup_order` was called before allowing `process_refund` to execute** ✅
- B) Add an instruction to the system prompt saying "Always look up the order before processing a refund"
- C) Remove `process_refund` from the tool list and have the agent return refund details as text
- D) Set `tool_choice` to force `lookup_order` on every request

**Correct Answer: A**

**Explanation:**

- **A is correct** because deterministic enforcement of ordering requires programmatic gates, not prompt-based suggestions. A PostToolUse hook can inspect the conversation history, confirm that `lookup_order` has been called with the relevant order ID, and block `process_refund` if the prerequisite was not met. This is a core Domain 1 principle: use hooks for things that *must* happen, prompts for things that *should* happen. See [[Hooks and Interception]] and [[Multi-Step Workflows and Handoffs]].
- **B is wrong** because system prompt instructions are advisory, not deterministic. Claude will *usually* follow them, but under pressure (long conversations, ambiguous user requests), it may skip steps. For a compliance-critical workflow like refunds, "usually" is not enough.
- **C is wrong** because removing the tool entirely eliminates the agent's ability to process refunds programmatically. Returning refund details as text means no structured action is taken -- the refund would not actually be processed.
- **D is wrong** because `tool_choice: {"type": "tool", "name": "lookup_order"}` forces that tool on *every* API call, not just refund requests. This breaks all non-refund interactions and does not create a conditional ordering constraint.

**Topics:** [[Hooks and Interception]] | [[Agentic Loop Lifecycle]] | [[Multi-Step Workflows and Handoffs]]

---

## Question 2 — Tool Description Improvement for Selection Reliability

**Scenario:** Customer Support Resolution Agent

> Your customer support agent has two tools: `get_customer` and `lookup_order`. During testing, you notice Claude sometimes calls `get_customer` when it should call `lookup_order`, and vice versa. Both tools have short, generic descriptions.

**What is the most effective way to improve tool selection reliability?**

- A) Add a system prompt instruction listing when to use each tool
- **B) Rewrite each tool description to include specific input formats, output shapes, use cases, and explicit boundary statements about what the tool does NOT do** ✅
- C) Rename both tools to have longer, more descriptive names
- D) Reduce the tool list to a single tool that handles both operations

**Correct Answer: B**

**Explanation:**

- **B is correct** because the tool description is the primary information Claude uses to decide which tool to call. Detailed descriptions that specify input formats, output shapes, intended use cases, and boundary conditions ("Does NOT return order history -- use lookup_order for that") give Claude the information it needs to differentiate tools reliably. This is the core lesson of [[Tool Interface Design]].
- **A is wrong** because while system prompt instructions can help, they are a secondary signal. The tool description is the primary driver of tool selection. If the descriptions are ambiguous, system prompt instructions cannot fully compensate -- and they can actually create keyword affinity problems (see [[Tool Interface Design#System Prompt Keyword Sensitivity]]).
- **C is wrong** because tool names are a weak signal. A longer name like `get_customer_profile_information` does not tell Claude what inputs are valid, what the output contains, or when NOT to use it. The description carries that weight.
- **D is wrong** because merging tools into a single generic tool makes the problem worse. A single `handle_customer_data` tool would need to handle multiple operation types, increasing the complexity of the input schema and making it harder for Claude to construct correct calls.

**Topics:** [[Tool Interface Design]] | [[Domain 2 - Tool Design & MCP Integration]]

---

## Question 3 — Escalation Calibration with Explicit Criteria

**Scenario:** Customer Support Resolution Agent

> Your customer support agent is supposed to escalate to a human when it cannot resolve an issue. During evaluation, you find it escalates too often for routine questions and too rarely for genuinely complex policy questions. The current system prompt says "escalate when you are not confident."

**What is the best way to calibrate escalation behavior?**

- **A) Replace the vague "not confident" instruction with explicit escalation criteria: specific trigger conditions (e.g., "customer explicitly requests a human," "issue involves a policy exception not covered in training data," "monetary amount exceeds $500") and few-shot examples of escalate vs. resolve decisions** ✅
- B) Add a confidence score threshold and escalate whenever the score drops below 0.7
- C) Remove escalation entirely and have the agent always attempt resolution
- D) Add a second agent that reviews every response and decides whether to escalate

**Correct Answer: A**

**Explanation:**

- **A is correct** because vague instructions like "escalate when not confident" give Claude no calibration signal. Claude does not have a reliable internal confidence meter that maps to your business requirements. Explicit criteria (customer request, policy gaps, monetary thresholds) combined with few-shot examples of correct escalation decisions give Claude concrete patterns to match against. This is a core concept in [[Escalation and Ambiguity Resolution]].
- **B is wrong** because Claude's self-reported confidence scores are not well-calibrated for business decisions. A score of 0.7 means different things in different contexts. The model may report high confidence on a wrong answer or low confidence on a correct one. Threshold-based approaches require external calibration data, which the question does not mention.
- **C is wrong** because removing escalation entirely means complex policy questions, legal issues, and frustrated customers are handled by an agent that lacks the authority or knowledge to resolve them. This increases risk and decreases customer satisfaction.
- **D is wrong** because adding a second agent to review every response doubles cost and latency without addressing the root cause: unclear escalation criteria. The second agent would face the same "when should I escalate?" ambiguity unless it also has explicit criteria -- at which point you should just give those criteria to the first agent.

**Topics:** [[Escalation and Ambiguity Resolution]] | [[Few-Shot Prompting]] | [[Domain 5 - Context Management & Reliability]]

---

## Question 4 — Custom /review Slash Command Location

**Scenario:** Code Generation with Claude Code

> Your team wants to create a custom `/review` slash command in Claude Code that runs a standardized code review process. The command should be available to all team members working on the project.

**Where should this custom command be defined?**

- **A) In the `.claude/commands/review.md` directory at the project root, committed to version control** ✅
- B) In `~/.claude/commands/review.md` in each developer's home directory
- C) In the system prompt of a Claude API call
- D) As an MCP tool in `.mcp.json`

**Correct Answer: A**

**Explanation:**

- **A is correct** because project-level slash commands live in `.claude/commands/` at the project root. Files in this directory become available as `/commands` to anyone working on the project. By committing this to version control, all team members get the same command automatically. The file content becomes the prompt for the command. See [[CLAUDE.md Configuration Hierarchy]] and [[Domain 3 - Claude Code Configuration & Workflows]].
- **B is wrong** because `~/.claude/commands/` is the *user-level* location. Commands placed here are personal to one developer and are not shared with the team. Each developer would need to manually create the same file, leading to drift and inconsistency.
- **C is wrong** because slash commands are a Claude Code feature, not a Claude API feature. The Claude API does not have a concept of slash commands. This conflates two different interfaces.
- **D is wrong** because MCP tools and slash commands are different mechanisms. An MCP tool defined in `.mcp.json` exposes a callable tool to the agent, but it does not create a slash command that developers invoke from the Claude Code interface. They serve different purposes.

**Topics:** [[CLAUDE.md Configuration Hierarchy]] | [[Domain 3 - Claude Code Configuration & Workflows]]

---

## Question 5 — Plan Mode for Microservice Restructuring

**Scenario:** Code Generation with Claude Code

> You need to use Claude Code to restructure a large monolithic module into three microservices. This requires coordinated changes across 15+ files, including moving functions, updating imports, and modifying configuration files. You want to review the plan before any files are changed.

**What is the best approach?**

- **A) Use plan mode (Shift+Tab to toggle) to have Claude analyze the codebase and produce a structured plan of all changes before executing any of them, then review and approve the plan** ✅
- B) Ask Claude Code to make all changes at once and use git to review the diff afterward
- C) Make each change as a separate Claude Code session, reviewing each one individually
- D) Write a detailed specification document and paste it into Claude Code as context

**Correct Answer: A**

**Explanation:**

- **A is correct** because plan mode is specifically designed for this use case: complex, multi-file changes where you want to understand the full scope before execution. In plan mode, Claude analyzes the codebase, identifies all necessary changes, and presents a structured plan. You can review, provide feedback, and iterate on the plan before toggling back to execution mode. This prevents partial changes and gives you a holistic view. See [[Domain 3 - Claude Code Configuration & Workflows]].
- **B is wrong** because making all changes at once without review risks incorrect transformations that are difficult to untangle. While git provides a safety net, reviewing a 15-file diff after the fact is much harder than reviewing a plan beforehand. You lose the opportunity for iterative feedback.
- **C is wrong** because separate sessions lose context. Each session would not know what the previous sessions changed, leading to inconsistent imports, duplicate function definitions, or missed dependencies. The restructuring requires a holistic view that spans all 15+ files.
- **D is wrong** because while providing context is helpful, a specification document does not give you the review-before-execute workflow that plan mode provides. Claude Code would still proceed to make changes immediately unless you use plan mode. The specification could complement plan mode, but it does not replace it.

**Topics:** [[Domain 3 - Claude Code Configuration & Workflows]] | [[Large Codebase Exploration]]

---

## Question 6 — Path-Specific Rules with Glob Patterns

**Scenario:** Code Generation with Claude Code

> Your project has strict formatting rules for API endpoint files in `src/api/**/*.ts` but more relaxed rules for test files. You want Claude Code to automatically apply different instructions based on which files it is editing.

**How should you configure this?**

- **A) Create rule files in `.claude/rules/` with YAML frontmatter containing glob patterns that specify which file paths each rule applies to** ✅
- B) Add all rules to the project-level `CLAUDE.md` with comments indicating which files they apply to
- C) Create separate CLAUDE.md files in each subdirectory
- D) Use the system prompt to describe all formatting rules with file path conditions

**Correct Answer: A**

**Explanation:**

- **A is correct** because `.claude/rules/` supports YAML frontmatter with `globs` fields that specify file path patterns. For example, a rule file with `globs: ["src/api/**/*.ts"]` in its frontmatter will only be applied when Claude Code is working on files matching that pattern. This is the purpose-built mechanism for path-specific rules. See [[CLAUDE.md Configuration Hierarchy]].
- **B is wrong** because a single `CLAUDE.md` file applies to all files in the project equally. Adding comments like "for API files only" is an advisory instruction that Claude may not reliably follow. There is no glob-based conditional logic in a flat `CLAUDE.md`.
- **C is wrong** because while directory-level `CLAUDE.md` files do scope instructions to that directory and its children, this is less precise than glob patterns. You cannot target `**/*.ts` files specifically -- the rules would apply to all files in the directory, including non-TypeScript files.
- **D is wrong** because Claude Code does not use the system prompt in the same way as the Claude API. Configuration is done through `CLAUDE.md` and `.claude/rules/`, not through system prompts. Even if you could set a system prompt, path-conditional logic in natural language is unreliable.

**Topics:** [[CLAUDE.md Configuration Hierarchy]] | [[Domain 3 - Claude Code Configuration & Workflows]]

---

## Question 7 — Overly Narrow Task Decomposition in Coordinator

**Scenario:** Multi-Agent Research System

> You are building a multi-agent research system where a coordinator decomposes research questions into subtasks for specialized subagents. During testing, you notice that the coordinator is breaking questions into extremely granular subtasks (e.g., "find the CEO's name," "find the CEO's start date," "find the CEO's previous company"), causing excessive API calls and fragmented context.

**What is the best way to address this?**

- A) Add more subagents, each specialized in a narrower domain
- **B) Adjust the coordinator's system prompt to define appropriate task granularity -- each subtask should be a coherent research unit that a subagent can complete in one pass, with examples of good vs. bad decomposition** ✅
- C) Remove the coordinator and have a single agent handle all research
- D) Set a hard limit on the number of subtasks the coordinator can create

**Correct Answer: B**

**Explanation:**

- **B is correct** because the problem is inappropriate granularity in task decomposition. The coordinator needs guidance on what constitutes a meaningful subtask. A good subtask like "Research the current CEO: background, tenure, and previous roles" gives the subagent enough scope to produce a coherent output, while a bad subtask like "find the CEO's name" fragments reasoning and multiplies API calls. Few-shot examples of good vs. bad decomposition calibrate the coordinator's judgment. See [[Task Decomposition Strategies]] and [[Multi-Agent Orchestration]].
- **A is wrong** because adding more specialized subagents does not fix the granularity problem. If the coordinator is already creating overly narrow tasks, more subagents would just handle the same fragmented tasks -- or the coordinator would decompose even further to match the new specializations.
- **C is wrong** because removing multi-agent orchestration entirely loses the benefits of parallel research, specialized tool access, and isolated context. The architecture is sound; the decomposition granularity needs tuning.
- **D is wrong** because a hard limit on subtask count is a blunt instrument. If the limit is 5, the coordinator might create 5 still-too-narrow tasks, or it might be unable to adequately decompose a genuinely complex question that needs 7 subtasks. The problem is quality of decomposition, not quantity.

**Topics:** [[Task Decomposition Strategies]] | [[Multi-Agent Orchestration]] | [[Domain 1 - Agentic Architecture & Orchestration]]

---

## Question 8 — Structured Error Propagation from Subagent

**Scenario:** Multi-Agent Research System

> In your multi-agent research system, a data-gathering subagent encounters an API rate limit error. The coordinator needs to understand what happened, whether to retry, and what partial results are available.

**What is the best pattern for the subagent to propagate this error?**

- **A) Return a structured error object containing `errorCategory: "transient"`, `isRetryable: true`, `retryAfterMs`, the partial results gathered so far, and the specific source that failed** ✅
- B) Return a text message saying "API rate limit hit, please try again later"
- C) Silently retry indefinitely until the rate limit clears
- D) Throw an exception that terminates the entire pipeline

**Correct Answer: A**

**Explanation:**

- **A is correct** because structured error propagation gives the coordinator actionable information: the error category (transient, so likely temporary), whether to retry (yes), how long to wait, what partial data is already available, and which source specifically failed. This allows the coordinator to make intelligent decisions -- retry the failed source while using the partial results, or proceed with incomplete data and annotate the gap. See [[Structured Error Responses]] and [[Error Propagation in Multi-Agent Systems]].
- **B is wrong** because an unstructured text message requires the coordinator to parse natural language to extract retry logic. "Please try again later" does not specify how long to wait, does not distinguish transient from permanent failures, and does not include partial results.
- **C is wrong** because infinite silent retries can hang the entire pipeline. The coordinator has no visibility into the subagent's status and cannot make decisions about timeouts, fallbacks, or proceeding with partial data. Rate limits may persist for minutes or hours.
- **D is wrong** because terminating the entire pipeline over a transient error in one source is disproportionate. Other subagents may have completed successfully. The coordinator should be able to proceed with partial results, not lose everything because one API was temporarily unavailable.

**Topics:** [[Structured Error Responses]] | [[Error Propagation in Multi-Agent Systems]] | [[Multi-Agent Orchestration]]

---

## Question 9 — Scoped verify_fact Tool for Synthesis Agent

**Scenario:** Multi-Agent Research System

> Your research system has a synthesis agent that combines findings from multiple subagents into a final report. You want the synthesis agent to be able to verify claims against source documents but NOT to make external API calls or modify any data.

**How should you configure the synthesis agent's tool access?**

- **A) Define the synthesis agent with an `allowedTools` list that includes only `verify_fact` and excludes all external API and write tools** ✅
- B) Give the synthesis agent all tools but add a system prompt instruction saying "do not use external API tools"
- C) Create a separate MCP server with only the verification tool
- D) Use `tool_choice` to force the synthesis agent to only use `verify_fact`

**Correct Answer: A**

**Explanation:**

- **A is correct** because the Claude Agent SDK's `allowedTools` parameter on agent/subagent definitions provides a programmatic allowlist. The synthesis agent physically cannot call tools not in the list -- this is enforcement, not suggestion. This follows the principle of least privilege: agents should only have access to the tools they need. See [[Multi-Agent Orchestration]] and [[Subagent Configuration and Context Passing]].
- **B is wrong** because system prompt instructions are advisory. Under sufficiently complex conditions, Claude may still attempt to call external tools. For a security-relevant constraint ("do not modify data"), you need programmatic enforcement, not prompt-based guidance.
- **C is wrong** because while creating a separate MCP server would technically work, it is over-engineered for this use case. The `allowedTools` configuration on the agent definition achieves the same isolation without infrastructure changes. MCP server separation is useful when tools need different hosting or authentication, not just access control.
- **D is wrong** because `tool_choice: {"type": "tool", "name": "verify_fact"}` forces the agent to call `verify_fact` on *every* API turn. The synthesis agent also needs to produce text output (the report). Forcing a single tool means it cannot stop and return its synthesis -- it would be stuck in an infinite tool-calling loop.

**Topics:** [[Multi-Agent Orchestration]] | [[Subagent Configuration and Context Passing]] | [[Tool Interface Design]]

---

## Question 10 — `-p` Flag for CI/CD Non-Interactive Mode

**Scenario:** Claude Code for Continuous Integration

> You are integrating Claude Code into a CI/CD pipeline to automatically review pull requests. The pipeline runs in a headless environment with no interactive terminal.

**What is the correct way to run Claude Code in this environment?**

- **A) Use the `-p` flag (e.g., `claude -p "Review this PR for security issues"`) to run Claude Code in non-interactive mode, which accepts a prompt as an argument and returns results to stdout** ✅
- B) Use `claude --batch` to enable batch processing mode
- C) Pipe the prompt into Claude Code's stdin using `echo "Review this PR" | claude`
- D) Set the `CLAUDE_HEADLESS=true` environment variable before running Claude Code

**Correct Answer: A**

**Explanation:**

- **A is correct** because the `-p` flag is Claude Code's purpose-built mechanism for non-interactive (headless) execution. It takes a prompt as a command-line argument and writes results to stdout, making it suitable for CI/CD pipelines, shell scripts, and any environment without a terminal. Combined with `--output-format json`, it produces machine-parseable output. See [[CI/CD Integration]] and [[Domain 3 - Claude Code Configuration & Workflows]].
- **B is wrong** because `--batch` is not a valid Claude Code CLI flag. The Message Batches API (for batch processing) is a separate Claude API feature, not a Claude Code CLI option.
- **C is wrong** because while piping to stdin might seem intuitive, Claude Code's non-interactive mode is activated by the `-p` flag, not by stdin piping. The CLI needs the explicit flag to know it should run in headless mode and exit after completion.
- **D is wrong** because `CLAUDE_HEADLESS` is not a recognized environment variable for Claude Code. The `-p` flag is the documented mechanism for non-interactive execution.

**Topics:** [[CI/CD Integration]] | [[Domain 3 - Claude Code Configuration & Workflows]]

---

## Question 11 — Batch API for Overnight Reports Only

**Scenario:** Claude Code for Continuous Integration

> Your company generates nightly compliance reports by processing 500 documents through Claude. The reports are not time-sensitive -- they just need to be ready by the next morning. You want to minimize cost.

**What is the most cost-effective approach?**

- **A) Use the Message Batches API to submit all 500 documents as a batch job. The Batches API offers 50% cost savings over synchronous requests and has a 24-hour processing window, which fits the overnight timeline.** ✅
- B) Use synchronous API calls with `max_tokens` set very low to reduce costs
- C) Process all 500 documents in a single API call with a very large context window
- D) Use Claude Code's `-p` flag in a shell loop to process each document sequentially

**Correct Answer: A**

**Explanation:**

- **A is correct** because the Message Batches API is purpose-built for high-volume, non-time-sensitive workloads. It provides a 50% cost reduction compared to synchronous API calls. Each request in the batch gets a `custom_id` for tracking, and results are available within a 24-hour window. For overnight reports that need to be ready by morning, this is the ideal fit. See [[Batch Processing]] and [[Domain 5 - Context Management & Reliability]].
- **B is wrong** because reducing `max_tokens` does not reduce input token costs -- it only limits the length of the response. If the reports require detailed analysis, artificially short responses would produce incomplete reports. Cost savings from shorter outputs are marginal compared to the 50% batch discount.
- **C is wrong** because you cannot process 500 separate documents in a single API call. The context window has a fixed token limit (regardless of size), and stuffing 500 documents into one prompt would exceed it. Even if it fit, the model would struggle to produce 500 separate reports in one response.
- **D is wrong** because using Claude Code's `-p` flag in a sequential loop makes 500 separate synchronous calls at full price. This is the most expensive option and also the slowest (sequential, not parallel). It uses the wrong tool for the job -- Claude Code is designed for interactive development, not batch document processing.

**Topics:** [[Batch Processing]] | [[CI/CD Integration]] | [[Domain 5 - Context Management & Reliability]]

---

## Question 12 — Multi-Pass Review for Large PRs

**Scenario:** Claude Code for Continuous Integration

> You are building an automated code review pipeline using Claude Code. Some pull requests are very large (1000+ lines changed). You find that single-pass reviews miss issues in files reviewed late in the context window.

**What is the best approach to improve review quality for large PRs?**

- **A) Implement a multi-pass review strategy: first pass reviews each file or logical group independently (preserving full attention), then a synthesis pass combines findings into a unified report with cross-file issues** ✅
- B) Increase `max_tokens` to ensure the full PR fits in the context window
- C) Only review the first 500 lines and skip the rest
- D) Ask Claude to focus on the most important files and ignore the rest

**Correct Answer: A**

**Explanation:**

- **A is correct** because context window degradation is a real phenomenon: information placed late in a long context receives less attention than information at the beginning (the "lost in the middle" effect). By reviewing files independently in separate passes, each file gets full attention. A synthesis pass then identifies cross-file issues (inconsistent naming, missing imports, API contract mismatches) that file-level reviews cannot catch. This is a practical application of [[Context Window Management]] principles and [[Prompt Engineering & Structured Output]] chaining patterns.
- **B is wrong** because `max_tokens` controls the *output* length, not the context window size. Even if the full PR fits in the context window, the degradation issue remains -- information in the middle of a very long context is processed less reliably than information at the edges. More context does not mean better attention distribution.
- **C is wrong** because arbitrarily truncating the review to 500 lines means you miss all issues in the remaining lines. Bugs and security vulnerabilities do not conveniently cluster in the first half of a PR.
- **D is wrong** because "most important files" requires the model to make a judgment call about importance before it has reviewed the content. It might deprioritize a utility file that contains a critical security vulnerability. Systematic coverage is more reliable than selective attention.

**Topics:** [[Context Window Management]] | [[CI/CD Integration]] | [[Domain 4 - Prompt Engineering & Structured Output]]

---

## Score Tracker

| Question | Domain(s) | Your Answer | Correct? |
|----------|-----------|-------------|----------|
| Q1 | 1, 2 | | |
| Q2 | 2 | | |
| Q3 | 5 | | |
| Q4 | 3 | | |
| Q5 | 3 | | |
| Q6 | 3 | | |
| Q7 | 1 | | |
| Q8 | 2, 5 | | |
| Q9 | 1, 2 | | |
| Q10 | 3 | | |
| Q11 | 5 | | |
| Q12 | 4, 5 | | |

---

## Related Pages

- [[Domain 1 - Agentic Architecture & Orchestration]]
- [[Domain 2 - Tool Design & MCP Integration]]
- [[Domain 3 - Claude Code Configuration & Workflows]]
- [[Domain 4 - Prompt Engineering & Structured Output]]
- [[Domain 5 - Context Management & Reliability]]
- [[Exercise 1 - Multi-Tool Agent]]
- [[Exercise 2 - Team Workflow Configuration]]
- [[Exercise 3 - Data Extraction Pipeline]]
- [[Exercise 4 - Multi-Agent Research Pipeline]]
