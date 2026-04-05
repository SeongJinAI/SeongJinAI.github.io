---
title: "서버 에러 자동 핫픽스 시스템 — Sentry 감지부터 PR 생성까지"
date: 2026-04-05T00:00:00+09:00
draft: false
tags: ["Claude Code", "Sentry", "Slack", "GitHub Actions", "CI/CD"]
categories: ["AI Automation Pipeline"]
description: "서버 에러를 AI가 자동으로 감지하고, 유형별로 분류하여 DB SQL 또는 코드 수정 PR을 생성하는 엔드-투-엔드 핫픽스 자동화 시스템"
image: ""
---

## 문제 정의

건설 ERP 시스템을 운영하면서 서버 에러가 발생하면, 개발자는 다음과 같은 반복적인 사이클을 거칩니다.

```
Slack 알림 확인 → SSH 접속 → Docker 로그 분석 → 스택트레이스 해석
→ 코드베이스 탐색 → 원인 파악 → 코드 수정 → 커밋 → PR → 리뷰 → 머지 → 배포
```

에러 하나를 처리하는 데 **30분에서 수 시간**이 소요됩니다. 특히 DB 스키마 불일치 같은 단순한 문제도 동일한 프로세스를 거쳐야 하는 비효율이 존재합니다.

**이 파이프라인은 위 전체 프로세스를 AI가 자동으로 수행합니다.**

---

## 핵심 설계 — 에러 유형별 자동 분기

이 시스템의 가장 중요한 설계 결정은 **에러 유형에 따라 대응 경로를 자동으로 분류**하는 것입니다. 모든 에러를 동일하게 처리하지 않습니다.

| 유형 | 판별 기준 | 대응 | 개발자 행동 |
|------|----------|------|-----------|
| **DB 에러** | `Unknown column`, `Data too long`, 스키마 불일치 | Slack으로 ALTER SQL 직접 전달 | SQL 실행만 |
| **코드 에러** | NPE, RuntimeException, 로직 버그 | 자동 분석 → 수정 계획 → PR 생성 | 계획 승인 + PR 머지 |
| **혼합** | Entity↔DB 컬럼 불일치 | SQL(Slack) + PR(코드) 동시 | SQL 실행 + PR 머지 |

이 분기가 중요한 이유: **DB 에러는 코드를 아무리 수정해도 해결되지 않습니다.** 기존에는 개발자가 코드부터 확인하고, DB 문제임을 깨닫고, 다시 SQL을 작성하는 시간 낭비가 있었습니다. 이 시스템은 에러 메시지만으로 즉시 유형을 판별합니다.

---

## 시스템 아키텍처

> 아키텍처 다이어그램은 draw.io로 제작 예정입니다. 현재는 텍스트 기반으로 설명합니다.

### 전체 흐름

```
┌──────────────┐     ┌──────────┐     ┌──────────┐
│  ERP Server  │────→│  Sentry  │────→│  Slack   │
│  (EC2/Docker)│     │ (캡처)   │     │ (알림)   │
└──────────────┘     └──────────┘     └────┬─────┘
                                           │
                     ┌─────────────────────┘
                     ▼
         ┌───────────────────────┐
         │   Claude Code Agent   │
         │  (Scheduled Trigger)  │
         │                       │
         │  ① Sentry API 조회    │
         │  ② 스택트레이스 분석   │
         │  ③ 코드베이스 탐색     │
         │  ④ 에러 유형 분류      │
         └───────┬───────────────┘
                 │
        ┌────────┼────────┐
        ▼        ▼        ▼
   ┌────────┐ ┌──────┐ ┌──────┐
   │ DB     │ │ Code │ │Mixed │
   │ Error  │ │Error │ │      │
   └───┬────┘ └──┬───┘ └──┬───┘
       │         │         │
       ▼         ▼         ▼
   ┌────────┐ ┌──────┐ ┌──────────┐
   │ Slack  │ │  PR  │ │Slack+PR  │
   │SQL전달 │ │ 생성 │ │동시 처리  │
   └────────┘ └──┬───┘ └──────────┘
                 │
                 ▼
         ┌───────────────┐
         │  Codex Review  │
         │  (자동 리뷰)   │
         └───────┬───────┘
                 │
                 ▼
         ┌───────────────┐
         │ 개발자 최종    │
         │ 확인 + 머지    │
         └───────────────┘
```

