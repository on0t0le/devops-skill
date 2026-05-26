---
name: loki
description: >
  Query Grafana Loki log aggregation using curl. Use whenever the user wants to
  search logs in Loki, fetch log streams, explore labels, run LogQL queries, or
  check log volume. Supports multiple named instances with persistent config.
  Works with Loki, Grafana Cloud Logs, and any LogQL-compatible backend.
  Trigger: /loki, "query loki", "search loki logs", "logql", "fetch logs from loki",
  "loki streams", "grafana logs", "show loki labels".
trigger: /loki
---

# Loki via curl

All operations use `curl` against the Loki HTTP API (`/loki/api/v1/`).
Read-only — never use `/loki/api/v1/push`.

## Instance Resolution

Resolve target in this order:

1. **URL in message** — user pastes `http://...` → use directly
2. **Instance name in message** — "prod", "grafana-cloud" → look up in config
3. **Config exists** — read `~/.claude/loki-instances.json`, one instance → use it; multiple → list and ask
4. **Nothing** — ask: "What's the Loki endpoint? (e.g. `http://localhost:3100`)"

Config file (`~/.claude/loki-instances.json`):
```json
{
  "instances": {
    "prod":          { "url": "http://loki-prod:3100", "default": true },
    "grafana-cloud": { "url": "https://logs-prod-xxx.grafana.net", "auth": "user_id:grafana_api_key" },
    "staging":       { "url": "http://loki-staging:3100", "auth_header": "Bearer eyJ..." },
    "multi-tenant":  { "url": "http://loki:3100", "org_id": "tenant-1" }
  }
}
```

`auth` → Basic auth (`-u user:pass`)  
`auth_header` → Raw `Authorization` header  
`org_id` → Adds `X-Scope-OrgID: <value>` header (required for multi-tenant Loki)  
No auth → unauthenticated

## curl Auth Patterns

```bash
# No auth
curl -s "http://HOST:PORT/loki/api/v1/ENDPOINT"

# Basic auth (Grafana Cloud)
curl -s -u "user_id:api_key" "http://HOST:PORT/loki/api/v1/ENDPOINT"

# Bearer token
curl -s -H "Authorization: Bearer TOKEN" "http://HOST:PORT/loki/api/v1/ENDPOINT"

# Multi-tenant
curl -s -H "X-Scope-OrgID: tenant-1" "http://HOST:PORT/loki/api/v1/ENDPOINT"

# Multi-tenant + auth
curl -s -u "user:pass" -H "X-Scope-OrgID: tenant-1" "http://HOST:PORT/loki/api/v1/ENDPOINT"
```

Always use `--data-urlencode` for LogQL expressions to handle `{`, `}`, `|` safely.  
Pipe through `| jq .` for readability.

## Operations

### Health Check

```bash
curl -s "http://HOST:PORT/ready"
curl -s "http://HOST:PORT/loki/api/v1/status/buildinfo" | jq .
```

### List Labels

```bash
curl -s [AUTH] "http://HOST:PORT/loki/api/v1/labels" | jq '.data | sort | .[]'

# With time range (labels seen in last 1h)
curl -s [AUTH] "http://HOST:PORT/loki/api/v1/labels?start=$(( $(date +%s) - 3600 ))000000000&end=$(date +%s)000000000" \
  | jq '.data | sort | .[]'
```

### Label Values

```bash
curl -s [AUTH] "http://HOST:PORT/loki/api/v1/label/LABEL_NAME/values" | jq '.data[]'
```

Useful labels: `app`, `namespace`, `pod`, `container`, `job`, `env`, `level`, `host`.

### List Streams (series)

```bash
curl -s [AUTH] -G "http://HOST:PORT/loki/api/v1/series" \
  --data-urlencode 'match[]={app="myapp"}' \
  --data-urlencode "start=$(( $(date +%s) - 3600 ))000000000" \
  --data-urlencode "end=$(date +%s)000000000" \
  | jq '.data[]'
```

### Instant Query (LogQL)

```bash
curl -s [AUTH] -G "http://HOST:PORT/loki/api/v1/query" \
  --data-urlencode 'query={app="myapp"} |= "error"' \
  --data-urlencode "limit=50" \
  | jq '.data.result[].values[] | .[1]'  # print log lines only
```

### Range Query (most common — fetch logs over time)

```bash
curl -s [AUTH] -G "http://HOST:PORT/loki/api/v1/query_range" \
  --data-urlencode 'query={app="myapp"}' \
  --data-urlencode "start=$(( $(date +%s) - 3600 ))000000000" \
  --data-urlencode "end=$(date +%s)000000000" \
  --data-urlencode "limit=100" \
  --data-urlencode "direction=backward" \
  | jq -r '.data.result[].values[] | .[0] + " " + .[1]'
```

