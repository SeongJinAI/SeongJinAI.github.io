---
title: "Overcoming AI Coding Agent's Context Limits with External Memory Architecture"
date: 2026-04-07T00:00:00+09:00
draft: false
tags: ["Claude Code", "Notion API", "Context Window", "Memory Architecture", "Productivity"]
categories: ["AI Automation Pipeline"]
description: "How I split Claude Code's finite context window into Notion (task state) + Handoff (technical context) + Memory (persistent references) — and made /notion pending retrieve all pending tasks in one command."
image: ""
---

## Problem Statement

When using AI coding agents (Claude Code, Cursor, etc.) for long-running projects, you hit the same wall every time a session changes:

```
Session 1: "Fixing payroll calculation logic... context full"
Session 2: "What did I do last time? Where did I leave off?"
Session 3: "Why did I modify this file? Do I need to re-analyze?"
```

**An AI agent's context window is finite.** No matter how large the model's context, a growing project eventually exceeds it. When a session ends, conversational context vanishes. The next session starts from a blank slate.

If you have to manually explain "last time I was working on X, got stuck at Y, modified Z for this reason..." every session, you're losing half the value of having an AI agent.

**This system lets the AI agent restore its own context across sessions.**

---

## Core Design — Separation of External Memory by Concern

Human memory isn't a single store either. Your to-do list, work notes, and contacts live in different places. AI agent external memory follows the same principle.

**Don't put everything in one file.** Choose the optimal store based on the nature of the information.

| Information Type | Store | Lifespan | Updated By |
|-----------------|-------|----------|------------|
| **Task list** (what to do?) | Notion | Until completion | Human + AI |
| **Technical context** (where did I leave off?) | Handoff files | Until next session | AI |
| **Persistent references** (Block IDs, API patterns) | Memory files | Semi-permanent | AI |
| **Auth credentials** (tokens, secrets) | .mcp.local.json | Semi-permanent | Human |

Why this separation matters: **each store has different update frequencies and access patterns.**

- Notion tasks change **daily**. Checked, added, modified.
- Handoff technical context is **overwritten per session**. End of session → start of next.
- Memory Block IDs **rarely change**. Permanent unless the Notion page structure changes.

Merging these into a single file means frequently-changing data mixes with stable data, making management difficult and forcing unnecessary information into context every session.

---

## System Architecture

> Architecture diagrams will be created with draw.io.

### Overview

```
┌─────────────────────────────────────────────────────┐
│                AI Agent (Claude Code)                │
│                                                     │
│  ┌─────────────────────────────────────────────┐    │
│  │         Context Window (finite)              │    │
│  │                                             │    │
│  │  Current conversation + code + tool results │    │
│  │  ← Can't fit everything here               │    │
│  └─────────────────────────────────────────────┘    │
│       │           │           │           │         │
│       ▼           ▼           ▼           ▼         │
│  ┌────────┐ ┌──────────┐ ┌────────┐ ┌──────────┐   │
│  │ Skill  │ │ Handoff  │ │ Memory │ │ Secret   │   │
│  │ System │ │ Files    │ │ Files  │ │ Store    │   │
│  └───┬────┘ └────┬─────┘ └───┬────┘ └────┬─────┘   │
└──────┼───────────┼───────────┼────────────┼─────────┘
       │           │           │            │
       ▼           ▼           ▼            ▼
  ┌─────────┐ ┌──────────┐ ┌────────┐ ┌──────────┐
  │ Command │ │ Technical│ │ Ref    │ │ Token    │
  │ Inter-  │ │ Context  │ │ Data   │ │ Mgmt     │
  │ face    │ │          │ │        │ │          │
  │ (what)  │ │ (where)  │ │ (how)  │ │ (auth)   │
  └─────────┘ └──────────┘ └────────┘ └──────────┘
       │                       │            │
       └───────────────────────┼────────────┘
                               ▼
                        ┌─────────────┐
                        │  Notion API │
                        │ (External   │
                        │  SSOT)      │
                        └─────────────┘
```

### Component Roles

