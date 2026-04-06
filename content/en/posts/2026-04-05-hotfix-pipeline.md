---
title: "Automated Server Error Hotfix System — From Sentry Detection to PR Generation"
date: 2026-04-05T00:00:00+09:00
draft: false
tags: ["Claude Code", "Sentry", "Slack", "GitHub Actions", "CI/CD"]
categories: ["AI Automation Pipeline"]
description: "An AI-powered system that automatically detects server errors, classifies them by type, and delivers DB fix SQL or generates code fix PRs"
image: ""
---

## Problem Statement

When a server error occurs in production, developers repeat the same cycle every time:

```
Check alert → SSH into server → Analyze logs → Identify cause → Fix → Test → PR → Review → Deploy
```

Each error takes **30 minutes to several hours** to resolve. The bigger problem: over half of all errors are simple issues like DB schema mismatches that **don't even require code changes** — yet they still go through the same process.

This system automates the entire process with AI.

---

## Core Design — Automatic Error Type Routing

Not all errors are treated the same. The system analyzes error messages to **automatically classify the type** and routes each to a completely different resolution path.

| Type | Detection Criteria | Response | Developer Action |
|------|-------------------|----------|-----------------|
| **DB Error** | Missing column, type mismatch, schema error | Fix SQL delivered via Slack | Execute SQL only |
| **Code Error** | NPE, logic bugs, validation gaps | AI fixes code and creates PR | Approve plan + merge PR |
| **Mixed** | Code and DB both misaligned | SQL (Slack) + PR (code) simultaneously | Execute SQL + merge PR |

Why this matters: **DB errors cannot be fixed by modifying code.** Previously, developers would check code first, then realize it's a DB issue — wasted time. This system identifies the type instantly from error message patterns.

---

## System Architecture

> Architecture diagrams will be created with draw.io.

### End-to-End Flow

```
┌──────────────┐     ┌──────────┐     ┌──────────┐
│   Server     │────→│  Sentry  │────→│  Slack   │
│              │     │ (capture)│     │ (alert)  │
└──────────────┘     └──────────┘     └────┬─────┘
                                           │
                     ┌─────────────────────┘
                     ▼
         ┌───────────────────────┐
         │   AI Agent            │
         │  (periodic monitor)   │
         │                       │
         │  ① Detect new errors  │
         │  ② Parse stacktrace   │
         │  ③ Search codebase    │
         │  ④ Classify type      │
         └───────┬───────────────┘
                 │
        ┌────────┼────────┐
        ▼        ▼        ▼
   ┌────────┐ ┌──────┐ ┌──────┐
   │   DB   │ │ Code │ │Mixed │
   └───┬────┘ └──┬───┘ └──┬───┘
       ▼         ▼         ▼
   ┌────────┐ ┌──────┐ ┌──────────┐
   │ Slack  │ │  PR  │ │Slack+PR  │
   │SQL fix │ │create│ │  both    │
   └────────┘ └──┬───┘ └──────────┘
                 ▼
         ┌───────────────┐
         │  Auto code    │
         │  review       │
         └───────┬───────┘
                 ▼
         ┌───────────────┐
         │  Developer    │
         │  final merge  │
         └───────────────┘
```

### Components

| Component | Role | Technology |
|-----------|------|-----------|
| Error Collection | Capture server exceptions, preserve stacktraces | Sentry |
| Notification | Instant alerts to developer + AI simultaneously | Slack Webhook |
| AI Analysis | Parse stacktrace → search code → identify cause → classify type | Claude Code |
| Code Fix | Auto-fix following project conventions | Claude Code |
| PR Management | Branch → commit → PR → address reviews | GitHub CLI |
| Code Review | Automated review on PR creation | GPT Codex |
| Deployment | dev merge → auto-deploy | Docker + CI/CD |

---

## Domain-Specialized Sub-Agents — Precision Through Delegation

A single AI agent analyzing the entire system leads to context waste and misdiagnosis. This system employs **domain-specialized sub-agents** that receive delegated analysis tasks for their area of expertise.

### Sub-Agent Routing

The main agent extracts the package path from the stacktrace and automatically dispatches to the appropriate domain sub-agent.

```
Main Agent (Hotfix Pipeline)
    │
    ├─ Extract domain from stacktrace
    │
    ├─ Dispatch to sub-agent ──→ Domain A Agent
    │                           "Analyze this error with your domain knowledge"
    │                              ├─ Entity relationship awareness
    │                              ├─ Business rule verification
    │                              ├─ Error pattern matching
    │                              └─ Detailed spec/architecture doc reference
    │
    ├─ Receive analysis result
    │
    └─ Classify type → DB/Code/Mixed → Respond
```

### Sub-Agent Package Structure

Domain context files are managed as a package inside the main agent. Each file contains the domain's **data models, business rules, API endpoints, error patterns, and technical documentation paths**.

```
hotfix-pipeline/
├── AGENT.md              ← Main (domain routing + error classification)
└── domains/
    ├── domain-a.md       ← Domain A context
    ├── domain-b.md       ← Domain B context
    ├── domain-c.md       ← Domain C context
    └── ...               ← Scales with system complexity
```

