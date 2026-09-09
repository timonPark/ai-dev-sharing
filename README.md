# ai-dev-sharing

Claude Code 스킬 및 AI 개발 워크플로우 명세 모음입니다.

## 구성

### backend-feature-builder.md

신규 API · 동기화 로직 · 집계 · 연동 기능 구현을 위한 9단계 에이전트 워크플로우 명세입니다.

| 단계 | 스킬 | 내용 |
|---|---|---|
| Step 1 | `issue-kickoff` | 브랜치 생성 + Jira 이슈 조회 |
| Step 2 | GATE | 요구사항 4가지 확인 |
| Step 3 | `brainstorming` | 설계 탐색 및 합의 |
| Step 4 | `writing-plans` | 레이어별 구현 청사진 작성 |
| Step 5 | `tdd-service` | TDD 사이클 구현 |
| Step 6 | `codex-review` | AI 간 교차 코드 리뷰 |
| Step 7 | — | 리뷰 결과 반영 |
| Step 8 | `postman-doc` | API 문서 자동 생성 |
| Step 9 | `commit` / `pr` | 커밋 · 푸시 · PR 생성 |

진입점: `/work-manager {Jira 이슈 URL 또는 이슈키}`

---

### server-monitoring-connect-skill/

Grafana(Loki · Tempo · Prometheus) 연동 Claude Code 스킬 모음입니다.  
운영 서버의 로그 · 트레이스 · 메트릭을 CLI에서 조회합니다.

| 스킬 | 백엔드 | 용도 |
|---|---|---|
| `/prod-log` | Loki | production-1~4 로그 일괄 조회 |
| `/prod-metrics` | Prometheus | PromQL 메트릭 조회 |
| `/prod-trace` | Tempo | 특정 traceId 스팬 드릴다운 |
| `/prod-trace-export` | Tempo | 서비스명 · minDuration 기준 트레이스 추출 |
| `/tempo-search` | Tempo | TraceQL + 날짜 반복 조회 + CSV 생성 |

권장 사용 순서:
```
1. /prod-metrics      → 이상 구간 시간대 확인
2. /prod-log          → 해당 시간대 에러·슬로우 로그 확인
3. /prod-trace-export → 느린 트레이스 목록 추출
4. /prod-trace        → 특정 traceId 스팬 드릴다운
```

설정: `server-monitoring-connect-skill/config/env/.env`에 모니터링 서버 정보 입력
