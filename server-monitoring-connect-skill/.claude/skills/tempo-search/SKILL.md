---
name: tempo-search
description: Grafana Tempo에서 TraceQL 조건(service.name + span name)과 날짜·시간 범위를 지정하여 트레이스 목록을 날짜별로 일괄 추출하고 traces-timeline.csv를 생성한다. 특정 날짜 범위의 API 성능 분포를 분석할 때 사용한다.
allowed-tools:
  [
    'Bash(curl:*)',
    'Bash(python3:*)',
    'Bash(mkdir:*)',
    'Bash(grep:*)',
    'Read',
    'Write',
    'AskUserQuestion',
  ]
---

TraceQL(service.name + span name) 기준으로 특정 날짜·시간 범위를 반복 조회해 `prod_trace/{폴더명}/{날짜}/` 구조로 저장하고 `traces-timeline.csv`를 생성한다.

**환경**: Tempo 주소 → `config/env/.env`의 `MONITORING_SERVER_IP` → `http://{IP}:3200` / 인증: Bearer token (`MONITORING_API_TOKEN`) / 저장: `prod_trace/` (`.gitignore` 등록) / 타임존: KST(UTC+9)

---

### 1단계: MONITORING_SERVER_IP 읽기 및 서비스 목록 조회

`AskUserQuestion` 호출 전에 먼저 IP와 토큰을 읽고 Tempo 서비스 목록을 가져온다.

```bash
MONITORING_IP=$(grep 'MONITORING_SERVER_IP' config/env/.env | cut -d'=' -f2 | tr -d ' ')
MONITORING_API_TOKEN=$(grep 'MONITORING_API_TOKEN' config/env/.env | cut -d'=' -f2 | tr -d ' ')
TEMPO_URL="http://${MONITORING_IP}:3200"
curl -s -H "Authorization: Bearer ${MONITORING_API_TOKEN}" "${TEMPO_URL}/api/search/tag/service.name/values"
```

조회 실패 시 오류 출력 후 중단.

### 2단계: 파라미터 수집

1단계 조회 결과로 `AskUserQuestion`으로 수집:

- **서비스명**: `tagValues` 배열을 옵션으로 제시 (4개 초과 시 상위 3개 + Other)
- **Span Name**: 직접 입력 (예: `POST /dcare-dev/w/v2/pcps`)
- **날짜 목록**: 쉼표 구분 직접 입력 (예: `2026-07-03, 2026-07-06`)
- **시간 범위**: 시작·종료 KST 직접 입력 (예: `08:30 ~ 09:30`)
- **limit**: 기본 1000, 최대 1000

### 3단계: 폴더명 처리

1. `AskUserQuestion`으로 한글/영문 자유 입력 (예: `PCP 등록 API 분석`)
2. Claude가 kebab-case 변환 (예: `pcp-registration-api-analysis`)
3. 변환 결과 확인 → OK면 확정, 수정 시 직접 입력 받아 확정
4. 최종 폴더: `prod_trace/{확정된 폴더명}/`

### 4단계: 날짜별 타임스탬프 계산 및 Tempo 조회

날짜 목록을 순회하며 각 날짜에 대해 실행. **반드시 `curl -G` GET 방식** (POST body 방식은 TraceQL 필터 무시됨).

**타임스탬프 계산** (macOS → python3):

```python
from datetime import datetime, timezone, timedelta
KST = timezone(timedelta(hours=9))
start = datetime(YYYY, M, D, HH, MM, 0, tzinfo=KST)
end   = datetime(YYYY, M, D, HH, MM, 0, tzinfo=KST)
START = int(start.timestamp())
END   = int(end.timestamp())
```

**Tempo 조회**:

```bash
mkdir -p "prod_trace/{폴더명}/{YYYY-MM-DD}"

curl -G \
  --data-urlencode 'q={resource.service.name="{서비스명}" && name="{span name}"}' \
  --data-urlencode "start=${START}" \
  --data-urlencode "end=${END}" \
  --data-urlencode "limit={limit}" \
  -H "Authorization: Bearer ${MONITORING_API_TOKEN}" \
  -s "${TEMPO_URL}/api/search" \
  -o "prod_trace/{폴더명}/{YYYY-MM-DD}/search-result.json"
```

결과 0건이면 오류가 아님을 안내하고 다음 날짜로 진행.

### 5단계: 날짜별 파일 저장

날짜별 폴더에 두 파일 저장:

- **search-result.json**: Tempo 원본 응답 그대로 (4단계에서 이미 저장됨)
- **traces-summary.json**: 핵심 필드 추출 + durationMs 내림차순 + startTimeKST 변환

