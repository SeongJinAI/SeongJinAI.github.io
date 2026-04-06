---
title: "Automated Server Error Hotfix System — From Sentry Detection to PR Generation"
date: 2026-04-05T00:00:00+09:00
draft: false
tags: ["Claude Code", "Sentry", "Slack", "GitHub Actions", "CI/CD"]
categories: ["AI Automation Pipeline"]
description: "An end-to-end hotfix automation system where AI automatically detects server errors, classifies them by type, and generates DB SQL fixes or code PR accordingly"
image: ""
---

## Problem Statement

Operating a construction ERP system, every server error triggers a repetitive cycle for developers:

```
Check Slack alert → SSH into server → Analyze Docker logs → Parse stacktrace
→ Search codebase → Identify root cause → Fix code → Commit → PR → Review → Merge → Deploy
```

Each error takes **30 minutes to several hours** to resolve. Even simple issues like DB schema mismatches require the same full process — a significant inefficiency.

**This pipeline automates the entire process with AI.**

---

## Core Design — Automatic Error Type Routing

The most critical design decision is **automatically classifying errors by type and routing them to different resolution paths**. Not all errors are treated the same.

| Type | Detection Criteria | Response | Developer Action |
|------|-------------------|----------|-----------------|
| **DB Error** | `Unknown column`, `Data too long`, schema mismatch | ALTER SQL delivered via Slack | Execute SQL only |
| **Code Error** | NPE, RuntimeException, logic bugs | Auto-analysis → Fix plan → PR creation | Approve plan + Merge PR |
| **Mixed** | Entity↔DB column mismatch | SQL (Slack) + PR (code) simultaneously | Execute SQL + Merge PR |

Why this matters: **DB errors cannot be fixed by modifying code.** Previously, developers would check code first, realize it's a DB issue, then write SQL — wasted time. This system identifies the type instantly from the error message.

---

## System Architecture

> Architecture diagrams will be created with draw.io. Currently described in text format.

### End-to-End Flow

```
┌──────────────┐     ┌���─────────┐     ┌──────────┐
│  ERP Server  │────→│  Sentry  │────→│  Slack   │
│  (EC2/Docker)│     │ (capture)│     │ (alert)  ��
└────────────���─┘     └──────────┘     └────┬─────┘
                                           │
                     ┌─────────────────────┘
                     ▼
         ┌───────────────────────┐
         │   Claude Code Agent   │
         │  (Scheduled Trigger)  │
         │                       │
         │  ① Query Sentry API   │
         │  ② Parse stacktrace   │
         │  ③ Search codebase    │
         │  ④ Classify error     │
         └───────���───────────────┘
                 │
        ┌────────┼────────┐
        ▼        ▼        ▼
   ┌────────┐ ┌──────┐ ┌──────┐
   │   DB   │ │ Code │ │Mixed │
   ��� Error  │ │Error │ │      │
   └───┬────┘ └──┬─��─┘ └──┬───┘
       │         │         │
       ▼         ��         ▼
   ┌────────┐ ┌──────┐ ┌──────────┐
   │ Slack  │ │  PR  │ │Slack+PR  │
   │SQL fix │ │create│ │  both    │
   └────────┘ └──┬───┘ └──────────┘
                 │
                 ▼
         ┌───────────────┐
         │  Codex Review  │
         │  (auto-review) │
         └───��───┬───────┘
                 │
                 ▼
         ┌───────────────┐
         │   Developer   │
         │ final + merge │
         └───────────────┘
```

### Component Responsibilities

| Component | Role | Technology |
|-----------|------|-----------|
| **Error Collection** | Capture server exceptions + preserve stacktraces | Sentry |
| **Notification** | Instant developer alerts + AI analysis delivery | Slack Webhook |
| **Error Analysis** | Stacktrace parsing → codebase search → root cause | Claude Code (Opus) |
| **Type Classification** | Auto-detect DB/Code/Mixed from error patterns | Rule-based + AI |
| **Code Fix** | Auto-fix following project code style guide | Claude Code |
| **PR Management** | Branch → Commit → PR → Review response | GitHub CLI |
| **Code Review** | Automated PR review on creation | GPT Codex (GitHub Actions) |
| **Deployment** | dev merge → auto-deploy | AWS ECR + EC2 Docker |

---

## Detailed Error Type Handling

### Type A: DB Errors → SQL Delivered via Slack (No PR)

DB-level issues cannot be resolved with code changes. The AI compares Entity definitions against actual DB DDL to generate precise ALTER SQL.

**Auto-Detection Logic:**

```
Error message analysis
  ├─ "Unknown column '{col}'" → Check @Column in Entity → Compare DDL → Generate ALTER TABLE
  ├─ "Data too long for column" → Compare parameter/column sizes → Generate expansion SQL
  └─ "Table '{table}' doesn't exist" → Generate CREATE TABLE from Entity
```

**Production Case Study:**

After deploying a certificate issuance feature, the PDF document URL column was missing from the production database.

```
Error: SQLSyntaxErrorException
       Unknown column 'aih1_0.DocumentUrl' in 'SELECT'

AI Analysis (automatic):
  ① Entity: @Column(name = "DocumentUrl") — defined
  ② DDL file: DocumentUrl VARCHAR(500) — defined
  ③ Production DB: column missing — migration not executed

Slack Delivery (automatic):
  ALTER TABLE AttestationIssueHistory
      ADD COLUMN DocumentUrl VARCHAR(500) NULL
      COMMENT 'Certificate document URL' AFTER SealName;

Time to resolution: Under 2 minutes from detection
```

Developer simply **executes the delivered SQL** — immediate resolution.

### Type B: Code Errors → Analysis + Fix + Auto PR

