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
| Notification | **Slack Webhook** | Developers already active here, bidirectional |
| AI Engine | **Claude Code** | Large context window, simultaneous analysis + modification |
| Remote Monitoring | **Scheduled Trigger** (Anthropic Cloud) | 24/7 serverless, no infrastructure |
| Local Fallback | **Shell Script** + `/loop` | 10-min interval, supplementary during sessions |
| PR Management | **GitHub CLI** | hotfix branch → dev PR automation |
| Code Review | **GPT Codex** | Auto-review on PR creation |
| Deployment | **Docker + CI/CD** | dev merge → auto build/deploy |

---

## Dual Monitoring — Remote Sentinel + Local Executor

Error monitoring doesn't rely on a single point. **Remote (24/7) and local (during session)** run in parallel.

### Remote Trigger (Anthropic Cloud)

Every hour, a fresh sandbox is created, the repo is cloned from GitHub, and the prompt runs. **Same concept as AWS Lambda** — serverless, stateless, no memory between runs.

```
Every hour:
  New sandbox → GitHub clone → Sentry API query
  → Error found → Read domain sub-agent files → Analyze
  → DB error: Send SQL to Slack
  → Code error: Create hotfix branch → PR → Send PR link to Slack
  → Destroy sandbox
```

**Business hours policy**: Weekdays 09:00-18:00 KST — developer monitors directly, remote auto-skips.

### Local Fallback (Developer PC)

```bash
/loop 10m .claude/scripts/sentry-check.sh
```

Checks Sentry every 10 minutes while the session is open. Unlike remote, it has **DB access, local files, and instant agent invocation**.

| | Remote (Sentinel) | Local (Executor) |
|--|:----------------:|:----------------:|
| Interval | 1 hour | 10 min |
| Persistence | 24/7 | Session only |
| Sentry analysis | ✅ | ✅ |
| Domain knowledge | ✅ (GitHub clone) | ✅ (local files) |
| Slack delivery | ✅ | ✅ |
| PR creation | ✅ (hotfix→dev) | ✅ (hotfix→dev) |
| DB access | ❌ | ✅ |

### Future: Push-Based Instant Detection

Currently polling-based. For instant detection: **Sentry Webhook → AWS Lambda → Claude RemoteTrigger run API**. Lambda acts as a bridge between Sentry's HTTP request and Claude's remote agent. Requires additional infrastructure — planned for next phase.

---

## Operational Policies — Rules the Agent Follows

The remote trigger runs in an **isolated, stateless environment** every time. It retains no memory between runs. Stable operation in this environment requires explicit policies.

### Prompt-Document Separation Architecture

Instead of stuffing all rules into the trigger prompt (requiring API calls for every change), the prompt only says **"read AGENT.md and follow it"**. Detailed policies live in version-controlled markdown files.

```
trigger-prompt.md (prompt source, git-tracked)
    │
    └─ "Read AGENT.md first"
         │
         ├─ Mandatory execution rules
         ├─ Error classification policy
         ├─ Slack message templates
         ├─ Deduplication policy
         └─ domains/*.md (domain knowledge)
```

Policy changes require **only editing md files in the repo** — next execution picks them up automatically. No trigger redeployment needed.

### 9-Step Execution Checklist

The agent must follow this sequence for every issue:

```
□ 1. Query Sentry — unresolved + unassigned + prod
□ 2. Get latest event
□ 3. Extract package path from stacktrace
□ 4. Classify error type (decision tree)
□ 5. Read matching domain md file ← mandatory
□ 6. Analyze root cause using domain context
□ 7. Execute response (DB→SQL / CODE→PR / MIXED→both)
□ 8. Send Slack message (apply template)
□ 9. Assign Sentry issue (prevent duplicates)
```

Step 5 (reading domain md) is critical. Each domain file contains entity relationships, business rules, and common error patterns — accurate root cause analysis requires this context.

### Error Classification Decision Tree

```
Error message analysis
│
├─ SQL keywords? ("Unknown column", "Data too long", "Table doesn't exist")
│   ├─ Entity @Column exists + DB column missing → MIXED
│   └─ Other → DB
│
├─ DataIntegrityViolationException?
│   ├─ NOT NULL / UNIQUE violation → CODE (validation gap)
│   └─ Column type mismatch → DB
│
├─ NullPointerException / RuntimeException → CODE
│
└─ Other → CODE (default)
```

### Slack Message Templates

Each error type has a fixed format. Required fields: **severity, Sentry ID, original error message, root cause analysis, solution**.

| Type | Emoji | Key Content |
|------|:-----:|------------|
| DB | 🔴 | Fix SQL + Entity/DDL reference |
| CODE | 🟡 | Root cause + PR link + modified files |
| MIXED | 🟠 | SQL (immediate) + PR (if needed) |

### Deduplication — Sentry Assign Strategy

In a stateless environment, reporting the same issue every hour creates noise. We use **Sentry's assign feature** as external state storage.

```
Query: is:unresolved + !is:assigned + environment:prod
       → Returns only unprocessed issues

Process: Analyze → Send Slack → Create PR

Complete: PUT /issues/{id}/ {"assignedTo": "..."}
          → Excluded from next run

Recurrence: Developer resolves → Error recurs → Sentry auto-reopens
            → unresolved + unassigned → Detected again
```

This creates a **natural circulation** even in a stateless serverless environment.

---

## Implementation Status

| Component | Status | Notes |
|-----------|:------:|-------|
| Sentry API integration | ✅ | prod environment filter, Internal Integration Token |
| Slack Webhook | ✅ | Direct SQL delivery for DB errors |
| GitHub CLI auth | ✅ | Auto-create hotfix branch → dev PR |
| Domain sub-agents (7) | ✅ | HR, payroll, attendance, construction, resource, material, contract |
| Scheduled Trigger | ✅ | Hourly, auto-skip during business hours |
| Branch protection | ✅ | Block direct push to dev/main, only hotfix→dev PR allowed |
| Execution rules | ✅ | 9-step checklist, mandatory file references |
| Classification policy | ✅ | DB/CODE/MIXED decision tree |
| Slack templates | ✅ | 3 types with required fields |
| Deduplication | ✅ | Sentry assign strategy |
| Prompt-document separation | ✅ | trigger-prompt.md + AGENT.md delegation |

### Branch Protection — Safety Net for Auto-Fixes

When AI auto-fixes code, the most important thing is **limiting the blast radius of mistakes**.

```
Allowed: hotfix/{description} branch → dev PR
Blocked: Direct push to dev/main
Blocked: dev → main PR creation
```

This is **enforced at 3 levels**:
1. `CLAUDE.md` — AI aware of the rule
2. `settings.json` Hook — Tool-level execution blocked
3. Memory — Rule persists across sessions

No matter how confident the AI is in its fix, changes won't reach dev without human review through a PR.

---

## Limitations & Roadmap

### Current Limitations

| Limitation | Cause | Impact |
|------------|-------|--------|
| 10-min detection delay | Polling (Scheduled Trigger) | Cannot respond instantly to P0 errors |
| Approval wait | Developer must manually review | Auto-fix delayed during absence |

### Roadmap

| Phase | Improvement | Effect |
|-------|------------|--------|
| Phase 2 | Webhook-based real-time detection | Eliminate polling, instant response |
| Phase 3 | Messenger approve/reject buttons | Approve without opening a session |
| Phase 4 | Error pattern learning → auto-resolve | Full automation without approval |
