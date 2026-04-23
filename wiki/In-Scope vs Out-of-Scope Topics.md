---
tags: [reference, exam-scope]
---

# In-Scope vs Out-of-Scope Topics

Use this page to focus your study time. Topics on the left WILL appear on the exam. Topics on the right will NOT.

---

## In-Scope (Will Be Tested)

| Topic | Wiki Page |
|-------|-----------|
| Agentic loop implementation (stop_reason, tool results, termination) | [[Agentic Loop Lifecycle]] |
| Multi-agent orchestration (coordinator-subagent, task decomposition, parallel execution) | [[Multi-Agent Orchestration]] |
| Subagent context management (explicit passing, state persistence, crash recovery) | [[Subagent Configuration and Context Passing]] |
| Tool interface design (descriptions, splitting/consolidating, naming) | [[Tool Interface Design]] |
| MCP tool and resource design (resources for catalogs, tools for actions) | [[MCP Server Configuration]] |
| MCP server configuration (project vs user scope, env vars, multi-server) | [[MCP Server Configuration]] |
| Error handling and propagation (structured errors, transient vs business, local recovery) | [[Structured Error Responses]] / [[Error Propagation in Multi-Agent Systems]] |
| Escalation decision-making (explicit criteria, customer preferences, policy gaps) | [[Escalation and Ambiguity Resolution]] |
| CLAUDE.md configuration (hierarchy, @import, .claude/rules/ with globs) | [[CLAUDE.md Configuration Hierarchy]] |
| Custom commands and skills (project/user scope, context:fork, allowed-tools) | [[Custom Slash Commands and Skills]] |
| Plan mode vs direct execution (complexity assessment, architectural decisions) | [[Plan Mode vs Direct Execution]] |
| Iterative refinement (input/output examples, test-driven, interview pattern) | [[Iterative Refinement Techniques]] |
| Structured output via tool_use (schema design, tool_choice, nullable fields) | [[JSON Schemas and Tool Use]] |
| Few-shot prompting (ambiguous scenarios, format consistency, false positive reduction) | [[Few-Shot Prompting]] |
| Batch processing (Message Batches API, latency tolerance, custom_id, failure handling) | [[Batch Processing]] |
| Context window optimization (trimming, fact extraction, position-aware ordering) | [[Context Window Management]] |
| Human review workflows (confidence calibration, stratified sampling, accuracy segmentation) | [[Human Review Workflows]] |
| Information provenance (claim-source mappings, temporal data, conflict annotation) | [[Information Provenance and Uncertainty]] |
| Hooks (PostToolUse, tool call interception, deterministic enforcement) | [[Hooks and Interception]] |
| Session management (--resume, fork_session, named sessions, context isolation) | [[Session Management]] |
| Path-specific rules (.claude/rules/ with YAML frontmatter glob patterns) | [[Path-Specific Rules]] |
| CI/CD integration (-p flag, --output-format json, --json-schema) | [[CI-CD Integration]] |
| Multi-instance review (independent review sessions, multi-pass architecture) | [[Multi-Instance Review Architectures]] |
| Large codebase exploration (context degradation, scratchpad files, /compact) | [[Large Codebase Exploration]] |

---

## Out-of-Scope (Will NOT Be Tested)

| Topic | Why Excluded |
|-------|-------------|
| Fine-tuning Claude models or training custom models | Model training is a separate domain |
| Claude API authentication, billing, or account management | Operational, not architectural |
| Detailed language/framework implementation | Beyond tool/schema configuration needs |
| Deploying or hosting MCP servers (infra, networking, containers) | Infrastructure, not design |
| Claude's internal architecture, training process, or model weights | Internal implementation |
| Constitutional AI, RLHF, or safety training methodologies | Research/training domain |
| Embedding models or vector database implementation | Different technology stack |
| Computer use (browser automation, desktop interaction) | Separate Claude capability |
| Vision/image analysis capabilities | Separate Claude capability |
| Streaming API implementation or server-sent events | Implementation detail |
| Rate limiting, quotas, or API pricing calculations | Operational concern |
| OAuth, API key rotation, or authentication protocol details | Security implementation |
| Specific cloud provider configurations (AWS, GCP, Azure) | Infrastructure-specific |
| Performance benchmarking or model comparison metrics | Evaluation domain |
| Prompt caching implementation details | Beyond "knowing it exists" |
| Token counting algorithms or tokenization specifics | Internal implementation |

---

## Key Insight

The exam focuses on **architectural decisions and tradeoffs**, not implementation details. You need to know:
- **When** to use each pattern (not how to deploy it)
- **Why** one approach is better than alternatives in a given scenario
- **What** the tradeoffs are (cost, latency, reliability, complexity)

---

## Related

- [[00 - Home]]
- [[Technologies and Concepts Glossary]]
