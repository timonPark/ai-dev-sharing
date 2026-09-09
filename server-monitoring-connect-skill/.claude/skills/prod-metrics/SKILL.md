---
name: prod-metrics
description: 운영 모니터링 서버(Prometheus)에서 특정 시간대의 메트릭을 조회한다. 애플리케이션·MongoDB·Redis 메트릭을 PromQL로 쿼리하고 결과를 prod_metrics/{폴더명}/ 에 저장한다. 성능 분석 시 사용한다.
allowed-tools:
  [
    'Bash(curl:*)',
    'Bash(python3:*)',
    'Bash(mkdir:*)',
    'Bash(jq:*)',
    'Bash(date:*)',
    'Read',
    'AskUserQuestion',
  ]
---

# prod-metrics

운영 모니터링 서버(Prometheus)에서 특정 시간대의 메트릭을 PromQL로 조회한다.  
성능 분석 체인(`03-performance-analyst`)의 3단계에서 사용한다.

## 사용법

```
/prod-metrics
```

## 환경 정보

- **Prometheus 주소**: `config/env/.env`의 `MONITORING_SERVER_IP` 값 사용 → `http://{IP}:9090`
- **인증**: Bearer token (`config/env/.env`의 `MONITORING_API_TOKEN`)
- **저장 경로**: 프로젝트 루트 `prod_metrics/` 폴더

## 프로세스

### 1단계: 필수 정보 수집

`AskUserQuestion`으로 아래 두 가지를 동시에 확인한다.

**시간 범위** 옵션:
- 최근 30분
- 최근 1시간
- 최근 3시간
- 최근 6시간
- 직접 입력 (분 단위)

**분석 대상** 옵션:
- 애플리케이션 메트릭 (HTTP 응답시간, 요청수, 에러율)
- MongoDB 메트릭 (쿼리 레이턴시, 연결수, 작업수)
- Redis 메트릭 (캐시 히트율, 메모리, 슬로우 커맨드)
- 전체 (위 세 가지 모두)

### 2단계: 저장 폴더 생성

```bash
DATETIME=$(date '+%Y-%m-%d-%H%M%S')
mkdir -p prod_metrics/${DATETIME}-metrics
```

### 3단계: 타임스탬프 계산

```bash
END=$(python3 -c "import time; print(int(time.time()))")
START=$(python3 -c "import time; print(int(time.time() - {초}))")
STEP="60"  # 1분 간격
```

### 4단계: PromQL 쿼리 실행

Prometheus `query_range` API를 사용한다. 토큰을 먼저 읽고 모든 curl에 Authorization 헤더를 포함한다.

```bash
TOKEN=$(grep MONITORING_API_TOKEN config/env/.env | cut -d= -f2)

curl -s "http://{MONITORING_SERVER_IP}:9090/api/v1/query_range" \
  -H "Authorization: Bearer $TOKEN" \
  --data-urlencode "query={PromQL}" \
  --data-urlencode "start=$START" \
  --data-urlencode "end=$END" \
  --data-urlencode "step=$STEP" | jq '.' > "prod_metrics/{폴더명}/{파일명}.json"
```

#### 애플리케이션 메트릭 쿼리

| 파일명 | PromQL | 설명 |
|---|---|---|
| `http_duration_p95.json` | `histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))` | 응답시간 P95 |
| `http_request_rate.json` | `rate(http_requests_total[5m])` | 분당 요청 수 |
| `http_error_rate.json` | `rate(http_requests_total{status=~"5.."}[5m])` | 5xx 에러율 |

#### MongoDB 메트릭 쿼리

| 파일명 | PromQL | 설명 |
|---|---|---|
| `mongo_op_latency.json` | `rate(mongodb_op_latencies_latency_total[5m]) / rate(mongodb_op_latencies_ops_total[5m])` | 작업별 평균 레이턴시 |
| `mongo_connections.json` | `mongodb_connections{state="current"}` | 현재 연결 수 |
| `mongo_document_ops.json` | `rate(mongodb_metrics_document_total[5m])` | 초당 도큐먼트 작업 수 |

#### Redis 메트릭 쿼리 (redis_exporter 연동 후 사용)

| 파일명 | PromQL | 설명 |
|---|---|---|
| `redis_hit_rate.json` | `rate(redis_keyspace_hits_total[5m]) / (rate(redis_keyspace_hits_total[5m]) + rate(redis_keyspace_misses_total[5m]))` | 캐시 히트율 |
| `redis_memory.json` | `redis_memory_used_bytes / redis_memory_max_bytes` | 메모리 사용률 |
| `redis_slowlog.json` | `redis_slowlog_length` | 슬로우 커맨드 수 |
| `redis_evictions.json` | `rate(redis_evicted_keys_total[5m])` | 초당 eviction 수 |

### 5단계: 결과 요약 출력

```
✅ 메트릭 조회 완료
- 시간 범위: {시작} ~ {종료}
- 저장 경로: prod_metrics/{폴더명}/
- 수집 항목: {목록}

주요 수치:
- HTTP P95 응답시간: Nms
- MongoDB 평균 레이턴시: Nms
- Redis 캐시 히트율: N%  (redis_exporter 연동 후 표시)
```

## 규칙

- `start` / `end`는 **초 단위** Unix 타임스탬프 정수를 사용한다. (Prometheus는 초 단위)
- Redis 메트릭은 `redis_exporter` 연동 후에만 조회 가능하다. 연동 전이면 해당 쿼리를 건너뛴다.
- `prod_metrics/` 폴더는 `.gitignore`에 등록한다.
