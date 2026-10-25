---
name: prod-log
description: Batch query production-1~4 logs from the production monitoring server (Loki). Confirms time range and search keyword first, then queries; results are saved to prod_log/{date-keyword}/ as per-server JSON files. Use when the user requests "prod logs", "production logs", "log query", "log search", etc.
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

Batch query production-1 ~ production-4 logs from the production monitoring server (Loki) and save results as JSON files.

## Usage

```
/prod-log
```

## Environment

- **Loki address**: use `MONITORING_SERVER_IP` from `config/env/.env` → `http://3.39.48.65:3100`
- **Target servers**: `production-1`, `production-2`, `production-3`, `production-4`
- **Auth**: Bearer token (`MONITORING_API_TOKEN` in `config/env/.env`)
- **Save path**: `prod_log/` folder at the project root

## Process

### Step 1: Collect required info (must confirm before querying)

**1-1. Ask for time range + search type simultaneously via `AskUserQuestion`.**

**Time range** options:
- Last 5 minutes
- Last 30 minutes
- Last 1 hour
- Last 3 hours
- Last 6 hours
- Custom (in minutes)

**Search type** options:
- userIdx
- URI path
- Error message
- Custom

**1-2. Ask for the search value via a separate `AskUserQuestion`.**

Depending on the search type chosen in 1-1, ask for the actual value to search for.
- userIdx selected → "Please enter the userIdx value to search for."
- URI path selected → "Please enter the URI path to search for. (e.g., /m/message/list)"
- Error message selected → "Please enter the error-message keyword to search for."
- Custom selected → "Please enter the keyword to search for."

→ Do not proceed with the query unless **both** time range and search value are confirmed.

### Step 2: Create save folder

Folder naming rules:
- Format: `{YYYY-MM-DD-HHmmss}-{keyword-slug}`
- Date/time: year-month-day-hour-min-sec at query time (`2026-07-02-114512` style)
- Keyword slug: keep only alphanumerics, hyphens, and underscores from the keyword; strip all other special characters (`:`, `"`, `/`, `{`, `}`, spaces, etc.).
- Example: keyword `6a1555c237760446d7546fec` → `prod_log/2026-07-02-114512-6a1555c237760446d7546fec/`
- The search time range is not part of the folder name.

```bash
DATETIME=$(date '+%Y-%m-%d-%H%M%S')
mkdir -p prod_log/${DATETIME}-{keyword-slug}
```

### Step 3: Compute timestamps

On macOS, use `python3` to generate nanosecond timestamps.

```bash
END=$(python3 -c "import time; print(int(time.time() * 1e9))")
START=$(python3 -c "import time; print(int((time.time() - {seconds}) * 1e9))")
```

Time range → seconds mapping:
- 5 min → 300
- 30 min → 1800
- 1 hour → 3600
- 3 hours → 10800
- 6 hours → 21600

### Step 4: Batch query 4 servers and save

For each server, call the Loki API and save the result as a JSON array.

```bash
TOKEN=$(grep MONITORING_API_TOKEN config/env/.env | cut -d= -f2)
for env in production-1 production-2 production-3 production-4; do
  curl -s "http://3.39.48.65:3100/loki/api/v1/query_range" \
    -H "Authorization: Bearer $TOKEN" \
    --data-urlencode "query={env=\"$env\"} |= \`{keyword}\`" \
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
" > "prod_log/{folder-name}/${env}.json"
done
```

### Step 5: Print result summary

Print per-server saved counts.

```
production-1: N entries
production-2: N entries
production-3: N entries
production-4: N entries
Save path: prod_log/{folder-name}/
```

Even servers with 0 results produce a file with an empty array `[]`.

## Rules

- `start` / `end` parameters must be **nanosecond Unix timestamp integers**. The string `"now"` is not supported.
- JSON files contain **only the raw log array**. Do not include metadata or markdown.
- Save file name: `{env}.json` (e.g., `production-1.json`)
- The `prod_log/` folder is registered in `.gitignore` and is not committed.