### 구성 요소별 역할

| 구성 요소 | 역할 | 기술 |
|----------|------|------|
| **에러 수집** | 서버 예외 캡처 + 스택트레이스 보존 | Sentry |
| **알림** | 에러 발생 즉시 개발자 알림 + AI 분석 결과 전달 | Slack Webhook |
| **에러 분석** | 스택트레이스 파싱 → 코드베이스 탐색 → 원인 파악 | Claude Code (Opus) |
| **유형 분류** | 에러 메시지 패턴으로 DB/코드/혼합 자동 판별 | Rule-based + AI |
| **코드 수정** | 프로젝트 코드 스타일 준수하며 자동 수정 | Claude Code |
| **PR 관리** | 브랜치 생성 → 커밋 → PR 생성 → 리뷰 반영 | GitHub CLI |
| **코드 리뷰** | PR 생성 시 자동 리뷰 | GPT Codex (GitHub Actions) |
| **배포** | dev 브랜치 머지 → 자동 배포 | AWS ECR + EC2 Docker |

---

## 에러 유형별 상세 처리

### Type A: DB 에러 → Slack SQL 직접 전달

DB 레벨 문제는 코드 변경으로 해결되지 않습니다. AI가 Entity 정의와 실제 DB DDL을 비교 분석하여 정확한 ALTER SQL을 생성합니다.

**자동 판별 로직:**

```
에러 메시지 분석
  ├─ "Unknown column '{column}'" → Entity에서 @Column 확인 → DDL 비교 → ALTER TABLE 생성
  ├─ "Data too long for column" → 파라미터/컬럼 크기 비교 → 확장 SQL 생성
  └─ "Table '{table}' doesn't exist" → Entity 기반 CREATE TABLE 생성
```

**실제 운영 사례:**

ERP 시스템에서 증명서 발급 기능 배포 후, PDF 문서 URL 저장 컬럼이 DB에 미반영된 상태로 운영 투입되었습니다.

```
에러: SQLSyntaxErrorException
      Unknown column 'aih1_0.DocumentUrl' in 'SELECT'

AI 분석 (자동):
  ① AttestationIssueHistory Entity → @Column(name = "DocumentUrl") 확인
  ② src/sql/migration/증명서발급_DDL.sql → DocumentUrl 정의 확인
  ③ 실제 DB → 해당 컬럼 없음 (마이그레이션 미실행)

Slack 전달 (자동):
  ALTER TABLE AttestationIssueHistory
      ADD COLUMN DocumentUrl VARCHAR(500) NULL
      COMMENT '증명서 문서 URL' AFTER SealName;

소요 시간: 감지 후 2분 이내
```

개발자는 **전달받은 SQL만 실행**하면 즉시 해결됩니다.

### Type B: 코드 에러 → 분석 + 수정 + PR 자동 생성

코드 레벨 버그는 AI가 스택트레이스를 따라가며 원인을 파악하고, 수정 계획을 세운 후 개발자 승인을 받아 PR을 생성합니다.

**실제 운영 사례:**

직원 정보 수정 시 Employee→User 동기화가 실패하여 전체 트랜잭션이 롤백되는 장애가 발생했습니다.

```
에러: BusinessException (ERR_NOT_EXISTS_SYNCED_USER)
      Employee 수정 API 호출 시 발생

AI 분석 (자동):
  ① EmployeeUserSyncService.syncOnEmployeeUpdate() 추적
  ② 주민번호로 User 조회 → orElseThrow() → 실패
  ③ 원인: Employee 주민번호(하이픈 포함)와 User 주민번호(하이픈 미포함) 불일치
  ④ 부가 문제: 동기화 실패가 직원 수정 자체를 차단하는 구조

수정 계획 (Slack/Notion 공유):
  - old → new 주민번호 fallback 조회 추가
  - 못 찾으면 Sentry 알림 + return (직원 수정은 정상 진행)
  - User.syncFromEmployee에 주민번호 동기화 파라미터 추가

개발자 승인 후:
  → hotfix/employee-sync-jumin 브랜치 생성
  → 3개 파일 수정 (Service, Entity, ErrorCode)
  → PR 생성 → Codex 리뷰 → 머지

소요 시간: 분석 5분 + 승인 대기 + 수정/PR 3분
```

