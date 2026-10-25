# CLAUDE.md

This project is a Claude Code skill repository for production server monitoring.
It integrates with Grafana (Loki · Tempo · Prometheus) to query logs, traces, and metrics.

## Environment

- **Monitoring server IP**: `MONITORING_SERVER_IP` in `config/env/.env`
- **Auth token**: `MONITORING_API_TOKEN` in `config/env/.env`
- **Loki**: `http://{IP}:3100` — nanosecond timestamps, Bearer token auth
- **Tempo**: `http://{IP}:3200` — second-precision timestamps, Bearer token auth
- **Prometheus**: `http://{IP}:9090` — second-precision timestamps, Bearer token auth
- **OS**: macOS (use `python3` for timestamp calculation; `date -d` is not available)

## Skill list

| Skill | Backend | Purpose |
|---|---|---|
| `/prod-log` | Loki | Batch query logs across production-1~4 |
| `/prod-metrics` | Prometheus | Query metrics via PromQL |
| `/prod-trace` | Tempo | Drill down into spans of a specific traceId |
| `/prod-trace-export` | Tempo | Extract list of traces by service name · minDuration |
| `/tempo-search` | Tempo | TraceQL + per-date iterative query + CSV generation |
| `/dcare-user-detail` | D.CARE Admin API | Look up member details by userIdx |
| `/extract-access-token` | D.CARE Admin API + Loki | Extract member accessToken by userIdx |

## Timestamp rules

Each backend uses a different unit, so always distinguish between them.

| Backend | Unit | How to generate |
|---|---|---|
| Loki | nanoseconds | `int(time.time() * 1e9)` |
| Tempo | seconds | `int(time.time())` |
| Prometheus | seconds | `int(time.time())` |

## Output save paths

All query results are saved to the paths below and are not committed.

```
prod_log/       ← Loki log query results
prod_trace/     ← Tempo trace query results
prod_metrics/   ← Prometheus metric query results
config/env/     ← Environment variables (IP, token)
```

## Notes

- The `tempo-search` skill must use `curl -G` GET method. The POST method causes TraceQL filters to be ignored.
- Zero results do not indicate an error. Guide the user to adjust the time range or search conditions and retry.
- `config/env/.env` is registered in `.gitignore` and is not committed.
