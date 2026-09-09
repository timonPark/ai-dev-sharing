# server-monitoring-connect-skill

Grafana(Loki · Tempo · Prometheus) 연동 Claude Code 스킬 모음입니다.  
운영 서버의 로그·트레이스·메트릭을 CLI에서 조회하고 파일로 저장합니다.

## 환경 설정

`config/env/.env` 에 모니터링 서버 정보를 입력합니다.

```
MONITORING_SERVER_IP=
MONITORING_API_TOKEN=
DCARE_API_BASE_URL=
DCARE_ADMIN_ACCESS_TOKEN=
DCARE_CLIENT_KEY=
DCARE_MASTER_KEY=
DCARE_REST_API_KEY=
```

## 스킬 목록

### `/prod-log`
Loki에서 production-1~4 서버 로그를 일괄 조회합니다.
- userIdx / URI 경로 / 에러 메시지 / 키워드 검색
- 결과: `prod_log/{날짜-키워드}/production-{1~4}.json`

### `/prod-metrics`
Prometheus에서 PromQL 메트릭을 조회합니다.
- 애플리케이션 (HTTP P95 응답시간, 요청수, 에러율)
- MongoDB (쿼리 레이턴시, 연결수, 도큐먼트 작업)
- Redis (캐시 히트율, 메모리, 슬로우 커맨드, eviction)
- 결과: `prod_metrics/{날짜}-metrics/`

### `/prod-trace`
Tempo에서 특정 API 요청의 트레이스를 조회하고 스팬별 병목을 분석합니다.
- traceId 직접 조회 또는 HTTP 경로·서비스명으로 검색
- 결과: `prod_trace/{날짜}-trace/`

### `/prod-trace-export`
Tempo에서 서비스명·최소 소요시간 기준으로 트레이스 목록을 일괄 추출합니다.
- 결과: `prod_trace/{날짜}-export/search-result.json`, `traces-summary.json`

### `/tempo-search`
TraceQL(service.name + span name)로 날짜 범위를 반복 조회해 성능 분포를 분석합니다.
- 날짜 목록 쉼표 구분 입력, KST 시간 범위 지정
- 결과: `prod_trace/{폴더명}/{날짜}/` + `traces-timeline.csv`

## 권장 사용 순서

```
1. /prod-metrics      → 이상 구간 시간대 확인
2. /prod-log          → 해당 시간대 에러·슬로우 로그 확인
3. /prod-trace-export → 느린 트레이스 목록 추출
4. /prod-trace        → 특정 traceId 스팬 드릴다운
```

## 저장 경로

조회 결과는 아래 경로에 저장되며 `.gitignore`로 커밋에서 제외됩니다.

```
prod_log/
prod_trace/
prod_metrics/
config/env/
```