This structure provides **only the template (interface) from the governance repo** — each project fills in the actual domain knowledge. When applying to a new project, copy the package structure and write domain files.

### Why Sub-Agents?

| Single Agent | Sub-Agents |
|-------------|------------|
| Loads entire system into context | Loads only relevant domain knowledge |
| Risk of losing critical info at context limits | Focused analysis with full domain depth |
| Possible rule confusion between domains | Independent execution, no cross-domain confusion |
| Same analysis depth for all errors | **Domain-tailored analysis** |

In enterprise environments with multiple domains, each containing dozens of entities and business rules, accurate analysis without sub-agents is practically impossible.

---

## Error Type Handling

### Type A: DB Errors — SQL Delivered Directly (No Code Changes)

DB-level issues cannot be resolved with code changes. The AI compares application schema definitions against the actual DB state to generate precise fix SQL.

**Auto-Detection:**
- "Unknown column" → missing column
- "Data too long" → type/size mismatch
- "Table doesn't exist" → table not created

**Flow:**
```
Error detected → Schema definition vs DB comparison → Generate ALTER SQL → Deliver via Slack → Developer executes SQL
```

DB errors don't need PRs — **resolved with a single Slack message**. What used to take 30 minutes (SSH → log analysis → write SQL) now takes **under 2 minutes**.

### Type B: Code Errors — Auto-Analysis + PR Generation

For code-level bugs, the AI traces the stacktrace to identify the root cause, creates a fix plan, and generates a PR after developer approval.

**Flow:**
```
Error detected → Trace stacktrace → Analyze code → Create fix plan → Share via Slack
→ Developer approval → Create branch → Fix code → PR → Code review → Address reviews
→ Developer final merge
```

### Type C: Mixed — SQL + PR Simultaneously

When both application definitions and DB are misaligned, fixes are processed in parallel.

---

## Design Philosophy — SOTT (Scoped One-Time Task)

The fundamental principle: **AI analyzes and suggests, but the developer always makes the final decision.**

The AI doesn't receive unlimited authority. For each error, it receives **"only for this error, only within this scope"** — a limited, one-time authorization.

```
  Error occurs → AI auto-analysis (no permission needed, read-only)
                        ↓
             Developer approval (SOTT grant)
             "Fix this error using this approach"
             Scope: this specific error only
             Auth: hotfix branch only
             Expires: on PR creation
                        ↓
             AI auto-fix (within approved scope only)
                        ↓
             Developer final merge decision
```

### Why Not Full Automation?

DB SQL execution and code merges are **hard to reverse**. Even 99% AI accuracy means 1% misjudgment could corrupt production data.

- **Analysis and planning: automated** — AI's strongest domain
- **Execution decisions: human** — final gate for irreversible actions
- **Repetitive work: automated** — branch creation, commits, PR, review response

### Developer Intervention Points (Only 3)

| # | Point | SOTT Scope | Time |
|---|-------|-----------|------|
| 1 | **Approve fix plan** | Fix this error using this approach | 1 min |
| 2 | **Execute DB SQL** | Run this SQL on production DB | 1 min |
| 3 | **Final PR merge** | Deploy this code to production | 2 min |

Everything else is **fully automated**. The developer only **makes decisions** — AI handles the execution.

---

## Developer Productivity Impact

### Quantitative Improvements

| Metric | Before | After | Improvement |
|--------|--------|-------|------------|
| DB error avg. resolution | 30 min | **2 min** | **93%** |
| Code error avg. resolution | 1-2 hours | **10 min** | **83-91%** |
| Off-hours initial response | Next business day | **Immediate** | Response gap eliminated |

### Qualitative Improvements

- **Minimized context switching** — AI handles error analysis, developers stay focused on features
- **Automatic DB vs code classification** — eliminates debugging in the wrong direction
- **Consistent fix quality** — auto-fixes follow project code conventions
- **Automatic history management** — all analysis/fix records auto-logged

---

## Tech Stack

| Layer | Tool | Rationale |
|-------|------|-----------|
| Error Collection | **Sentry** | Stacktrace preservation, REST API |
| Notification | **Slack** | Developers already active here, bidirectional |
| AI Engine | **Claude Code** | Large context window, simultaneous analysis + modification |
| Periodic Execution | **Scheduled Trigger** | No additional infrastructure needed |
| PR Management | **GitHub CLI** | Automate branch/commit/PR via CLI |
| Code Review | **GPT Codex** | Auto-review on PR creation |
| Deployment | **Docker + CI/CD** | dev merge → auto build/deploy |

---

## Limitations & Roadmap

### Current Limitations

| Limitation | Cause | Impact |
|------------|-------|--------|
| 5-10 min detection delay | Polling approach (periodic API query) | Cannot respond instantly to P0 errors |
| Approval wait | Developer must manually review | Auto-fix delayed during absence |

### Roadmap

| Phase | Improvement | Effect |
|-------|------------|--------|
| Phase 2 | Webhook-based real-time detection | Eliminate polling, instant response |
| Phase 3 | Messenger approve/reject buttons | Approve without opening a session |
| Phase 4 | Error pattern learning → auto-resolve | Full automation without approval |
