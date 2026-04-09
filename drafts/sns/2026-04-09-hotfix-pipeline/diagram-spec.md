# Draw.io Diagram — Hotfix Pipeline Architecture

> 작성일: 2026.04.09
> 블로그 원본: `content/en/posts/2026-04-05-hotfix-pipeline.md`

---

## 노드 (위→아래 배치)

| # | 노드명 | 형태 | 비고 |
|---|--------|------|------|
| 1 | **Production Server (AWS)** | 컨테이너(큰 사각형) | "Vertical Container"에서 변경 |
| 2 | **Sentry** | 둥근 사각형 | 로고 자리 확보 |
| 3 | **Claude Code** | 큰 중앙 사각형 (강조) | 중앙 배치, 가장 큰 노드 |
| 4 | **DB** | 분기 박스 (좌) | 3분기 중 좌측 |
| 5 | **Code** | 분기 박스 (중) | 3분기 중 중앙 |
| 6 | **Mixed** | 분기 박스 (우) | 3분기 중 우측 |
| 7 | **Slack** | 결과 전달 (좌) | Fix SQL 전달용 |
| 8 | **GitHub** | 결과 전달 (중) | PR 생성용 |
| 9 | **Slack + GitHub** | 결과 전달 (우) | SQL + PR 동시 |
| 10 | **Developer** | 사람 아이콘 + 라벨 | 최하단, 최종 승인 게이트 |

## 화살표 라벨 (6개)

| # | 출발 → 도착 | 라벨 | 스타일 |
|---|-------------|------|--------|
| 1 | Production Server → Sentry | **Runtime Exception** | 실선 |
| 2 | Sentry → Claude Code | **Unresolved Issues** | 실선 |
| 3 | DB 분기 → Slack | **Fix SQL** | 실선, 좌측 |
| 4 | Code 분기 → GitHub | **Create PR** | 실선, 중앙 |
| 5 | 결과들 → Developer | **Notify Result** | 실선, 수렴 |
| 6 | Developer → (완료) | **Review & Merge** | 실선, 하단 |

## 레이아웃 (ASCII 스케치)

```
┌─────────────────────────────────┐
│    Production Server (AWS)      │
│    [Spring Boot App icon]       │
└───────────────┬─────────────────┘
                │ Runtime Exception
                ▼
         ┌─────────────┐
         │   Sentry    │
         └──────┬──────┘
                │ Unresolved Issues
                ▼
      ┌───────────────────┐
      │                   │
      │    Claude Code    │
      │    (AI Agent)     │
      │                   │
      └──┬──────┬──────┬──┘
         │      │      │
     ┌───┘      │      └───┐
     ▼          ▼          ▼
  ┌──────┐  ┌──────┐  ┌──────┐
  │  DB  │  │ Code │  │Mixed │
  └──┬───┘  └──┬───┘  └──┬───┘
     │ Fix     │ Create  │
     │ SQL     │ PR      │ Both
     ▼         ▼         ▼
 ┌───────┐ ┌───────┐ ┌──────────┐
 │ Slack │ │GitHub │ │Slack     │
 │       │ │       │ │+ GitHub  │
 └───┬───┘ └───┬───┘ └────┬─────┘
     │         │           │
     └────┬────┘───────────┘
          │ Notify Result
          ▼
   ┌──────────────┐
   │  Developer   │
   │ (Final Gate) │
   └──────┬───────┘
          │ Review & Merge
          ▼
      [Complete]
```

## 색상/스타일 제안

- Production Server: 회색 컨테이너
- Sentry: 보라/빨강 계열 (Sentry 브랜드)
- Claude Code: 주황/갈색 계열 (Anthropic 브랜드), 가장 큰 노드
- DB/Code/Mixed 분기: 각각 빨강/노랑/주황 (Slack 이모지 색상 매칭: 🔴🟡🟠)
- Developer: 초록 계열 (승인/안전)
- 화살표: 진한 회색, 라벨은 작은 글씨

## 로고 삽입 (수동)

직접 삽입할 로고 위치:
- Production Server 내부: Spring Boot / AWS 로고
- Sentry 노드: Sentry 로고
- Claude Code 노드: Anthropic 로고
- GitHub 노드: GitHub 로고
- Slack 노드: Slack 로고