### Type C: 혼합 → SQL + PR 동시 처리

Entity에 필드가 정의되어 있지만 DB에 컬럼이 없는 경우, 양쪽 수정이 동시에 필요합니다.

```
Slack → ALTER TABLE SQL (DB 즉시 수정)
PR   → Entity/코드 추가 수정 (필요 시)
```

---

## 개발자 업무 개선 효과

### 정량적 개선

| 지표 | 기존 | 자동화 후 | 개선율 |
|------|------|----------|-------|
| DB 에러 평균 처리 시간 | 30분 (SSH→분석→SQL 작성) | **2분** (SQL 자동 전달) | **93% 단축** |
| 코드 에러 평균 처리 시간 | 1~2시간 | **10분** (분석+PR 자동) | **83~91% 단축** |
| 에러 원인 파악 정확도 | 개발자 경험 의존 | Entity↔DDL 자동 비교 | 오진 제거 |
| 야간/주말 초기 대응 | 다음 영업일 | **즉시** (Scheduled Trigger) | 대응 공백 제거 |

### 정성적 개선

- **개발자 컨텍스트 스위칭 최소화**: 에러 분석을 AI가 수행하므로, 개발자는 기능 개발에 집중
- **DB 에러 vs 코드 에러 자동 판별**: 잘못된 방향으로 디버깅하는 시간 제거
- **일관된 수정 품질**: 프로젝트 코드 스타일 가이드를 준수한 자동 수정
- **이력 자동 관리**: 모든 에러 분석/수정 이력이 Notion + Git에 자동 기록

---

## 개발자 인터랙션 — 최소 개입 설계

전체 파이프라인에서 **개발자가 직접 행동하는 지점은 3곳**뿐입니다.

| # | 지점 | 행동 | 소요 시간 |
|---|------|------|----------|
| 1 | 수정 계획 승인 | Slack 메시지 확인 → 승인/거부 | 1분 |
| 2 | DB SQL 실행 | 전달받은 SQL 복사 → DB 실행 | 1분 |
| 3 | PR 최종 머지 | Codex 리뷰 확인 → dev 머지 | 2분 |

나머지(감지, 분석, 유형 분류, 코드 수정, 브랜치 관리, PR 생성, 리뷰 반영)는 **전부 자동**입니다.

---

## 기술 스택

| 계층 | 도구 | 선택 이유 |
|------|------|----------|
| 에러 수집 | **Sentry** | 스택트레이스 보존, API 제공, 이슈 상태 관리 |
| 알림 | **Slack Webhook** | 개발자가 이미 상주하는 채널, 양방향 소통 가능 |
| AI 엔진 | **Claude Code (Opus)** | 100만 토큰 컨텍스트, 코드 분석/수정 동시 가능 |
| 주기적 실행 | **Scheduled Trigger** | Claude Code 내장 cron, 별도 인프라 불필요 |
| PR 관리 | **GitHub CLI** | 브랜치/커밋/PR을 CLI 한 줄로 처리 |
| 코드 리뷰 | **GPT Codex** | GitHub Actions 연동, PR 생성 시 자동 실행 |
| 배포 | **AWS ECR + EC2** | dev 머지 → Docker 자동 배포 (기존 파이프라인 활용) |

---

## 현재 한계 및 로드맵

### 현재 한계

| 한계 | 원인 | 영향 |
|------|------|------|
| 5~10분 감지 지연 | Scheduled Trigger 방식 (polling) | P0 에러에 즉시 대응 불가 |
| 승인 프로세스 수동 | Slack 메시지 확인 필요 | 개발자 부재 시 대기 |

### 로드맵

| 단계 | 개선 | 효과 |
|------|------|------|
| Phase 2 | Sentry Webhook → GitHub Actions → Claude Remote Trigger | **실시간 감지** (polling 제거) |
| Phase 3 | Slack Interactive Message 승인 버튼 | 세션 없이 승인 가능 |
| Phase 4 | 반복 에러 패턴 학습 → 자동 해결 (승인 없이) | 완전 자동화 |
