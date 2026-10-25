# ai-dev-sharing

A collection of Claude Code skills and AI development workflow specifications.

## Contents

### backend-feature-builder.md

A 9-step agent workflow specification for implementing new APIs, sync logic, aggregations, and integrations.

| Step | Skill | Description |
|---|---|---|
| Step 1 | `issue-kickoff` | Create branch + fetch Jira issue |
| Step 2 | GATE | Verify 4 requirement items |
| Step 3 | `brainstorming` | Design exploration and agreement |
| Step 4 | `writing-plans` | Write per-layer implementation blueprint |
| Step 5 | `tdd-service` | TDD cycle implementation |
| Step 6 | `codex-review` | Cross-AI code review |
| Step 7 | — | Apply review feedback |
| Step 8 | `postman-doc` | Auto-generate API documentation |
| Step 9 | `commit` / `pr` | Commit · push · create PR |

Entry point: `/work-manager {Jira issue URL or issue key}`

---

### server-monitoring-connect-skill/

A collection of Claude Code skills integrated with Grafana (Loki · Tempo · Prometheus).
Query production server logs, traces, and metrics from the CLI.

| Skill | Backend | Purpose |
|---|---|---|
| `/prod-log` | Loki | Batch query logs across production-1~4 |
| `/prod-metrics` | Prometheus | Query metrics via PromQL |
| `/prod-trace` | Tempo | Drill down into spans of a specific traceId |
| `/prod-trace-export` | Tempo | Extract traces by service name · minDuration |
| `/tempo-search` | Tempo | TraceQL + per-date iterative query + CSV generation |

Recommended usage order:
```
1. /prod-metrics      → Identify anomalous time windows
2. /prod-log          → Check error/slow logs in that window
3. /prod-trace-export → Extract list of slow traces
4. /prod-trace        → Drill down into spans of a specific traceId
```

Configuration: enter monitoring server details in `server-monitoring-connect-skill/config/env/.env`
