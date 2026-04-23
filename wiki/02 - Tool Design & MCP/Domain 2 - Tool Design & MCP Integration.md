---
tags:
  - domain-2
  - tool-design
  - mcp
  - exam-overview
domain: "Tool Design & MCP Integration"
weight: "18%"
---

# Domain 2 — Tool Design & MCP Integration

> **Exam Weight: 18%**
> This domain tests your ability to design, describe, distribute, and configure tools that Claude agents use effectively. It also covers MCP (Model Context Protocol) server setup and Claude Code's built-in tool suite.

---

## Task Statements

| Task | Title | Link |
|------|-------|------|
| 2.1 | Design tool interfaces that enable reliable LLM-driven selection | [[Tool Interface Design]] |
| 2.2 | Implement structured error handling across tool boundaries | [[Structured Error Responses]] |
| 2.3 | Distribute tools and control selection behavior | [[Tool Distribution and Tool Choice]] |
| 2.4 | Configure MCP servers at project and user levels | [[MCP Server Configuration]] |
| 2.5 | Leverage Claude Code's built-in tools for codebase exploration | [[Built-in Tools]] |

---

## Key Themes

### Tool Descriptions Drive Selection
Claude does not inspect source code to decide which tool to call. The **tool description** is the sole interface between the model and the tool. Poorly written descriptions lead to misrouting, hallucinated parameters, and wasted tokens. See [[Tool Interface Design]] for patterns that eliminate ambiguity.

### Error Handling Is an Architecture Decision
A tool that returns `"Operation failed"` leaves the agent with no recovery path. Structured error responses with categories, retryability flags, and human-readable context let agents — and subagents — recover locally or escalate intelligently. See [[Structured Error Responses]].

### Fewer Tools, Tighter Scopes
Giving an agent 18 tools when it needs 4 degrades selection accuracy. Each agent should receive only the tools relevant to its role. The `tool_choice` parameter adds further control. See [[Tool Distribution and Tool Choice]].

### MCP Configuration and Discovery
MCP servers expose tools discovered at connection time. Project-level and user-level configuration files control which servers are available and how credentials flow. See [[MCP Server Configuration]].

### Built-in Tools for Code Exploration
Claude Code ships with Grep, Glob, Read, Write, and Edit. Understanding their strengths and limitations — especially how to chain them for incremental codebase understanding — is a core skill. See [[Built-in Tools]].

---

## Related Domains

- [[Domain 1 - Agentic Architecture]] — how agents orchestrate tool calls
- [[Domain 3 - Claude Code Configuration]] — workspace-level settings that affect tool availability
- [[Domain 4 - Prompt Engineering]] — system prompts that influence tool selection behavior
- [[Domain 5 - Context Management]] — keeping tool results within context window limits
