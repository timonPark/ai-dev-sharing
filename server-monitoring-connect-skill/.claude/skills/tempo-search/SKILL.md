---
name: tempo-search
description: Query Grafana Tempo iteratively over date/time ranges with a TraceQL condition (service.name + span name), batch-extracting per-date trace lists and generating traces-timeline.csv. Use when analyzing API performance distribution over a specific date range.
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

Iteratively query specific date/time ranges based on TraceQL (service.name + span name), save results to the structure `prod_trace/{folder-name}/{date}/`, and generate `traces-timeline.csv`.

**Environment**: Tempo address → `MONITORING_SERVER_IP` in `config/env/.env` → `http://{IP}:3200` / Auth: Bearer token (`MONITORING_API_TOKEN`) / Save: `prod_trace/` (registered in `.gitignore`) / Time zone: KST (UTC+9)

---

### Step 1: Read MONITORING_SERVER_IP and fetch service list

Before calling `AskUserQuestion`, read the IP and token and fetch the Tempo service list.

```bash
MONITORING_IP=$(grep 'MONITORING_SERVER_IP' config/env/.env | cut -d'=' -f2 | tr -d ' ')
MONITORING_API_TOKEN=$(grep 'MONITORING_API_TOKEN' config/env/.env | cut -d'=' -f2 | tr -d ' ')
TEMPO_URL="http://${MONITORING_IP}:3200"
curl -s -H "Authorization: Bearer ${MONITORING_API_TOKEN}" "${TEMPO_URL}/api/search/tag/service.name/values"
```

On failure, print the error and abort.

### Step 2: Collect parameters

Use the Step 1 result with `AskUserQuestion` to collect:

- **Service name**: present the `tagValues` array as options (if more than 4, use top 3 + Other)
- **Span Name**: custom input (e.g., `POST /dcare-dev/w/v2/pcps`)
- **Date list**: comma-separated custom input (e.g., `2026-07-03, 2026-07-06`)
- **Time range**: KST start/end as custom input (e.g., `08:30 ~ 09:30`)
- **limit**: default 1000, max 1000

### Step 3: Folder name handling

1. Ask freely in Korean/English via `AskUserQuestion` (e.g., `PCP registration API analysis`)
2. Claude converts to kebab-case (e.g., `pcp-registration-api-analysis`)
3. Confirm the conversion → OK: finalize; otherwise request a custom input to finalize
4. Final folder: `prod_trace/{finalized-folder-name}/`

### Step 4: Compute per-date timestamps and query Tempo

Iterate over the date list, running once per date. **Must use `curl -G` GET** (POST body ignores TraceQL filters).

**Timestamp computation** (macOS → python3):

```python
from datetime import datetime, timezone, timedelta
KST = timezone(timedelta(hours=9))
start = datetime(YYYY, M, D, HH, MM, 0, tzinfo=KST)
end   = datetime(YYYY, M, D, HH, MM, 0, tzinfo=KST)
START = int(start.timestamp())
END   = int(end.timestamp())
```

**Tempo query**:

```bash
mkdir -p "prod_trace/{folder-name}/{YYYY-MM-DD}"

curl -G \
  --data-urlencode 'q={resource.service.name="{service-name}" && name="{span name}"}' \
  --data-urlencode "start=${START}" \
  --data-urlencode "end=${END}" \
  --data-urlencode "limit={limit}" \
  -H "Authorization: Bearer ${MONITORING_API_TOKEN}" \
  -s "${TEMPO_URL}/api/search" \
  -o "prod_trace/{folder-name}/{YYYY-MM-DD}/search-result.json"
```

If 0 results, announce it is not an error and move to the next date.

### Step 5: Per-date file saving

Save two files per date folder:

- **search-result.json**: raw Tempo response as-is (already saved in Step 4)
- **traces-summary.json**: key fields extracted + sorted by durationMs descending + startTimeKST converted

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
    'query': '{resource.service.name="{service-name}" && name="{span name}"}',
    'serviceName': '{service-name}',
    'spanName': '{span name}',
    'timeRange': '{YYYY-MM-DD} {start}~{end} KST',
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

### Step 6: Generate traces-timeline.csv (all dates combined)

Accumulate data for all dates in `prod_trace/{folder-name}/traces-timeline.csv`. Columns: `date, startTimeKST, durationMs, traceID`. If the file already exists, append and re-sort.

```python
import json, csv, os
from datetime import datetime, timezone, timedelta

KST = timezone(timedelta(hours=9))
base_folder = 'prod_trace/{folder-name}'
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

### Step 7: Print result summary

```
Trace list extraction complete
- Service name: {service-name}
- Span Name: {span name}
- Time range: {start}~{end} KST
- Save path: prod_trace/{folder-name}/

Entries per date:
| Date       | Count   |
|------------|---------|
| 2026-07-03 | 1,000   |
| 2026-07-06 |   948   |

traces-timeline.csv: total N entries (ascending by startTimeKST)

File structure:
prod_trace/{folder-name}/
├── {YYYY-MM-DD}/
│   ├── search-result.json
│   └── traces-summary.json
└── traces-timeline.csv
```

After printing the result, use `AskUserQuestion` to check whether to query additional dates:
- **Yes** → repeat Steps 4–7, update traces-timeline.csv incrementally
- **No** → end

## Rules

- `start` / `end` are **integer Unix timestamps in seconds**.
- Tempo queries must use `curl -G` GET method. POST body form causes TraceQL filters to be ignored.
- The `prod_trace/` folder is registered in `.gitignore` — not committed.
- `limit` defaults to 1000. Warn when it is reached, since there may actually be more.
- Time zone is fixed to KST (UTC+9) — handle with `timezone(timedelta(hours=9))`.
- Difference from `prod-trace-export`: this skill targets TraceQL + date-range iterative queries + CSV generation. For "last N minutes across a service", use `/prod-trace-export`.
