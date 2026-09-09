# CLAUDE.md

이 프로젝트는 운영 서버 모니터링을 위한 Claude Code 스킬 저장소입니다.  
Grafana(Loki · Tempo · Prometheus)와 연동해 로그·트레이스·메트릭을 조회합니다.

## 환경 정보

- **모니터링 서버 IP**: `config/env/.env`의 `MONITORING_SERVER_IP`
- **인증 토큰**: `config/env/.env`의 `MONITORING_API_TOKEN`
- **Loki**: `http://{IP}:3100` — 나노초 타임스탬프, Bearer token 인증
- **Tempo**: `http://{IP}:3200` — 초 단위 타임스탬프, Bearer token 인증
- **Prometheus**: `http://{IP}:9090` — 초 단위 타임스탬프, Bearer token 인증
- **OS**: macOS (타임스탬프 계산 시 `python3` 사용, `date -d` 사용 불가)

## 스킬 목록

| 스킬 | 백엔드 | 용도 |
|---|---|---|
| `/prod-log` | Loki | production-1~4 로그 일괄 조회 |
| `/prod-metrics` | Prometheus | PromQL 메트릭 조회 |
| `/prod-trace` | Tempo | 특정 traceId 스팬 드릴다운 |
| `/prod-trace-export` | Tempo | 서비스명·minDuration 기준 트레이스 목록 추출 |
| `/tempo-search` | Tempo | TraceQL + 날짜 반복 조회 + CSV 생성 |
| `/dcare-user-detail` | D.CARE Admin API | userIdx로 회원 상세 조회 |
| `/extract-access-token` | D.CARE Admin API + Loki | userIdx로 회원 accessToken 추출 |

## 타임스탬프 규칙

백엔드마다 단위가 다르므로 반드시 구분해서 사용한다.

| 백엔드 | 단위 | 생성 방법 |
|---|---|---|
| Loki | 나노초 | `int(time.time() * 1e9)` |
| Tempo | 초 | `int(time.time())` |
| Prometheus | 초 | `int(time.time())` |

## 결과 저장 경로

모든 조회 결과는 아래 경로에 저장하며 커밋하지 않는다.

```
prod_log/       ← Loki 로그 조회 결과
prod_trace/     ← Tempo 트레이스 조회 결과
prod_metrics/   ← Prometheus 메트릭 조회 결과
config/env/     ← 환경 변수 (IP, 토큰)
```

## 주의사항

- `tempo-search` 스킬은 반드시 `curl -G` GET 방식을 사용한다. POST 방식은 TraceQL 필터가 무시된다.
- 조회 결과가 0건이면 오류가 아니다. 시간 범위나 검색 조건을 조정해 재시도를 안내한다.
- `config/env/.env`는 `.gitignore`에 등록되어 있어 커밋되지 않는다.