| Component | Role | Location |
|-----------|------|----------|
| **Skill System** | Command definitions (add/done/list/pending) | `~/.claude/skills/` |
| **Handoff Files** | Technical context transfer between sessions | `handoff/{session}.md` |
| **Memory Files** | Persistent references (Block IDs, API patterns) | `~/.claude/projects/{project}/memory/` |
| **Secret Store** | Notion token (git-ignored) | `.claude/.mcp.local.json` |
| **Notion** | Task state SSOT (Single Source of Truth) | External SaaS |

---

## Why Notion — Requirements for an External SSOT

Notion was chosen as the AI agent's external memory because it's **"a SSOT that both humans and AI can use simultaneously."**

You could manage checklists in markdown files. But:

| Approach | Human Access | AI Access | SSOT |
|----------|-------------|-----------|------|
| Markdown files | Editor needed | ✅ Direct read/write | ❌ Git conflicts possible |
| Notion | ✅ Browser/app | API calls needed | ✅ Always current |
| Jira/Linear | ✅ Browser | API calls needed | ✅ Always current |

Notion was chosen because it was already the communication channel with project managers. The same principle applies to Jira, Linear, GitHub Issues, or any external tool.

The key isn't the tool — it's the separation principle: **"task state in external SSOT, technical context in local files."**

---

## Detailed Processing Flows

### Flow 1: Session Start — Automatic Context Recovery

When a new session begins, the AI agent combines two sources to understand the current state:

```
User: "Let's continue"
  │
  ├─ 1. Read handoff/main.md
  │     └─ "dev branch, payroll calc fix complete, Lambda deploy needed"
  │
  ├─ 2. Run /notion pending
  │     └─ "Payroll 7 items, HR 5 items, Infra 6 items pending"
  │
  └─ 3. Synthesize
        "Payroll calc is done. Next: payroll settlement salaryType validation.
         Lambda 4 files need AWS console update (not code work)."
```

**No human explanation needed.** The AI restores its own context.

### Flow 2: `/notion pending` — Pending Task Query

What happens internally when this command runs:

```
"/notion pending"
  │
  ├─ 1. Skill Loading
  │     └─ notion.md injected as prompt (command parsing rules)
  │
  ├─ 2. Memory Lookup
  │     └─ Block ID mappings from reference_notion_workspace.md
  │        ├─ Construction: 329c97b4...16e3
  │        ├─ Expenses: 329c97b4...9cb1
  │        ├─ Payroll: 329c97b4...2631
  │        └─ ... (8 menus)
  │
  ├─ 3. Token Retrieval
  │     └─ Read NOTION_TOKEN from .mcp.local.json
  │
  ├─ 4. First API batch (parallel, 8 menus)
  │     └─ GET /blocks/{toggle_id}/children
  │        → Filter to_do blocks where checked == false
  │
  ├─ 5. Second API batch (sub-toggles)
  │     └─ Recursive query for nested menu toggles
  │
  └─ 6. Output
        [pending] All pending items (31)
          📌 Construction (4)
            [ ] DB > [ConcretePouring] > Fix > ...
          📌 Payroll (8)
            [ ] API > [PayrollCalc] > Improve > ...
```

**Block IDs aren't searched every time.** Stored once in Memory, reused across all sessions.

### Flow 3: Session End — Context Preservation

```
User: "/clear"
  │
  ├─ 1. Update handoff/{session}.md
  │     ├─ Completed work
  │     ├─ Branch/build status
  │     ├─ Caveats
  │     └─ Next session start prompt
  │
  ├─ 2. Update handoff/INDEX.md
  │     └─ Last update timestamp
  │
  └─ 3. Release context
        "Handoff complete (session: main)"
```

**Pending tasks are NOT recorded in Handoff.** That's Notion's job. Handoff only contains "what you technically need to know."

---

## Multi-Session — Parallel Workstream Context Isolation

As projects grow, a single session can't handle everything. Feature development gets interrupted by urgent bugs, and infrastructure changes need separate attention.

```
handoff/
├── INDEX.md          ← Session routing rules
├── main.md           ← Feature development (dev branch)
├── hotfix.md         ← Urgent bugs (hotfix/* branches)
└── infra.md          ← Infrastructure/automation
```

