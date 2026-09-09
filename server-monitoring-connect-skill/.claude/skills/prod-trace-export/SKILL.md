---
name: prod-trace-export
description: Grafana Tempo에서 서비스명·최소 소요시간 기준으로 트레이스 목록을 검색하고 결과를 파일로 저장한다. Grafana Explore UI의 Tempo Search 탭 결과를 CLI로 추출할 때 사용한다.
allowed-tools:
  [
    'Bash(curl:*)',
    'Bash(python3:*)',
    'Bash(mkdir:*)',
    'Bash(jq:*)',
    'Bash(date:*)',
    'Bash(grep:*)',
    'Read',
    'AskUserQuestion',
  ]
---

# prod-trace-export

Grafana Explore → Tempo → Search 탭에서 서비스명과 Duration을 지정했을 때 나오는 트레이스 목록을 CLI로 추출하고 파일로 저장한다.

`prod-trace` 스킬이 특정 API 1개의 **스팬 드릴다운**(병목 분석)을 목적으로 한다면, 이 스킬은 서비스 전체에서 조건에 맞는 **트레이스 목록 일괄 추출**을 목적으로 한다.

## 사용법

```
/prod-trace-export
```

## 환경 정보

- **Tempo 주소**: `config/env/.env`의 `MONITORING_SERVER_IP` 값 사용 → `http://{IP}:3200`
- **인증**: Bearer token (`MONITORING_API_TOKEN`, `config/env/.env`에서 읽음)
- **저장 경로**: 프로젝트 루트 `prod_trace/` 폴더 (`.gitignore` 등록됨)

## 프로세스

### 1단계: MONITORING_SERVER_IP 읽기 및 서비스 목록 조회

`AskUserQuestion` 호출 **전에** 먼저 IP·토큰을 읽고 Tempo에서 실제 서비스 목록을 가져온다.

```bash
MONITORING_IP=$(grep 'MONITORING_SERVER_IP' config/env/.env | cut -d'=' -f2 | tr -d ' ')
MONITORING_API_TOKEN=$(grep 'MONITORING_API_TOKEN' config/env/.env | cut -d'=' -f2 | tr -d ' ')
TEMPO_URL="http://${MONITORING_IP}:3200"
```

서비스명 목록 조회:

```bash
curl -s \
  -H "Authorization: Bearer ${MONITORING_API_TOKEN}" \
  "${TEMPO_URL}/api/search/tag/service.name/values"
# 응답 예: { "tagValues": ["application-server", "another-service"] }
```

조회 결과를 파싱해 서비스명 배열을 만든다. 조회 실패 시 오류를 출력하고 중단한다.

### 2단계: 필수 정보 수집

1단계에서 얻은 서비스명 목록을 `AskUserQuestion` 옵션으로 제시한다. 서비스명·최소 소요시간·시간 범위를 동시에 확인한다.

**서비스명**: 1단계에서 조회한 `tagValues` 배열을 그대로 옵션으로 사용한다.
- 예: `application-server`, `another-service` … (조회된 값이 곧 선택지)
- `AskUserQuestion`의 옵션 개수 상한(4개)을 초과하면 상위 3개를 옵션으로 제시하고 나머지는 Other(직접 입력)로 안내한다.

**최소 소요시간 (minDuration)** 옵션:
- 500ms
- 1s
- 2s
- 5s

**시간 범위** 옵션:
- 최근 30분
- 최근 1시간
- 최근 3시간
- 직접 입력 (분 단위)

선택 사항: 최대 결과 수 (기본값 100, 최대 1000)

### 3단계: 저장 폴더 생성

```bash
DATETIME=$(date '+%Y-%m-%d-%H%M%S')
FOLDER="prod_trace/${DATETIME}-export"
mkdir -p "$FOLDER"
```

### 4단계: 타임스탬프 계산

macOS 환경이므로 `python3`으로 초 단위 Unix 타임스탬프를 생성한다.

```bash
END=$(python3 -c "import time; print(int(time.time()))")
START=$(python3 -c "import time; print(int(time.time()) - {초})")
```

시간 범위 → 초 변환:
- 30분 → 1800
- 1시간 → 3600
- 3시간 → 10800

