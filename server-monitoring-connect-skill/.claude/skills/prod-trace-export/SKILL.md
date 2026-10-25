---
name: prod-trace-export
description: Search for a trace list in Grafana Tempo by service name and minimum duration, saving the results to files. Use to extract the Grafana Explore UI Tempo Search tab results from the CLI.
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

Extract the trace list that appears when specifying a service name and Duration in Grafana Explore → Tempo → Search tab, and save it to files.

While the `prod-trace` skill targets **span drill-down** (bottleneck analysis) for a single API, this skill targets **batch extraction of a trace list** matching conditions across the entire service.

## Usage

```
/prod-trace-export
```

## Environment

- **Tempo address**: use `MONITORING_SERVER_IP` from `config/env/.env` → `http://{IP}:3200`
- **Auth**: Bearer token (`MONITORING_API_TOKEN`, read from `config/env/.env`)
- **Save path**: `prod_trace/` folder at the project root (registered in `.gitignore`)

## Process

### Step 1: Read MONITORING_SERVER_IP and fetch service list

**Before** calling `AskUserQuestion`, read the IP/token and fetch the actual service list from Tempo.

```bash
MONITORING_IP=$(grep 'MONITORING_SERVER_IP' config/env/.env | cut -d'=' -f2 | tr -d ' ')
MONITORING_API_TOKEN=$(grep 'MONITORING_API_TOKEN' config/env/.env | cut -d'=' -f2 | tr -d ' ')
TEMPO_URL="http://${MONITORING_IP}:3200"
```

Fetch the service name list:

```bash
curl -s \
  -H "Authorization: Bearer ${MONITORING_API_TOKEN}" \
  "${TEMPO_URL}/api/search/tag/service.name/values"
# Response example: { "tagValues": ["application-server", "another-service"] }
```

Parse the response to build the service name array. On failure, print an error and abort.

### Step 2: Collect required info

Present the service name list from Step 1 as `AskUserQuestion` options. Confirm service name, minimum duration, and time range simultaneously.

**Service name**: use the `tagValues` array obtained in Step 1 as options directly.
- Example: `application-server`, `another-service`, … (fetched values become choices)
- If the number exceeds the `AskUserQuestion` option limit (4), present the top 3 as options and route the rest via Other (custom input).

**Minimum duration (minDuration)** options:
- 500ms
- 1s
- 2s
- 5s

**Time range** options:
- Last 30 minutes
- Last 1 hour
- Last 3 hours
- Custom (in minutes)

Optional: maximum result count (default 100, max 1000)

### Step 3: Create save folder

```bash
DATETIME=$(date '+%Y-%m-%d-%H%M%S')
FOLDER="prod_trace/${DATETIME}-export"
mkdir -p "$FOLDER"
```

### Step 4: Compute timestamps

On macOS, use `python3` to generate second-precision Unix timestamps.

```bash
END=$(python3 -c "import time; print(int(time.time()))")
START=$(python3 -c "import time; print(int(time.time()) - {seconds})")
```

Time range → seconds:
- 30 min → 1800
- 1 hour → 3600
- 3 hours → 10800

### Step 5: Call Tempo Search API

Use Tempo's `/api/search` endpoint. Must use `curl -G` GET method (filters are ignored with POST).

```bash
curl -G -s \
  -H "Authorization: Bearer ${MONITORING_API_TOKEN}" \
  "${TEMPO_URL}/api/search" \
  --data-urlencode "tags=service.name={service-name}" \
  --data-urlencode "minDuration={minDuration}" \
  --data-urlencode "start=${START}" \
  --data-urlencode "end=${END}" \
  --data-urlencode "limit={limit}" \
  > "${FOLDER}/search-result.json"
```

**Response structure example:**
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

On API failure (HTTP error, empty response, etc.), print an error message and abort.

### Step 6: Generate summary file

Extract key fields from the raw JSON to create `traces-summary.json`.

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
    'serviceName': '{service-name}',
    'timeRange': '{time-range}',
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

print(f'Total {len(traces)} entries')
"
```

### Step 7: Print result summary

```
✅ Trace list extraction complete
- Service name: {service-name}
- Minimum duration: {minDuration}
- Time range: last {N} minutes
- Extracted: {N} / Inspected traces: {N}
- Save path: {FOLDER}/

Top 10 (by duration, descending):
┌─────────────────────────────────────────┬────────────┐
│ rootTraceName                           │ durationMs │
├─────────────────────────────────────────┼────────────┤
│ POST /m/medication/list                 │  4,521 ms  │
│ ...                                     │  ...       │
└─────────────────────────────────────────┴────────────┘

Files:
- search-result.json : full raw Tempo API response
- traces-summary.json: key fields extracted + sorted by duration descending
```

## Rules

- `start` / `end` are **integer Unix timestamps in seconds** (same as `prod-trace`).
- The Tempo Search API must use `curl -G` GET method. Filters are ignored with POST.
- The `prod_trace/` folder is registered in `.gitignore` and is not committed.
- When there are 0 results, announce it is not an error and guide the user to adjust the time range or minDuration and retry.
- `limit` defaults to 100, max 1000. If over 1000, warn that Tempo API limits may truncate the results.
- Difference from `prod-trace`: this skill targets **trace list extraction** and does not perform span drill-down. For span analysis of a specific traceID, use `/prod-trace`.
