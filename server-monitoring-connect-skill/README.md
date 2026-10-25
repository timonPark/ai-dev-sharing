# server-monitoring-connect-skill

A collection of Claude Code skills integrated with Grafana (Loki · Tempo · Prometheus).
Query production server logs, traces, and metrics from the CLI and save them to files.

## Environment setup

Enter monitoring server details in `config/env/.env`.

```
MONITORING_SERVER_IP=
MONITORING_API_TOKEN=
DCARE_API_BASE_URL=
DCARE_ADMIN_ACCESS_TOKEN=
DCARE_CLIENT_KEY=
DCARE_MASTER_KEY=
DCARE_REST_API_KEY=
```

## Skill list

### `/prod-log`
Batch query production-1~4 server logs from Loki.
- Search by userIdx / URI path / error message / keyword
- Output: `prod_log/{date-keyword}/production-{1~4}.json`

### `/prod-metrics`
Query PromQL metrics from Prometheus.
- Application (HTTP P95 response time, request count, error rate)
- MongoDB (query latency, connection count, document operations)
- Redis (cache hit rate, memory, slow commands, eviction)
- Output: `prod_metrics/{date}-metrics/`

### `/prod-trace`
Query the trace of a specific API request from Tempo and analyze per-span bottlenecks.
- Direct lookup by traceId, or search by HTTP path / service name
- Output: `prod_trace/{date}-trace/`

### `/prod-trace-export`
Batch-extract a list of traces from Tempo by service name and minimum duration.
- Output: `prod_trace/{date}-export/search-result.json`, `traces-summary.json`

### `/tempo-search`
Iteratively query a date range via TraceQL (service.name + span name) and analyze performance distribution.
- Date list is comma-separated input; KST time range specified
- Output: `prod_trace/{folder-name}/{date}/` + `traces-timeline.csv`

## Recommended usage order

```
1. /prod-metrics      → Identify anomalous time windows
2. /prod-log          → Check error/slow logs in that window
3. /prod-trace-export → Extract list of slow traces
4. /prod-trace        → Drill down into spans of a specific traceId
```

## Save paths

Query results are saved to the paths below and excluded from commits via `.gitignore`.

```
prod_log/
prod_trace/
prod_metrics/
config/env/
```
