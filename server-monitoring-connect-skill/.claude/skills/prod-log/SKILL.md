---
name: prod-log
description: 운영 모니터링 서버(Loki)에서 production-1~4 로그를 일괄 조회한다. 시간 범위와 검색 키워드를 먼저 확인한 후 조회하며, 결과는 prod_log/{날짜-키워드}/ 폴더에 서버별 JSON 파일로 저장한다. 사용자가 "운영 로그", "prod 로그", "로그 조회", "로그 검색" 등을 요청하면 사용한다.
allowed-tools:
  [
    'Bash(curl:*)',
    'Bash(python3:*)',
    'Bash(mkdir:*)',
    'Bash(jq:*)',
    'Bash(date:*)',
    'Bash(wc:*)',
    'AskUserQuestion',
  ]
---

# prod-log

운영 모니터링 서버(Loki)에서 production-1 ~ production-4 로그를 일괄 조회하고 결과를 JSON 파일로 저장한다.

## 사용법

```
/prod-log
```

## 환경 정보

- **Loki 주소**: `config/env/.env`의 `MONITORING_SERVER_IP` 값 사용 → `http://3.39.48.65:3100`
- **조회 대상 서버**: `production-1`, `production-2`, `production-3`, `production-4`
- **인증**: Bearer token (`config/env/.env`의 `MONITORING_API_TOKEN`)
- **저장 경로**: 프로젝트 루트 `prod_log/` 폴더

## 프로세스

### 1단계: 필수 정보 수집 (조회 전 반드시 확인)

**1-1. 시간 범위 + 검색 유형을 `AskUserQuestion`으로 동시에 질문한다.**

**시간 범위** 옵션:
- 최근 5분
- 최근 30분
- 최근 1시간
- 최근 3시간
- 최근 6시간
- 직접 입력 (분 단위)

**검색 유형** 옵션:
- userIdx
- URI 경로
- 에러 메시지
- 직접 입력

**1-2. 검색 값을 `AskUserQuestion`으로 추가 질문한다.**

1-1에서 선택한 검색 유형에 따라 실제 검색할 값을 별도로 질문한다.
- userIdx 선택 시 → "검색할 userIdx 값을 입력해주세요."
- URI 경로 선택 시 → "검색할 URI 경로를 입력해주세요. (예: /m/message/list)"
- 에러 메시지 선택 시 → "검색할 에러 메시지 키워드를 입력해주세요."
- 직접 입력 선택 시 → "검색할 키워드를 입력해주세요."

→ 시간 범위와 검색 값 **둘 다** 확인되지 않으면 조회를 진행하지 않는다.

### 2단계: 저장 폴더 생성

폴더명 규칙:
- 형식: `{YYYY-MM-DD-HHmmss}-{키워드슬러그}`
- 날짜·시간: 조회 시점의 년월일시분초 (`2026-07-02-114512` 형태)
- 키워드슬러그: 검색 키워드에서 영문·숫자·하이픈·언더스코어만 남기고 나머지 특수문자(`:`, `"`, `/`, `{`, `}`, 공백 등)는 모두 제거한다.
- 예) 키워드 `6a1555c237760446d7546fec` → `prod_log/2026-07-02-114512-6a1555c237760446d7546fec/`
- 검색 시간 범위는 폴더명에 포함하지 않는다.

```bash
DATETIME=$(date '+%Y-%m-%d-%H%M%S')
mkdir -p prod_log/${DATETIME}-{키워드슬러그}
```

### 3단계: 타임스탬프 계산

macOS 환경이므로 `python3`으로 나노초 타임스탬프를 생성한다.

```bash
END=$(python3 -c "import time; print(int(time.time() * 1e9))")
START=$(python3 -c "import time; print(int((time.time() - {초}) * 1e9))")
```

시간 범위 → 초 변환:
- 5분 → 300
- 30분 → 1800
- 1시간 → 3600
- 3시간 → 10800
- 6시간 → 21600

### 4단계: 4개 서버 일괄 조회 및 저장

각 서버별로 Loki API를 호출하고 결과를 JSON 배열로 저장한다.

```bash
TOKEN=$(grep MONITORING_API_TOKEN config/env/.env | cut -d= -f2)
for env in production-1 production-2 production-3 production-4; do
  curl -s "http://3.39.48.65:3100/loki/api/v1/query_range" \
    -H "Authorization: Bearer $TOKEN" \
    --data-urlencode "query={env=\"$env\"} |= \`{키워드}\`" \
    --data-urlencode "start=$START" \
    --data-urlencode "end=$END" \
    --data-urlencode 'limit=500' \
    --data-urlencode 'direction=backward' | \
    jq -r '.data.result[] | .values[] | .[1]' 2>/dev/null | \
    python3 -c "
import sys, json
logs = []
for line in sys.stdin:
    line = line.strip()
    try:
        logs.append(json.loads(line))
    except:
        pass
print(json.dumps(logs, ensure_ascii=False, indent=2))
" > "prod_log/{폴더명}/${env}.json"
done
```

### 5단계: 결과 요약 출력

각 서버별 저장 건수를 출력한다.

```
production-1: N건
production-2: N건
production-3: N건
production-4: N건
저장 경로: prod_log/{폴더명}/
```

건수가 0인 서버도 빈 배열 `[]`로 파일을 생성한다.

## 규칙

- `start` / `end` 파라미터는 반드시 **나노초 Unix 타임스탬프** 정수를 사용한다. `"now"` 문자열은 지원되지 않는다.
- JSON 파일은 **순수 로그 배열**만 담는다. 메타 정보나 마크다운은 포함하지 않는다.
- 저장 파일명: `{env}.json` (예: `production-1.json`)
- `prod_log/` 폴더는 `.gitignore`에 등록되어 있으므로 커밋되지 않는다.
