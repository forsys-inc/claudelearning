# Claude Certified Architect – Foundations Study Guide

An Obsidian-based knowledge wiki covering the complete **Claude Certified Architect – Foundations** certification exam curriculum. Built from the official exam guide, organized topic-by-topic with examples, practice questions, and hands-on exercises.

## What's Inside

**44 interconnected notes** covering all 5 exam domains:

| Domain | Weight | Topics |
|--------|--------|--------|
| Agentic Architecture & Orchestration | 27% | Agentic loops, multi-agent orchestration, hooks, session management |
| Tool Design & MCP Integration | 18% | Tool descriptions, MCP servers, error responses, built-in tools |
| Claude Code Configuration & Workflows | 20% | CLAUDE.md hierarchy, slash commands, skills, plan mode, CI/CD |
| Prompt Engineering & Structured Output | 20% | Few-shot prompting, JSON schemas, validation loops, batch processing |
| Context Management & Reliability | 15% | Context windows, escalation, error propagation, human review |

Plus:
- **12 sample questions** with detailed answer explanations
- **4 hands-on exercises** with code templates
- **Glossary** of all tested technologies and concepts
- **Learning path** from basics to advanced topics

## How to View

### Option 1: Obsidian (Recommended)

1. Install [Obsidian](https://obsidian.md) (free)
2. Clone this repo:
   ```bash
   git clone https://github.com/forsys-inc/claudelearning.git
   ```
3. Open Obsidian → **Open folder as vault** → select the `wiki/` folder
4. Start at `00 - Home.md` for the guided learning path

Obsidian gives you clickable wikilinks, graph view showing topic connections, tag filtering by domain, and split panes for side-by-side study. See `wiki/Getting Started with Obsidian.md` for detailed setup instructions.

### Option 2: GitHub

Browse the `wiki/` folder directly on GitHub. Most markdown renders correctly, though `[[wikilinks]]` won't be clickable.

### Option 3: Any Markdown Editor

The notes are standard markdown files. Open the `wiki/` folder in VS Code, Typora, or any markdown viewer.

## Repository Structure

```
├── README.md
├── CLAUDE.md                                        # Context for Claude Code
├── claude architect exam official documentation.pdf  # Official exam guide
└── wiki/                                            # Obsidian vault
    ├── 00 - Home.md                                 # Start here
    ├── 01 - Agentic Architecture/                   # Domain 1 (8 notes)
    ├── 02 - Tool Design & MCP/                      # Domain 2 (6 notes)
    ├── 03 - Claude Code Configuration/              # Domain 3 (7 notes)
    ├── 04 - Prompt Engineering/                     # Domain 4 (7 notes)
    ├── 05 - Context Management/                     # Domain 5 (7 notes)
    ├── Sample Questions/                            # 12 practice questions
    ├── Exercises/                                   # 4 hands-on exercises
    ├── Technologies and Concepts Glossary.md
    ├── In-Scope vs Out-of-Scope Topics.md
    └── Getting Started with Obsidian.md
```

## Study Approach

Follow the 5-phase learning path in `00 - Home.md`:

1. **Foundations** — Agentic loops, built-in tools, CLAUDE.md, JSON schemas
2. **Architecture & Design** — Multi-agent patterns, tool design, MCP, few-shot prompting
3. **Advanced Patterns** — Hooks, task decomposition, context management, validation loops
4. **Production & Operations** — CI/CD, batch processing, human review, error propagation
5. **Practice & Review** — Sample questions and hands-on exercises
