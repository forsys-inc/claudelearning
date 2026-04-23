---
tags:
  - domain-2
  - task-2-4
  - mcp
  - configuration
  - mcp-servers
domain: "Tool Design & MCP Integration"
task: "2.4"
---

# MCP Server Configuration (Task 2.4)

> **Core principle:** MCP servers expose tools that Claude discovers at connection time. Configuration happens at two levels — project (shared with the team) and user (personal credentials). Understanding scoping, environment variable expansion, resource exposure, and when to use community servers vs custom implementations is essential.

---

## Configuration Scoping

MCP server configuration lives in two locations with different scopes:

### Project-Level: `.mcp.json`

Lives in the project root. Checked into version control. Shared by every team member who clones the repo.

**Use for:**
- Servers the entire team needs (GitHub, Jira, internal APIs)
- Tool configurations that are part of the project's architecture
- Servers where credentials come from environment variables (not hardcoded)

**Path:** `<project-root>/.mcp.json`

### User-Level: `~/.claude.json`

Lives in the user's home directory. Never checked into version control. Personal to each developer.

**Use for:**
- Servers with personal API keys (personal GitHub token, personal Notion workspace)
- Experimental or optional servers not needed by the whole team
- Servers that vary by developer (e.g., different cloud accounts)

**Path:** `~/.claude.json`

### Precedence

When both files define the same server name, the project-level `.mcp.json` takes precedence. This ensures team-wide consistency while allowing personal overrides for servers not defined at the project level.

---

## Environment Variable Expansion

The `.mcp.json` file supports `${ENV_VAR}` syntax for credentials. This keeps secrets out of version control while letting the team share a common configuration.

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_TOKEN": "${GITHUB_TOKEN}"
      }
    }
  }
}
```

At connection time, `${GITHUB_TOKEN}` is resolved from the shell environment. Each developer sets their own token in their `.bashrc`, `.zshrc`, or `.env` file. The `.mcp.json` is safely committed without exposing any credentials.

---

## Tool Discovery at Connection Time

MCP tools are **not** statically declared in configuration files. Instead:

1. Claude Code starts and reads `.mcp.json` and `~/.claude.json`.
2. It launches each configured MCP server process.
3. It sends a `tools/list` request to each server.
4. Each server responds with its available tools (names, descriptions, schemas).
5. All tools from all connected servers become available simultaneously.

**Implication:** If a server adds a new tool, Claude discovers it on the next connection — no configuration change needed. If a server is down, its tools are simply absent.

**Implication for descriptions:** Since tools from multiple servers are merged into a single list, tool naming conflicts and description quality become critical. A poorly described MCP tool competes with well-described built-in tools and may be ignored or misused.

---

## MCP Resources

Beyond tools, MCP servers can expose **resources** — read-only content that Claude can access without calling a tool. Resources are useful for content catalogs, reference data, and pre-computed summaries.

Resources have a URI pattern and return structured content:

```
resource://jira/sprint-summary
resource://docs/api-reference/auth
resource://metrics/dashboard/q1-2026
```

**Use resources when:**
- The content is read-only and does not require parameters.
- You want to expose a catalog of items Claude can browse.
- The data changes infrequently and can be served without computation.

**Use tools when:**
- The operation requires input parameters.
- The operation has side effects (writes, updates, deletes).
- The result depends on dynamic computation.

---

## Community MCP Servers vs Custom Implementations

The MCP ecosystem has a growing library of community-built servers for popular services (GitHub, Slack, Notion, PostgreSQL, etc.). Before building a custom MCP server, check whether a community server exists.

**Prefer community servers when:**
- A well-maintained server exists for your service.
- The server's tool descriptions are adequate (or can be supplemented).
- You do not need custom business logic beyond what the service API provides.

**Build custom servers when:**
- No community server exists for your service.
- You need to combine multiple API calls into a single tool (e.g., "create issue and assign to sprint" as one operation).
- You need custom tool descriptions tailored to your team's workflow.
- You need to enforce business rules at the tool level.

---

## Examples

### Example 1: `.mcp.json` with Environment Variable Expansion

A project that uses GitHub for code, Jira for tracking, and a custom internal API:

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_TOKEN": "${GITHUB_TOKEN}"
      }
    },
    "jira": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-jira"],
      "env": {
        "JIRA_URL": "${JIRA_URL}",
        "JIRA_API_TOKEN": "${JIRA_API_TOKEN}",
        "JIRA_USER_EMAIL": "${JIRA_USER_EMAIL}"
      }
    },
    "internal-api": {
      "command": "node",
      "args": ["./mcp-servers/internal-api/index.js"],
      "env": {
        "API_BASE_URL": "${INTERNAL_API_URL}",
        "API_KEY": "${INTERNAL_API_KEY}"
      }
    }
  }
}
```

**Key points:**
- `github` and `jira` use community servers via `npx` — no custom code needed.
- `internal-api` is a custom server (no community option exists for this proprietary API).
- All credentials use `${...}` expansion — the file is safe to commit.
- Each developer sets their own environment variables locally.

