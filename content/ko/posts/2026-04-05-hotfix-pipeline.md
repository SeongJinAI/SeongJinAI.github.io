---
title: "Sentry 에러 자동 감지 → 분석 → PR 생성 핫픽스 파이프라인"
date: 2026-04-05T00:00:00+09:00
draft: false
tags: ["Claude Code", "Sentry", "Slack", "GitHub Actions", "자동화", "DevOps"]
categories: ["AI Automation"]
description: "Sentry에서 감지된 서버 에러를 AI가 자동으로 분석하고, 에러 유형에 따라 DB SQL 전달 또는 코드 수정 PR을 생성하는 핫픽스 자동화 파이프라인"
---

## 개요

서버에서 에러가 발생하면 개발자는 다음 과정을 반복합니다:

1. Sentry/Slack 알림 확인
2. SSH 접속 → Docker 로그 분석
3. 원인 파악 → 코드 수정
4. 커밋 → PR → 리뷰 → 머지

**이 파이프라인은 1~4번 전체를 AI가 자동으로 처리합니다.**

### Before vs After

| 기존 프로세스 | 자동화 후 |
|-------------|----------|
| Slack 알림 확인 → SSH 접속 → 로그 분석 → 원인 파악 → 수정 → PR | AI가 Sentry에서 자동 감지 → 분석 → 해결책 제시 → 승인 후 PR |
| 에러당 30분~수시간 | 에러당 5~10분 (승인 대기 제외) |
| DB 에러도 코드부터 확인 | **에러 유형 자동 분류** → DB면 SQL 직접 전달, 코드면 PR |

---

## 아키텍처

```
┌─────────────────────────────────┐
│      서버 (AWS EC2 Docker)       │
│              │                   │
│         에러 발생                │
│              │                   │
│        ┌─────┴─────┐            │
│        │  Sentry   │            │
│        └─────┬─────┘            │
└──────────────┼──────────────────┘
               │ webhook
               ▼
       ┌──────────────┐
       │    Slack     │ ← 개발자 알림
       └──────┬───────┘
              │
              ▼
┌─────────────────────────────────────────┐
│  Claude Code (Scheduled Trigger, 주기적) │
│                                          │
│  ① Sentry API 조회 (새 미해결 이슈)      │
│  ② 스택트레이스 분석 + 코드베이스 탐색    │
│  ③ 에러 유형 분류                        │
│         │                                │
│    ┌────┼────────┐                       │
│    ▼    ▼        ▼                       │
│   DB  코드     혼합                      │
│    │    │        │                       │
│    ▼    ▼        ▼                       │
│  Slack  PR    SQL+PR                     │
│  SQL전달 생성                             │
└─────────────────────────────────────────┘
              │
              ▼
     개발자 최종 확인 + 머지
              │
              ▼
     AWS ECR+EC2 자동 배포
```

---

## 에러 유형별 분기 — 핵심 설계

이 파이프라인의 가장 중요한 설계 결정은 **에러 유형에 따라 대응 경로를 자동으로 분류**하는 것입니다.

### Type A: DB 에러 — PR 없이 SQL 직접 전달

DB 레벨 문제는 코드 수정으로 해결되지 않습니다. AI가 Entity와 DB DDL을 비교 분석하여 ALTER SQL을 생성하고, Slack으로 직접 전달합니다.