For code-level bugs, the AI traces the stacktrace, identifies the root cause, creates a fix plan, and generates a PR after developer approval.

**Production Case Study:**

Employee information update was failing because Employee→User sync threw an exception, rolling back the entire transaction.

```
Error: BusinessException (ERR_NOT_EXISTS_SYNCED_USER)

AI Analysis (automatic):
  ① Traced EmployeeUserSyncService.syncOnEmployeeUpdate()
  ② User lookup by juminNumber → orElseThrow() → fails
  ③ Root cause: juminNumber format mismatch (with/without hyphen)
  ④ Secondary issue: sync failure blocks the employee update itself

Fix Plan (shared via Slack/Notion):
  - Add old → new juminNumber fallback lookup
  - If not found: Sentry alert + return (update proceeds normally)
  - Add juminNumber sync to User.syncFromEmployee

After approval:
  → Branch: hotfix/employee-sync-jumin
  → 3 files modified (Service, Entity, ErrorCode)
  → PR created → Codex review → merge

Time: 5min analysis + approval wait + 3min fix/PR
```

---

## Developer Productivity Impact

### Quantitative Improvements

| Metric | Before | After | Improvement |
|--------|--------|-------|------------|
| DB error avg. resolution | 30 min (SSH→analyze→write SQL) | **2 min** (SQL auto-delivered) | **93% reduction** |
| Code error avg. resolution | 1-2 hours | **10 min** (auto-analysis + PR) | **83-91% reduction** |
| Root cause accuracy | Depends on developer experience | Entity↔DDL auto-comparison | Misdiagnosis eliminated |
| Off-hours initial response | Next business day | **Immediate** (Scheduled Trigger) | Response gap eliminated |

### Qualitative Improvements

- **Minimized context switching**: AI handles error analysis, developers stay focused on feature development
- **Automatic DB vs Code classification**: Eliminates time wasted debugging in the wrong direction
- **Consistent fix quality**: Auto-fixes follow project code style guides
- **Automatic history management**: All analysis/fix records auto-logged in Notion + Git

---

## Design Philosophy — SOTT (Scoped One-Time Task)

The fundamental principle: **AI analyzes and suggests, but the developer always makes the final decision.**

The AI agent doesn't receive unlimited authority. For each detected error, it receives **"only for this error, only within this scope"** — a limited, one-time authorization. This is the SOTT (Scoped One-Time Task) pattern.

```
                    ┌──────────────┐
                    │ Error occurs  │
                    └──────┬───────┘
                           ▼
                    ┌──────────────┐
                    │ AI auto-     │ ← No permission needed (read-only)
                    │ analysis +   │
                    │ plan         │
                    └──────┬───────┘
                           ▼
                 ┌─────────────────────┐
                 │ Developer approval   │ ← "Fix THIS error only"
                 │ (SOTT grant)        │
                 │ Scope: single error  │
                 │ Auth: hotfix branch  │
                 │ Expires: on PR       │
                 └─────────┬───────────┘
                           ▼
                    ┌──────────────┐
                    │ AI auto-fix  │ ← Executes within approved scope only
                    │ branch + PR  │
                    └──────┬───────┘
                           ▼
                    ┌──────────────┐
                    │ Developer    │ ← Reviews code, decides to merge
                    │ final merge  │
                    └──────────────┘
```

### Why Not Full Automation?

DB SQL execution and code merges are **hard to reverse**. Even if AI is 99% accurate, the 1% misjudgment could corrupt production data. Therefore:

- **Analysis/planning is automated** — AI's strongest domain (pattern matching, code search)
- **Execution decisions are human** — final gate for irreversible actions
- **Repetitive work is automated** — branch creation, commits, PR, review response

### Developer Intervention Points (Only 3)

| # | Point | SOTT Scope | Time |
|---|-------|-----------|------|
| 1 | **Approve fix plan** | "Fix this error using this approach" | 1 min |
| 2 | **Execute DB SQL** | "Run this SQL on production DB" | 1 min |
| 3 | **Final PR merge** | "Deploy this code to dev" | 2 min |

Everything else (detection, analysis, classification, code fix, branch management, PR creation, review response) is **fully automated**. The developer only **makes decisions** — AI handles the execution.

---

## Tech Stack

| Layer | Tool | Rationale |
|-------|------|-----------|
| Error Collection | **Sentry** | Stacktrace preservation, API access, issue state management |
| Notification | **Slack Webhook** | Developers already active here, bidirectional communication |
| AI Engine | **Claude Code (Opus)** | 1M token context, simultaneous code analysis + modification |
| Periodic Execution | **Scheduled Trigger** | Built-in Claude Code cron, no additional infrastructure |
| PR Management | **GitHub CLI** | Branch/commit/PR in single CLI commands |
| Code Review | **GPT Codex** | GitHub Actions integration, auto-triggers on PR creation |
| Deployment | **AWS ECR + EC2** | dev merge → Docker auto-deploy (existing pipeline) |

---

## Limitations & Roadmap

### Current Limitations

| Limitation | Cause | Impact |
|------------|-------|--------|
| 5-10 min detection delay | Scheduled Trigger (polling) | Cannot respond instantly to P0 errors |
| Manual approval process | Requires Slack message review | Delayed when developer is unavailable |

### Roadmap

| Phase | Improvement | Effect |
|-------|------------|--------|
| Phase 2 | Sentry Webhook → GitHub Actions → Claude Remote Trigger | **Real-time detection** (eliminate polling) |
| Phase 3 | Slack Interactive Message approve/reject buttons | Approve without opening a session |
| Phase 4 | Error pattern learning → auto-resolve recurring errors | Full automation (no approval needed) |
