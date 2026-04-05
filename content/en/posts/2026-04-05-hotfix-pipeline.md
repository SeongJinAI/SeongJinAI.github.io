---
title: "Automated Hotfix Pipeline: Sentry Error Detection → Analysis → PR Generation"
date: 2026-04-05T00:00:00+09:00
draft: false
tags: ["Claude Code", "Sentry", "Slack", "GitHub Actions", "Automation", "DevOps"]
categories: ["AI Automation"]
description: "An automated hotfix pipeline where AI detects server errors from Sentry, classifies them by type (DB/Code/Mixed), and either delivers SQL fixes via Slack or generates code fix PRs"
---

## Overview

When a server error occurs, developers typically repeat this cycle:

1. Check Sentry/Slack alerts
2. SSH into server → Analyze Docker logs
3. Identify root cause → Fix code
4. Commit → PR → Review → Merge

**This pipeline automates the entire process with AI.**

### Before vs After

| Traditional Process | After Automation |
|-------------|----------|
| Check Slack → SSH → Log analysis → Root cause → Fix → PR | AI auto-detects from Sentry → Analyzes → Suggests fix → PR after approval |
| 30 min to hours per error | 5-10 min per error (excluding approval wait) |
| Check code first even for DB errors | **Auto-classifies error type** → DB errors get SQL directly, code errors get PR |

---

## Architecture

```
┌─────────────────────────────────┐
│    Server (AWS EC2 Docker)       │
│              │                   │
│        Error occurs              │
│              │                   │
│        ┌─────┴─────┐            │
│        │  Sentry   │            │
│        └─────┬─────┘            │
└──────────────┼──────────────────┘
               │ webhook
               ▼
       ┌──────────────┐
       │    Slack     │ ← Developer notification
       └──────┬───────┘
              │
              ▼
┌─────────────────────────────────────────┐
│  Claude Code (Scheduled Trigger)         │
│                                          │
│  ① Query Sentry API (new unresolved)     │
│  ② Analyze stacktrace + search codebase  │
│  ③ Classify error type                   │
│         │                                │
│    ┌────┼────────┐                       │
│    ▼    ▼        ▼                       │
│   DB  Code    Mixed                      │
│    │    │        │                       │
│    ▼    ▼        ▼                       │
│  Slack  PR    SQL+PR                     │
│  SQL   Create                            │
└─────────────────────────────────────────┘
              │
              ▼
     Developer final review + merge
              │
              ▼
     AWS ECR+EC2 auto-deploy
```

---

## Error Type Routing — Core Design Decision

The most critical design decision in this pipeline is **automatically classifying errors by type and routing them to different resolution paths**.

### Type A: DB Errors — SQL Delivered via Slack (No PR)

DB-level issues cannot be resolved with code changes. The AI compares Entity definitions with DB DDL, generates ALTER SQL, and delivers it directly via Slack.

**Detection criteria:**
- `SQLSyntaxErrorException` (Unknown column, Table doesn't exist)
- `Data too long for column`
- Stored procedure parameter overflow

**Real-world example:**

```
Error: Unknown column 'aih1_0.DocumentUrl' in 'SELECT'

AI Analysis:
- DocumentUrl field defined in Entity (AttestationIssueHistory.java:104)
- Column missing from actual DB table
- DDL file has the definition but DB migration was not applied

Slack Delivery:
ALTER TABLE AttestationIssueHistory
    ADD COLUMN DocumentUrl VARCHAR(500) NULL
    COMMENT 'Certificate document URL' AFTER SealName;
```

### Type B: Code Errors — Auto-fix + PR Generation

Code-level bugs go through: AI analysis → fix plan → developer approval → PR creation.

**Detection criteria:**
- `NullPointerException`
- `RuntimeException` (business logic bugs)
- `Sentry.captureException()` direct calls

**Real-world example:**

```
Error: ERR_NOT_EXISTS_SYNCED_USER (during Employee update)

AI Analysis:
- EmployeeUserSyncService.syncOnEmployeeUpdate() fails
  when looking up User by juminNumber → entire transaction rolls back
- Root cause: juminNumber format mismatch between Employee and User
  (with/without hyphen)

Fix:
- Add old → new juminNumber fallback lookup
- If not found: Sentry alert + return (employee update proceeds normally)
- Add juminNumber sync to User.syncFromEmployee

→ hotfix branch → PR created → Codex review → merge
```

### Type C: Mixed — SQL + PR Simultaneously

When Entity has a field but DB lacks the column, both sides need fixes.

```
Error: Unknown column 'ForeignName'

Response:
1. Slack → ALTER TABLE SQL (immediate DB fix)
2. PR → Additional Entity/code changes if needed
```

---

## Developer Interaction Points

Only **3 points** in the pipeline require developer intervention:

| # | Point | Action |
|---|-------|--------|
| 1 | **Approve fix plan** | Review plan posted to Slack/Notion, approve or reject |
| 2 | **Execute DB SQL** | For Type A errors, run the delivered SQL on database |
| 3 | **Final PR merge** | Merge the PR after Codex review is addressed |

Everything else is fully automated.

---

## Tech Stack

| Component | Tool | Role |
|-----------|------|------|
| Error Collection | Sentry | Server exception capture + stacktrace |
| Notifications | Slack Webhook | Error alerts + SQL/plan delivery |
| Task Tracking | Notion API | Team task management integration |
| Code Analysis/Fix | Claude Code (Opus) | AI-powered code analysis + auto-fix |
| Branch/PR | GitHub CLI (`gh`) | Branch creation, PR generation |
| Code Review | GPT Codex (GitHub Actions) | Automated PR review |
| Periodic Execution | Claude Code Scheduled Trigger | Periodic Sentry check |
| Deployment | AWS ECR+EC2 Docker | Auto-deploy on dev push |

---

## Session Management

### Hotfix Session Initial Briefing

When a Claude Code Remote session opens, it automatically assesses the current state:

```
Starting hotfix session.

🔴 DB fix required: 1 item
  [P1] DocumentUrl column missing — SQL delivered, pending execution

🟡 Code fix pending: 1 item
  [P2] Employee sync NPE — awaiting approval

🔄 Open PRs: 1
  #45 fix: attendance timeout — 2 Codex reviews unaddressed

Which item should we handle first?
```

### Multi-Session Architecture

| Session | Role |
|---------|------|
| main | Feature development |
| **hotfix** | **Error detection + fix automation (this pipeline)** |
| infra | Pipeline improvement |

---

## Limitations & Future Improvements

### Current Limitations
- Claude Code is not a persistent daemon — uses **Scheduled Trigger (periodic check)** approach
- Developer approval is manual within the session (no Slack button integration yet)
- 5-10 minute delay possible in Sentry → Slack → Claude path

### Future Improvements
- Sentry Webhook → GitHub Actions → Claude Remote Trigger for **real-time detection**
- Slack Interactive Messages for approve/reject buttons
- Error pattern learning → auto-resolve recurring errors (without approval)