Each session maintains **independent technical context**.

| Rule | Reason |
|------|--------|
| Write only to your own file | Prevent cross-session context pollution |
| Read-only access to other files | Allow referencing related work |
| Auto-routing by branch | `hotfix/*` → hotfix.md |

Without this structure, the AI agent falls into "I was fixing payroll, then had to handle a Sentry error, switched branches, came back, and now I don't know where I was..."

---

## When MCP Doesn't Work — Workaround Design

Claude Code uses MCP (Model Context Protocol) for external tool integration. But in reality, MCP doesn't always work.

**The actual problem:** Built-in Notion MCP connected to a different workspace's OAuth, making the work Notion inaccessible.

```
Ideal flow:
  Claude Code → MCP → Notion (OAuth) → ✅

Reality:
  Claude Code → MCP → Wrong workspace OAuth → ❌

Workaround:
  Claude Code → Skill (direct curl) → Notion API → ✅
```

Instead of depending on MCP, we use **curl + REST API + Integration Token** directly. When MCP is fixed, we can switch back, but the current approach is more reliable.

---

## Interface-Implementation Separation — Cross-Project Reuse

This system isn't tied to a specific project. **A governance repo defines the interface**, and each project overrides the implementation.

```
Governance Repo (Interface)              Project (Implementation)
┌─────────────────────┐          ┌──────────────────────┐
│ skills/              │          │ .claude/skills/       │
│   intg-notion.md     │ ──────→ │   notion.md           │
│   "add/done/list/    │ install │   "Menu→Block ID map  │
│    pending commands"  │          │    Token location"    │
└─────────────────────┘          └──────────────────────┘
```

To use this system in a new project:

1. Run `install.sh` → Copy skill interfaces
2. Write Block ID mappings in project `.claude/skills/notion.md`
3. Set Notion token in `.mcp.local.json`

**Command format, output style, and error handling are managed by governance** — no need to redefine per project.

---

## Developer Productivity Impact

### Quantitative Improvements

| Metric | Before | After | Improvement |
|--------|--------|-------|-------------|
| Session start context recovery | 5-10 min (manual explanation) | **30 sec** (`/notion pending`) | **90%+** |
| Task list check | Open Notion browser + scroll | **One command** | Instant |
| Session end handoff | Manual or skipped | **Auto-generated** (`/clear`) | No more gaps |
| First question in new session | "What was I doing?" | **Unnecessary** | Eliminated |

### Qualitative Improvements

- **Zero context waste** — No context tokens spent re-explaining previous work
- **Multi-session safety** — Feature dev ↔ hotfix switching without context bleed
- **PM communication integrated** — Same Notion page where PMs add requirements, AI checks completion

---

## Tech Stack

| Layer | Tool | Why |
|-------|------|-----|
| Task Management | **Notion** | Shared SSOT with PMs, REST API available |
| AI Agent | **Claude Code** | Skill/Memory/Hook extension system |
| Command Interface | **Skill (.md)** | Commands defined in markdown, prompt injection |
| Persistent Memory | **Memory (.md)** | Persists across conversations, auto-loaded |
| Session Context | **Handoff (.md)** | Per-session technical context files |
| Token Management | **.mcp.local.json** | git-ignored, project-local |
| Governance | **claude-dotfiles** | Multi-project interface management |

---

## Current Limitations & Roadmap

### Current Limitations

| Limitation | Cause | Impact |
|-----------|-------|--------|
| Sequential Notion API calls | Recursive sub-toggle queries needed | 8 menus + 6 sub-menus = 14 API calls |
| Manual Block ID mapping | Must update Memory when Notion structure changes | Manual work on menu add/remove |
| MCP not used | OAuth workspace mismatch | Working around with direct curl |

### Roadmap

| Phase | Improvement | Effect |
|-------|------------|--------|
| Phase 2 | Auto-discover Block IDs (page structure parsing) | Auto-adapt to menu changes |
| Phase 3 | Switch to native MCP when fixed | Remove curl workaround |
| Phase 4 | Bidirectional sync (code changes → auto-check Notion) | Remove manual `/notion done` |
