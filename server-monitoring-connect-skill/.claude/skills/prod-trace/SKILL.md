---
name: prod-trace
description: 운영 모니터링 서버(Tempo)에서 특정 API 요청의 트레이스를 조회한다. 스팬별 소요 시간을 분석해 병목 구간을 파악하고 결과를 prod_trace/{폴더명}/ 에 저장한다. 성능 분석 시 사용한다.
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

# prod-trace

운영 모니터링 서버(Tempo)에서 특정 API 요청의 트레이스를 조회한다.  
스팬별 소요 시간을 분석해 어느 구간(DB, Redis, 외부 API 등)에서 병목이 발생하는지 파악한다.  
성능 분석 체인(`03-performance-analyst`)의 4단계에서 사용한다.

## 사용법

```
/prod-trace
```

## 환경 정보

- **Tempo 주소**: `config/env/.env`의 `MONITORING_SERVER_IP` 값 사용 → `http://{IP}:3200`
- **인증**: Bearer token (`MONITORING_API_TOKEN`, `config/env/.env`에서 읽음)
- **저장 경로**: 프로젝트 루트 `prod_trace/` 폴더

## 프로세스

### 1단계: 필수 정보 수집

IP와 토큰을 먼저 읽는다.

```bash
MONITORING_IP=$(grep 'MONITORING_SERVER_IP' config/env/.env | cut -d'=' -f2 | tr -d ' ')
MONITORING_API_TOKEN=$(grep 'MONITORING_API_TOKEN' config/env/.env | cut -d'=' -f2 | tr -d ' ')
TEMPO_URL="http://${MONITORING_IP}:3200"
```

`AskUserQuestion`으로 아래 두 가지를 동시에 확인한다.

**조회 방식** 옵션:
- 특정 traceId로 조회 (traceId를 이미 알고 있는 경우)
- 서비스명 + 작업명으로 검색
- HTTP 경로로 검색 (예: /m/medication/list)
- 느린 트레이스 검색 (특정 응답시간 이상)

**시간 범위** 옵션:
- 최근 30분
- 최근 1시간
- 최근 3시간
- 직접 입력 (분 단위)

### 2단계: 저장 폴더 생성

```bash
DATETIME=$(date '+%Y-%m-%d-%H%M%S')
mkdir -p prod_trace/${DATETIME}-trace
```

### 3단계: 트레이스 검색 및 조회

#### 방식 A: traceId로 직접 조회

```bash
curl -s \
  -H "Authorization: Bearer ${MONITORING_API_TOKEN}" \
  "http://{MONITORING_SERVER_IP}:3200/api/traces/{traceId}" | \
  jq '.' > "prod_trace/{폴더명}/trace-{traceId}.json"
```

#### 방식 B: 태그 기반 검색 (HTTP 경로, 서비스명 등)

Tempo Search API를 사용한다. 반드시 `curl -G` GET 방식을 사용한다 (POST 방식은 필터가 무시됨).

```bash
curl -G -s \
  -H "Authorization: Bearer ${MONITORING_API_TOKEN}" \
  "http://{MONITORING_SERVER_IP}:3200/api/search" \
  --data-urlencode "tags=http.url={경로}" \
  --data-urlencode "minDuration=1000ms" \
  --data-urlencode "start=$START" \
  --data-urlencode "end=$END" \
  --data-urlencode "limit=20" | \
  jq '.' > "prod_trace/{폴더명}/search-result.json"
```

검색 결과에서 traceId 목록을 추출하고, 상위 5개를 개별 조회한다.

```bash
# search-result.json에서 traceId 추출 후 각각 조회
jq -r '.traces[].traceID' prod_trace/{폴더명}/search-result.json | head -5 | while read traceId; do
  curl -s \
    -H "Authorization: Bearer ${MONITORING_API_TOKEN}" \
    "http://{MONITORING_SERVER_IP}:3200/api/traces/${traceId}" | \
    jq '.' > "prod_trace/{폴더명}/trace-${traceId}.json"
done
```

### 4단계: 스팬 분석 요약

저장된 트레이스 파일을 읽어 스팬별 소요 시간을 분석한다.

분석 항목:
- 전체 요청 소요 시간
- 스팬별 소요 시간 (DB 쿼리, Redis, 외부 API, 내부 함수 등)
- 가장 오래 걸린 스팬 Top 5
- 병렬 vs 직렬 구간 파악

### 5단계: 결과 요약 출력

```
✅ 트레이스 조회 완료
- 조회 traceId 수: N건
- 저장 경로: prod_trace/{폴더명}/

병목 구간 요약:
- 전체 응답시간: Nms
- DB 쿼리 구간: Nms (전체의 N%)
- Redis 구간: Nms (전체의 N%)
- 가장 느린 스팬: {스팬명} Nms
```

## 규칙

- `start` / `end`는 **초 단위** Unix 타임스탬프 정수를 사용한다.
- Tempo Search API(방식 B)는 반드시 `curl -G` GET 방식을 사용한다. POST 방식은 필터가 무시된다.
- 트레이스 파일은 크기가 클 수 있으므로 상위 5건만 개별 조회한다.
- `prod_trace/` 폴더는 `.gitignore`에 등록한다.
- traceId를 직접 알고 있으면 방식 A, 모르면 방식 B로 진행한다.