---

### Example 2: `~/.claude.json` Personal Server Configuration

A developer's personal configuration with a Notion workspace and an experimental database explorer:

```json
{
  "mcpServers": {
    "notion": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-notion"],
      "env": {
        "NOTION_API_KEY": "ntn_abc123456789"
      }
    },
    "db-explorer": {
      "command": "python",
      "args": ["/Users/dev/tools/mcp-db-explorer/server.py"],
      "env": {
        "DB_CONNECTION_STRING": "postgresql://localhost:5432/devdb"
      }
    }
  }
}
```

**Key points:**
- This file is in `~/.claude.json`, not in any project repo.
- Hardcoded credentials are acceptable here because this file is never committed.
- The `notion` server uses this developer's personal workspace token.
- The `db-explorer` is an experimental tool this developer is testing — not ready for the team.
- These tools are available in *every* project this developer opens, alongside any project-level tools.

---

### Example 3: MCP Resource Exposing Issue Summaries

An MCP server that exposes Jira sprint summaries as resources (read-only, no parameters needed):

```typescript
// In the MCP server implementation
server.setRequestHandler(ListResourcesRequestSchema, async () => {
  const sprints = await jira.getActiveSprints();
  return {
    resources: sprints.map(sprint => ({
      uri: `resource://jira/sprint/${sprint.id}/summary`,
      name: `Sprint ${sprint.name} Summary`,
      description: `Current status of sprint "${sprint.name}": ${sprint.issueCount} issues, ${sprint.completedCount} completed, ${sprint.inProgressCount} in progress.`,
      mimeType: "application/json"
    }))
  };
});

server.setRequestHandler(ReadResourceRequestSchema, async (request) => {
  const sprintId = extractSprintId(request.params.uri);
  const summary = await jira.getSprintSummary(sprintId);
  return {
    contents: [{
      uri: request.params.uri,
      mimeType: "application/json",
      text: JSON.stringify({
        sprintName: summary.name,
        startDate: summary.startDate,
        endDate: summary.endDate,
        issues: summary.issues.map(i => ({
          key: i.key,
          title: i.summary,
          status: i.status,
          assignee: i.assignee
        })),
        burndownPercent: summary.burndownPercent
      })
    }]
  };
});
```

**Why a resource and not a tool?**
- Sprint summaries are read-only — Claude does not need to modify them.
- No input parameters are needed — the resource URI identifies the sprint.
- The data is a browsable catalog — Claude can list available sprints and pick one.

---

### Example 4: Enhanced Tool Description Preventing Built-in Tool Preference

**Problem:** When an MCP server provides a `search_codebase` tool alongside Claude Code's built-in Grep, Claude tends to prefer the built-in tool because it has well-crafted descriptions. A minimal MCP tool description loses the competition.

**Bad — minimal description loses to built-in Grep:**

```json
{
  "name": "search_codebase",
  "description": "Searches the codebase."
}
```

Claude will almost always prefer its built-in Grep over this, because Grep's description is specific and trusted.

**Good — enhanced description explains unique value:**

```json
{
  "name": "search_codebase_semantic",
  "description": "Performs semantic code search using an embeddings index. Unlike regex/grep, this finds conceptually related code even when exact keywords differ. Input: {query: string, scope?: 'functions'|'classes'|'comments'|'all', top_k?: number (default 10)}. Returns: [{file_path, line_range, snippet, relevance_score}]. Use this when: (1) you don't know the exact function/variable name, (2) you want to find code that implements a concept (e.g., 'retry logic' finds backoff implementations even if they don't contain the word 'retry'), (3) you want ranked results by relevance. Use built-in Grep instead when you know the exact string to search for."
}
```

**Key techniques:**
- Renamed from `search_codebase` to `search_codebase_semantic` to signal differentiation.
- Description explicitly contrasts with Grep: "Unlike regex/grep..."
- Provides concrete use cases where this tool wins over the built-in.
- Tells Claude *when to use Grep instead* — this counterintuitively makes Claude more likely to use the MCP tool correctly, because it demonstrates the tool understands its own niche.

---

## Summary Checklist

- [ ] Project-shared servers are in `.mcp.json` (committed to repo)
- [ ] Personal servers are in `~/.claude.json` (never committed)
- [ ] All credentials use `${ENV_VAR}` expansion in `.mcp.json`
- [ ] Tool descriptions in MCP servers are detailed enough to compete with built-in tools
- [ ] Community servers are used when available; custom servers only when necessary
- [ ] Read-only content is exposed as MCP resources, not tools
- [ ] Tool names from MCP servers do not conflict with built-in tool names

---

## Related Topics

- [[Domain 2 - Tool Design & MCP Integration]] — domain overview
- [[Tool Interface Design]] — description quality for MCP-exposed tools
- [[Tool Distribution and Tool Choice]] — how MCP tools interact with tool_choice
- [[Built-in Tools]] — the tools MCP servers compete with
- [[Structured Error Responses]] — error patterns for MCP tool results