`direction=backward` → newest first (default for log tailing).  
`direction=forward` → oldest first (good for incident replay).

**Time shortcuts** (nanosecond timestamps):
- Last 15m: `start=$(( $(date +%s) - 900 ))000000000`
- Last 1h:  `start=$(( $(date +%s) - 3600 ))000000000`
- Last 24h: `start=$(( $(date +%s) - 86400 ))000000000`

### Metric Query (log rate, count over time)

```bash
# Log rate per second (aggregation query)
curl -s [AUTH] -G "http://HOST:PORT/loki/api/v1/query_range" \
  --data-urlencode 'query=rate({app="myapp"}[5m])' \
  --data-urlencode "start=$(( $(date +%s) - 3600 ))000000000" \
  --data-urlencode "end=$(date +%s)000000000" \
  --data-urlencode "step=60" \
  | jq '.data.result[] | {stream: .stream, values: (.values | length)}'

# Count errors in last hour
curl -s [AUTH] -G "http://HOST:PORT/loki/api/v1/query" \
  --data-urlencode 'query=count_over_time({app="myapp"} |= "error" [1h])' \
  | jq '.data.result[].value[1]'
```

## Common LogQL Patterns

Translate user requests into LogQL:

| User says | LogQL |
|---|---|
| "logs from app X" | `{app="X"}` |
| "logs from namespace Y" | `{namespace="Y"}` |
| "logs containing 'error'" | `{app="X"} \|= "error"` |
| "logs NOT containing 'healthcheck'" | `{app="X"} != "healthcheck"` |
| "error logs (level field)" | `{app="X"} \| json \| level="error"` |
| "logs matching regex" | `{app="X"} \|~ "timeout.*5[0-9]{2}"` |
| "logs with status 500" | `{app="X"} \| json \| status=500` |
| "log rate" | `rate({app="X"}[5m])` |
| "error count last hour" | `count_over_time({app="X"} \|= "error" [1h])` |
| "top 5 error messages" | `topk(5, count by (msg) (rate({app="X"} \| json \| level="error" [5m])))` |
| "logs from pod P" | `{pod="P"}` or `{pod=~"deployment-.*"}` |

**Pipeline operators** (chain with `|`):
- `|= "text"` — must contain
- `!= "text"` — must not contain
- `|~ "regex"` — matches regex
- `!~ "regex"` — not matches regex
- `| json` — parse JSON log line into fields
- `| logfmt` — parse logfmt (`key=value`) into fields
- `| pattern "<ip> - <user> <_> <method> <path>"` — extract by pattern
- `| line_format "{{.level}} {{.msg}}"` — reformat output

## Output Formatting

- Raw log query → print timestamp + log line, newest first
- Large results (>50 lines) → show first 20, report total count
- Metric query → summarize: metric, time range, peak/avg value
- Empty result → say so, suggest: check label names with label list, widen time range
- Parse errors in LogQL → explain syntax issue, show corrected query

## Config Management (`/loki config`)

Config file: `~/.claude/loki-instances.json`  
Use `Read` + `Write` tools only — no shell commands.

### `/loki config add`

Ask for (only what's missing):
- **name** — alias: `prod`, `grafana-cloud`
- **url** — full base URL: `http://loki:3100`
- **auth** (optional) — `user:pass` → `"auth"`, Bearer token → `"auth_header": "Bearer TOKEN"`
- **org_id** (optional) — multi-tenant org ID → `"org_id": "tenant"`
- **default** (optional) → `"default": true`, unset on others

### `/loki config list`

Read config, display table:

| Name | URL | Auth | Org ID | Default |
|------|-----|------|--------|---------|
| prod | http://... | — | — | ✓ |

### `/loki config remove <name>`

Remove entry, write back, confirm.

### `/loki config test [name]`

```bash
curl -s [AUTH] "http://HOST:PORT/ready"
```

`ready` → connected. Otherwise diagnose connection error.

### Default Instance

One instance or one marked `"default": true` → use without asking.  
Multiple, none default → list and ask.

## Error Reference

| Error | Cause | Fix |
|---|---|---|
| `connection refused` | Wrong host/port or Loki down | Check URL |
| `parse error` in response | Invalid LogQL syntax | Check `{` `}` matching, operator spelling |
| `entry out of order` | Pushing out-of-order (not relevant for reads) | — |
| `context deadline exceeded` | Query too broad, too much data | Add label filters, reduce time range |
| `maximum of series` exceeded | Too many streams matched | Add more label filters |
| `401 Unauthorized` | Missing/wrong auth | Check credentials in config |
| `no org id` | Multi-tenant Loki needs `X-Scope-OrgID` | Add `org_id` to instance config |
| Empty `data.result` | No logs matched | Check label values, widen time range |