**판별 기준:**
- `SQLSyntaxErrorException` (Unknown column, Table doesn't exist)
- `Data too long for column`
- 프로시저 파라미터 오버플로우

**실제 사례:**

```
에러: Unknown column 'aih1_0.DocumentUrl' in 'SELECT'

AI 분석:
- Entity(AttestationIssueHistory.java:104)에 DocumentUrl 필드 정의됨
- DB 테이블에 해당 컬럼 없음
- DDL(증명서발급_DDL.sql:104)에는 정의되어 있으나 DB 미반영

Slack 전달:
ALTER TABLE AttestationIssueHistory
    ADD COLUMN DocumentUrl VARCHAR(500) NULL
    COMMENT '증명서 문서 URL' AFTER SealName;
```

### Type B: 코드 에러 — 자동 수정 + PR 생성

코드 레벨 버그는 AI가 분석 → 수정 계획 → 개발자 승인 → PR 생성까지 처리합니다.

**판별 기준:**
- `NullPointerException`
- `RuntimeException` (비즈니스 로직 버그)
- `Sentry.captureException()` 직접 호출

**실제 사례:**

```
에러: ERR_NOT_EXISTS_SYNCED_USER (Employee 수정 시)

AI 분석:
- EmployeeUserSyncService.syncOnEmployeeUpdate()에서
  주민번호로 User 조회 실패 → 전체 트랜잭션 롤백
- 원인: Employee와 User의 주민번호 형식 불일치 (하이픈 포함/미포함)

수정:
- old → new 주민번호 fallback 조회
- 못 찾으면 Sentry 알림 + return (직원 수정은 정상 진행)
- User.syncFromEmployee에 주민번호 동기화 추가

→ hotfix 브랜치 → PR 생성 → Codex 리뷰 → 머지
```

### Type C: 혼합 — SQL + PR 동시

Entity에 필드가 있지만 DB에 없는 경우 등 양쪽 수정이 필요합니다.

```
에러: Unknown column 'ForeignName'

대응:
1. Slack → ALTER TABLE SQL (DB 즉시 수정)
2. PR → 필요 시 Entity/코드 추가 수정
```

---

## 개발자 인터랙션 포인트

파이프라인에서 **개발자가 개입하는 지점**은 3곳뿐입니다:

| # | 지점 | 행동 |
|---|------|------|
| 1 | **수정 계획 승인** | Slack/Notion에 게시된 계획을 승인/거부 |
| 2 | **DB SQL 실행** | Type A 에러 시 전달받은 SQL을 DB에 직접 실행 |
| 3 | **최종 PR 머지** | Codex 리뷰 반영 완료된 PR을 dev에 머지 |

나머지는 전부 자동입니다.

---

## 기술 스택

| 구성 요소 | 도구 | 역할 |
|----------|------|------|
| 에러 수집 | Sentry | 서버 예외 캡처 + 스택트레이스 |
| 알림 | Slack Webhook | 에러 알림 + SQL/계획 전달 |
| 작업 기록 | Notion API | 팀 업무 관리 연동 |
| 코드 분석/수정 | Claude Code (Opus) | AI 기반 코드 분석 + 자동 수정 |
| 브랜치/PR | GitHub CLI (`gh`) | 브랜치 생성, PR 생성 |
| 코드 리뷰 | GPT Codex (GitHub Actions) | PR 자동 리뷰 |
| 주기적 실행 | Claude Code Scheduled Trigger | Sentry 주기적 체크 |
| 배포 | AWS ECR+EC2 Docker | dev push 시 자동 배포 |

---

## 세션 운영

### 핫픽스 세션 초기 브리핑

Claude Code Remote 세션이 열리면 자동으로 상태를 파악하여 브리핑합니다:

```
핫픽스 세션을 시작합니다.

🔴 DB 수정 필요: 1건
  [P1] DocumentUrl 컬럼 누락 — SQL 전달 완료, 적용 대기

🟡 코드 수정 대기: 1건
  [P2] Employee 동기화 NPE — 승인 대기

🔄 진행 중 PR: 1건
  #45 fix: attendance timeout — Codex 리뷰 2건 미반영

어떤 항목부터 처리할까요?
```

### 멀티 세션 구조

| 세션 | 역할 |
|------|------|
| main | 기능 개발 |
| **hotfix** | **에러 감지 + 수정 자동화 (이 파이프라인)** |
| infra | 파이프라인 자체 개선 |

---

## 한계 및 개선 방향

### 현재 한계
- Claude Code는 상시 데몬이 아니므로 **Scheduled Trigger(주기적 체크)** 방식 사용
- 개발자 승인은 세션 내에서 수동으로 진행 (Slack 버튼 연동 미구현)
- Sentry → Slack → Claude 경로에 5~10분 지연 가능

### 향후 개선
- Sentry Webhook → GitHub Actions → Claude Remote Trigger로 **실시간 감지**
- Slack Interactive Message로 승인/거부 버튼 추가
- 에러 패턴 학습 → 반복 에러 자동 해결 (승인 없이)
