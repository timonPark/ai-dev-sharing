---
name: prod-trace
description: Query the trace of a specific API request from the production monitoring server (Tempo). Analyze per-span durations to identify bottlenecks and save results to prod_trace/{folder-name}/. Use during performance analysis.
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

Query the trace of a specific API request from the production monitoring server (Tempo).
Analyze per-span durations to identify which segment (DB, Redis, external API, etc.) is the bottleneck.
Used in Step 4 of the performance analysis chain (`03-performance-analyst`).

## Usage

```
/prod-trace
```

## Environment

- **Tempo address**: use `MONITORING_SERVER_IP` from `config/env/.env` → `http://{IP}:3200`
- **Auth**: Bearer token (`MONITORING_API_TOKEN`, read from `config/env/.env`)
- **Save path**: `prod_trace/` folder at the project root

## Process

### Step 1: Collect required info

First read the IP and token.

```bash
MONITORING_IP=$(grep 'MONITORING_SERVER_IP' config/env/.env | cut -d'=' -f2 | tr -d ' ')
MONITORING_API_TOKEN=$(grep 'MONITORING_API_TOKEN' config/env/.env | cut -d'=' -f2 | tr -d ' ')
TEMPO_URL="http://${MONITORING_IP}:3200"
```

Confirm the two items below simultaneously via `AskUserQuestion`.

**Query mode** options:
- Direct lookup by specific traceId (when the traceId is already known)
- Search by service name + operation name
- Search by HTTP path (e.g., /m/medication/list)
- Search slow traces (above a given response time)

**Time range** options:
- Last 30 minutes
- Last 1 hour
- Last 3 hours
- Custom (in minutes)

### Step 2: Create save folder

```bash
DATETIME=$(date '+%Y-%m-%d-%H%M%S')
mkdir -p prod_trace/${DATETIME}-trace
```

### Step 3: Search / fetch traces

#### Mode A: Direct lookup by traceId

```bash
curl -s \
  -H "Authorization: Bearer ${MONITORING_API_TOKEN}" \
  "http://{MONITORING_SERVER_IP}:3200/api/traces/{traceId}" | \
  jq '.' > "prod_trace/{folder-name}/trace-{traceId}.json"
```

#### Mode B: Tag-based search (HTTP path, service name, etc.)

Use the Tempo Search API. Must use `curl -G` GET method (filters are ignored with POST).

```bash
curl -G -s \
  -H "Authorization: Bearer ${MONITORING_API_TOKEN}" \
  "http://{MONITORING_SERVER_IP}:3200/api/search" \
  --data-urlencode "tags=http.url={path}" \
  --data-urlencode "minDuration=1000ms" \
  --data-urlencode "start=$START" \
  --data-urlencode "end=$END" \
  --data-urlencode "limit=20" | \
  jq '.' > "prod_trace/{folder-name}/search-result.json"
```

Extract the traceId list from the search result and look up the top 5 individually.

```bash
# Extract traceIds from search-result.json and look up each one
jq -r '.traces[].traceID' prod_trace/{folder-name}/search-result.json | head -5 | while read traceId; do
  curl -s \
    -H "Authorization: Bearer ${MONITORING_API_TOKEN}" \
    "http://{MONITORING_SERVER_IP}:3200/api/traces/${traceId}" | \
    jq '.' > "prod_trace/{folder-name}/trace-${traceId}.json"
done
```

### Step 4: Span analysis summary

Read the saved trace files and analyze per-span durations.

Analysis items:
- Total request duration
- Per-span duration (DB query, Redis, external API, internal functions, etc.)
- Top 5 slowest spans
- Parallel vs serial segments

### Step 5: Print result summary

```
✅ Trace query complete
- Number of traceIds queried: N
- Save path: prod_trace/{folder-name}/

Bottleneck summary:
- Total response time: N ms
- DB query segment: N ms (N% of total)
- Redis segment: N ms (N% of total)
- Slowest span: {span name} N ms
```

## Rules

- `start` / `end` must be **integer Unix timestamps in seconds**.
- The Tempo Search API (Mode B) must use `curl -G` GET method. Filters are ignored with POST.
- Trace files can be large, so only look up the top 5 individually.
- Register the `prod_trace/` folder in `.gitignore`.
- Use Mode A if the traceId is known directly, otherwise use Mode B.