```python
import json
from datetime import datetime, timezone, timedelta

KST = timezone(timedelta(hours=9))

with open('search-result.json') as f:
    data = json.load(f)

traces = data.get('traces', [])

def to_kst(nano_str):
    ns = int(nano_str)
    sec = ns / 1e9
    dt = datetime.fromtimestamp(sec, tz=KST)
    return dt.strftime('%Y-%m-%d %H:%M:%S KST')

summary = {
    'total': len(traces),
    'query': '{resource.service.name="{서비스명}" && name="{span name}"}',
    'serviceName': '{서비스명}',
    'spanName': '{span name}',
    'timeRange': '{YYYY-MM-DD} {시작}~{종료} KST',
    'exportedAt': '{YYYY-MM-DD}',
    'traces': [
        {
            'traceID': t.get('traceID', ''),
            'rootTraceName': t.get('rootTraceName', ''),
            'rootServiceName': t.get('rootServiceName', ''),
            'durationMs': t.get('durationMs', 0),
            'startTimeUnixNano': t.get('startTimeUnixNano', ''),
            'startTimeKST': to_kst(t.get('startTimeUnixNano', '0'))
        }
        for t in sorted(traces, key=lambda x: x.get('durationMs', 0), reverse=True)
    ]
}

with open('traces-summary.json', 'w') as f:
    json.dump(summary, f, ensure_ascii=False, indent=2)
```

### 6단계: traces-timeline.csv 생성 (전 날짜 합산)

`prod_trace/{폴더명}/traces-timeline.csv`에 모든 날짜 데이터 누적 저장. 컬럼: `date, startTimeKST, durationMs, traceID`. 기존 파일 있으면 누적 추가 후 재정렬.

```python
import json, csv, os
from datetime import datetime, timezone, timedelta

KST = timezone(timedelta(hours=9))
base_folder = 'prod_trace/{폴더명}'
csv_path = f'{base_folder}/traces-timeline.csv'

rows = []
if os.path.exists(csv_path):
    with open(csv_path, newline='') as f:
        reader = csv.DictReader(f)
        rows = list(reader)

for date_dir in sorted(os.listdir(base_folder)):
    summary_path = f'{base_folder}/{date_dir}/traces-summary.json'
    if not os.path.exists(summary_path):
        continue
    with open(summary_path) as f:
        data = json.load(f)
    for t in data.get('traces', []):
        rows.append({
            'date': date_dir,
            'startTimeKST': t.get('startTimeKST', ''),
            'durationMs': t.get('durationMs', 0),
            'traceID': t.get('traceID', ''),
            '_nano': int(t.get('startTimeUnixNano', '0'))
        })

seen = set()
unique_rows = []
for r in sorted(rows, key=lambda x: int(x.get('_nano', 0))):
    if r['traceID'] not in seen:
        seen.add(r['traceID'])
        unique_rows.append(r)

with open(csv_path, 'w', newline='') as f:
    writer = csv.DictWriter(f, fieldnames=['date', 'startTimeKST', 'durationMs', 'traceID'])
    writer.writeheader()
    for r in unique_rows:
        writer.writerow({k: r[k] for k in ['date', 'startTimeKST', 'durationMs', 'traceID']})
```

### 7단계: 결과 요약 출력

```
트레이스 목록 추출 완료
- 서비스명: {서비스명}
- Span Name: {span name}
- 시간 범위: {시작}~{종료} KST
- 저장 경로: prod_trace/{폴더명}/

날짜별 건수:
| 날짜       | 건수    |
|------------|---------|
| 2026-07-03 | 1,000건 |
| 2026-07-06 |   948건 |

traces-timeline.csv: 총 N건 (startTimeKST 오름차순)

파일 구조:
prod_trace/{폴더명}/
├── {YYYY-MM-DD}/
│   ├── search-result.json
│   └── traces-summary.json
└── traces-timeline.csv
```

결과 출력 후 `AskUserQuestion`으로 추가 날짜 조회 여부 확인:
- **있음** → 4~7단계 반복, traces-timeline.csv 누적 갱신
- **없음** → 종료

## 규칙

- `start`/`end`는 **초 단위** Unix 타임스탬프 정수.
- Tempo 조회는 반드시 `curl -G` GET 방식. POST body 방식은 TraceQL 필터 무시됨.
- `prod_trace/` 폴더는 `.gitignore` 등록 — 커밋되지 않음.
- `limit` 기본값 1000. 1000건이 채워지면 실제 더 있을 수 있음을 안내.
- 타임존은 KST(UTC+9) 고정 — `timezone(timedelta(hours=9))`로 처리.
- `prod-trace-export`와의 차이: 이 스킬은 TraceQL + 날짜 범위 반복 조회 + CSV 생성 목적. 서비스 전체 최근 N분 조회는 `/prod-trace-export` 사용.
