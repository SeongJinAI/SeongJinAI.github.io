# Threads Post — Hotfix Pipeline

> 작성일: 2026.04.09
> 분량: ~430자
> 첨부: 다이어그램 이미지

---

서버 에러 터지면 SSH 접속 → 로그 분석 → 원인 파악 → 수정 → PR...
매번 30분~수 시간 소요 😮‍💨

AI 핫픽스 파이프라인 만들었더니:

🔴 DB 에러 → SQL이 Slack으로 바로 도착 (30분 → 2분)
🟡 코드 에러 → AI가 PR 자동 생성 (1-2시간 → 10분)

개발자가 하는 건 딱 3가지:
✅ 수정 계획 승인
✅ SQL 실행
✅ PR 머지

나머지? 전부 자동 🤖

핵심은 에러 유형 자동 분류.
DB 문제인데 코드 뒤지는 삽질이 사라짐.

야간/주말도 즉시 감지 → 출근하면 이미 분석 완료.

Sentry + Claude Code + Slack + GitHub CLI

블로그에 상세 정리 👉 [링크]

#ClaudeCode #AIAutomation #DevOps #Sentry