### 5단계: Tempo Search API 호출

Tempo `/api/search` 엔드포인트를 사용한다. 반드시 `curl -G` GET 방식을 사용한다 (POST 방식은 필터가 무시됨).

```bash
curl -G -s \
  -H "Authorization: Bearer ${MONITORING_API_TOKEN}" \
  "${TEMPO_URL}/api/search" \
  --data-urlencode "tags=service.name={서비스명}" \
  --data-urlencode "minDuration={minDuration}" \
  --data-urlencode "start=${START}" \
  --data-urlencode "end=${END}" \
  --data-urlencode "limit={limit}" \
  > "${FOLDER}/search-result.json"
```

**응답 구조 예시:**
```json
{
  "traces": [
    {
      "traceID": "abc123...",
      "rootServiceName": "application-server",
      "rootTraceName": "POST /m/medication",
      "startTimeUnixNano": "1234567890000000000",
      "durationMs": 1543
    }
  ],
  "metrics": {
    "inspectedTraces": 500
  }
}
```

API 호출 실패(HTTP 오류, 빈 응답 등) 시 오류 메시지를 출력하고 중단한다.

### 6단계: 요약 파일 생성

원본 JSON에서 핵심 필드만 추출해 `traces-summary.json`을 생성한다.

```bash
python3 -c "
import json, sys

with open('${FOLDER}/search-result.json') as f:
    data = json.load(f)

traces = data.get('traces', [])
metrics = data.get('metrics', {})

summary = {
    'total': len(traces),
    'inspectedTraces': metrics.get('inspectedTraces', 0),
    'minDuration': '{minDuration}',
    'serviceName': '{서비스명}',
    'timeRange': '{시간 범위}',
    'exportedAt': '$(date)',
    'traces': [
        {
            'traceID': t.get('traceID', ''),
            'rootTraceName': t.get('rootTraceName', ''),
            'rootServiceName': t.get('rootServiceName', ''),
            'durationMs': t.get('durationMs', 0),
            'startTime': t.get('startTimeUnixNano', '')
        }
        for t in sorted(traces, key=lambda x: x.get('durationMs', 0), reverse=True)
    ]
}

with open('${FOLDER}/traces-summary.json', 'w') as f:
    json.dump(summary, f, ensure_ascii=False, indent=2)

print(f'총 {len(traces)}건')
"
```

### 7단계: 결과 요약 출력

```
✅ 트레이스 목록 추출 완료
- 서비스명: {서비스명}
- 최소 소요시간: {minDuration}
- 시간 범위: 최근 {N}분
- 추출 건수: {N}건 / 검사된 트레이스: {N}건
- 저장 경로: {FOLDER}/

상위 10건 (소요시간 내림차순):
┌─────────────────────────────────────────┬────────────┐
│ rootTraceName                           │ durationMs │
├─────────────────────────────────────────┼────────────┤
│ POST /m/medication/list                 │  4,521 ms  │
│ ...                                     │  ...       │
└─────────────────────────────────────────┴────────────┘

파일:
- search-result.json : Tempo API 원본 응답 전체
- traces-summary.json: 핵심 필드 추출 + 소요시간 내림차순 정렬
```

## 규칙

- `start` / `end`는 **초 단위** Unix 타임스탬프 정수를 사용한다 (`prod-trace`와 동일).
- Tempo Search API는 반드시 `curl -G` GET 방식을 사용한다. POST 방식은 필터가 무시된다.
- `prod_trace/` 폴더는 `.gitignore`에 등록되어 있어 커밋되지 않는다.
- 결과가 0건이면 오류가 아님을 알리고, 시간 범위 또는 minDuration을 조정해 재시도를 안내한다.
- `limit` 기본값은 100, 최대 1000. 1000건 초과 시 Tempo API 제한으로 잘릴 수 있음을 안내한다.
- `prod-trace`와의 차이: 이 스킬은 트레이스 **목록 추출**이 목적이며 스팬 드릴다운은 수행하지 않는다. 특정 traceID의 스팬 분석이 필요하면 `/prod-trace`를 사용한다.
