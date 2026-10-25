---
name: prod-metrics
description: Query metrics for a specific time window from the production monitoring server (Prometheus). Runs PromQL queries for application, MongoDB, and Redis metrics and saves results to prod_metrics/{folder-name}/. Use during performance analysis.
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

Query metrics for a specific time window from the production monitoring server (Prometheus) via PromQL.
Used in Step 3 of the performance analysis chain (`03-performance-analyst`).

## Usage

```
/prod-metrics
```

## Environment

- **Prometheus address**: use `MONITORING_SERVER_IP` from `config/env/.env` → `http://{IP}:9090`
- **Auth**: Bearer token (`MONITORING_API_TOKEN` in `config/env/.env`)
- **Save path**: `prod_metrics/` folder at the project root

## Process

### Step 1: Collect required info

Confirm the two items below simultaneously via `AskUserQuestion`.

**Time range** options:
- Last 30 minutes
- Last 1 hour
- Last 3 hours
- Last 6 hours
- Custom (in minutes)

**Analysis target** options:
- Application metrics (HTTP response time, request count, error rate)
- MongoDB metrics (query latency, connection count, operation count)
- Redis metrics (cache hit rate, memory, slow commands)
- All (all three above)

### Step 2: Create save folder

```bash
DATETIME=$(date '+%Y-%m-%d-%H%M%S')
mkdir -p prod_metrics/${DATETIME}-metrics
```

### Step 3: Compute timestamps

```bash
END=$(python3 -c "import time; print(int(time.time()))")
START=$(python3 -c "import time; print(int(time.time() - {seconds}))")
STEP="60"  # 1-minute interval
```

### Step 4: Run PromQL queries

Use the Prometheus `query_range` API. Read the token first and include an Authorization header on every curl.

```bash
TOKEN=$(grep MONITORING_API_TOKEN config/env/.env | cut -d= -f2)

curl -s "http://{MONITORING_SERVER_IP}:9090/api/v1/query_range" \
  -H "Authorization: Bearer $TOKEN" \
  --data-urlencode "query={PromQL}" \
  --data-urlencode "start=$START" \
  --data-urlencode "end=$END" \
  --data-urlencode "step=$STEP" | jq '.' > "prod_metrics/{folder-name}/{filename}.json"
```

#### Application metric queries

| File name | PromQL | Description |
|---|---|---|
| `http_duration_p95.json` | `histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))` | P95 response time |
| `http_request_rate.json` | `rate(http_requests_total[5m])` | Requests per minute |
| `http_error_rate.json` | `rate(http_requests_total{status=~"5.."}[5m])` | 5xx error rate |

#### MongoDB metric queries

| File name | PromQL | Description |
|---|---|---|
| `mongo_op_latency.json` | `rate(mongodb_op_latencies_latency_total[5m]) / rate(mongodb_op_latencies_ops_total[5m])` | Average latency per operation |
| `mongo_connections.json` | `mongodb_connections{state="current"}` | Current connection count |
| `mongo_document_ops.json` | `rate(mongodb_metrics_document_total[5m])` | Document operations per second |

#### Redis metric queries (available after redis_exporter is integrated)

| File name | PromQL | Description |
|---|---|---|
| `redis_hit_rate.json` | `rate(redis_keyspace_hits_total[5m]) / (rate(redis_keyspace_hits_total[5m]) + rate(redis_keyspace_misses_total[5m]))` | Cache hit rate |
| `redis_memory.json` | `redis_memory_used_bytes / redis_memory_max_bytes` | Memory usage ratio |
| `redis_slowlog.json` | `redis_slowlog_length` | Slow command count |
| `redis_evictions.json` | `rate(redis_evicted_keys_total[5m])` | Evictions per second |

### Step 5: Print result summary

```
✅ Metric query complete
- Time range: {start} ~ {end}
- Save path: prod_metrics/{folder-name}/
- Collected items: {list}

Key numbers:
- HTTP P95 response time: N ms
- MongoDB average latency: N ms
- Redis cache hit rate: N%  (shown after redis_exporter integration)
```

## Rules

- `start` / `end` must be **integer Unix timestamps in seconds**. (Prometheus uses seconds.)
- Redis metrics are only queryable after `redis_exporter` is integrated. Skip those queries before integration.
- Register the `prod_metrics/` folder in `.gitignore`.
