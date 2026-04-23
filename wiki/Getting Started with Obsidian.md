---
tags: [setup, guide]
---

# How to View This Wiki in Obsidian

## What is Obsidian?

Obsidian is a free markdown-based knowledge management app that renders `[[wikilinks]]`, tags, and graph views — making it ideal for navigating interconnected study notes like this wiki.

## Installation

1. Download Obsidian from [https://obsidian.md](https://obsidian.md)
2. Install and open the app

## Opening This Wiki as a Vault

1. In Obsidian, click **"Open folder as vault"**
2. Navigate to and select the `wiki/` folder inside this repository:
   ```
   claudelearning/wiki/
   ```
3. Click **Open** — Obsidian will index all 43 markdown files

> **Important:** Open the `wiki/` folder specifically, not the repository root. This keeps the vault clean and ensures all relative links work correctly.

## Navigating the Wiki

### Start Here
- Open `00 - Home.md` from the file explorer on the left — this is the main landing page with the full learning path

### Key Navigation Features

| Feature | How to Use |
|---------|-----------|
| **Wikilinks** | Click any `[[blue link]]` to jump to that topic |
| **Back navigation** | `Cmd+Alt+Left` (Mac) / `Ctrl+Alt+Left` (Windows) to go back |
| **Quick switcher** | `Cmd+O` (Mac) / `Ctrl+O` (Windows) to search and open any note by name |
| **Search** | `Cmd+Shift+F` (Mac) / `Ctrl+Shift+F` (Windows) to search across all notes |
| **Graph view** | `Cmd+G` (Mac) / `Ctrl+G` (Windows) to see how all topics connect visually |
| **Tag search** | Click any `#tag` or use the tag pane to filter notes by domain |

### Recommended Layout

1. **Pin the Home page** — Right-click `00 - Home.md` tab → "Pin tab" so you always have the learning path accessible
2. **Split panes** — Drag a tab to the right side to view two notes side-by-side (e.g., a topic + its sample questions)
3. **Enable backlinks** — In the right sidebar, open the "Backlinks" panel to see which notes link to the current one

## Suggested Obsidian Settings

Open Settings (`Cmd+,` / `Ctrl+,`) and configure:

### Core Settings
- **Editor → Default editing mode:** Set to "Reading view" for distraction-free study, switch to "Live Preview" when you want to add your own notes

### Core Plugins (Enable These)
- **Graph view** — Visualize connections between topics and domains
- **Tag pane** — Browse notes by domain tags (`#domain/1`, `#domain/2`, etc.)
- **Outline** — See the heading structure of long notes in the right sidebar
- **Search** — Full-text search across all notes
- **Backlinks** — See which notes reference the current note

### Optional Community Plugins
- **Dataview** — Query notes by frontmatter fields (e.g., list all notes with `domain: 1`)
- **Calendar** — Track your daily study progress
- **Kanban** — Create a study board to track which topics you've mastered

## Study Workflow

### Phase-Based Approach
Follow the 5 phases in `00 - Home.md`:
1. Read each topic note thoroughly
2. Study the 3-4 examples in each note
3. Test yourself with [[Sample Questions]]
4. Complete the [[Exercise 1 - Multi-Tool Agent|hands-on exercises]]
5. Use Graph View to review connections between topics

### Adding Your Own Notes
- Create notes directly in the vault — they won't interfere with the study materials
- Use `[[wikilinks]]` to connect your notes to existing topics
- Add a `#personal` tag to distinguish your notes from the study content

### Tracking Progress
Add checkboxes to topic files as you study:
```markdown
- [x] Read through the full note
- [x] Understood all 4 examples
- [ ] Can explain the concept without referring to notes
- [ ] Completed related practice questions
```

## Folder Structure

```
wiki/
├── 00 - Home.md                          ← Start here
├── 01 - Agentic Architecture/            ← Domain 1 (27% of exam)
│   ├── Domain 1 - Agentic Architecture & Orchestration.md
│   ├── Agentic Loop Lifecycle.md
│   ├── Multi-Agent Orchestration.md
│   ├── Subagent Configuration and Context Passing.md
│   ├── Multi-Step Workflows and Handoffs.md
│   ├── Hooks and Interception.md
│   ├── Task Decomposition Strategies.md
│   └── Session Management.md
├── 02 - Tool Design & MCP/              ← Domain 2 (18% of exam)
│   ├── Domain 2 - Tool Design & MCP Integration.md
│   ├── Tool Interface Design.md
│   ├── Structured Error Responses.md
│   ├── Tool Distribution and Tool Choice.md
│   ├── MCP Server Configuration.md
│   └── Built-in Tools.md
├── 03 - Claude Code Configuration/       ← Domain 3 (20% of exam)
│   ├── Domain 3 - Claude Code Configuration & Workflows.md
│   ├── CLAUDE.md Configuration Hierarchy.md
│   ├── Custom Slash Commands and Skills.md
│   ├── Path-Specific Rules.md
│   ├── Plan Mode vs Direct Execution.md
│   ├── Iterative Refinement Techniques.md
│   └── CI-CD Integration.md
├── 04 - Prompt Engineering/              ← Domain 4 (20% of exam)
│   ├── Domain 4 - Prompt Engineering & Structured Output.md
│   ├── Explicit Criteria for Precision.md
│   ├── Few-Shot Prompting.md
│   ├── JSON Schemas and Tool Use.md
│   ├── Validation and Retry Loops.md
│   ├── Batch Processing.md
│   └── Multi-Instance Review Architectures.md
├── 05 - Context Management/              ← Domain 5 (15% of exam)
│   ├── Domain 5 - Context Management & Reliability.md
│   ├── Context Window Management.md
│   ├── Escalation and Ambiguity Resolution.md
│   ├── Error Propagation in Multi-Agent Systems.md
│   ├── Large Codebase Exploration.md
│   ├── Human Review Workflows.md
│   └── Information Provenance and Uncertainty.md
├── Sample Questions/
│   └── Sample Questions.md               ← All 12 practice questions
├── Exercises/
│   ├── Exercise 1 - Multi-Tool Agent.md
│   ├── Exercise 2 - Team Workflow Configuration.md
│   ├── Exercise 3 - Data Extraction Pipeline.md
│   └── Exercise 4 - Multi-Agent Research Pipeline.md
├── Technologies and Concepts Glossary.md
├── In-Scope vs Out-of-Scope Topics.md
└── Getting Started with Obsidian.md      ← This file
```

## Troubleshooting

| Issue | Fix |
|-------|-----|
| Links show as plain text | Make sure you opened the `wiki/` folder as the vault, not the repo root |
| Notes not appearing | Check that Obsidian has indexed all files (bottom-left shows note count — should be 44) |
| Graph view is empty | Click the refresh icon in the graph view panel |
| Tags not working | Ensure the Tag Pane core plugin is enabled in Settings → Core Plugins |
