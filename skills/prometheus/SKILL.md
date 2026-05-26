---
name: prometheus
model: haiku
description: >
  Query Prometheus instances using curl. Use whenever the user wants to query metrics,
  check alerts, inspect targets, explore label values, run PromQL expressions, or
  investigate anything in Prometheus or VictoriaMetrics. Supports multiple named
  instances. Config persisted to ~/.claude/prometheus-instances.json.
  Trigger: /prometheus, "query prometheus", "check metrics", "promql", "show alerts",
  "prometheus targets", "what metrics", "query victoriametrics", "check VM".
trigger: /prometheus
---

# Prometheus via curl

All operations use `curl` against the Prometheus HTTP API. Works with Prometheus,
VictoriaMetrics, Thanos, Cortex, Mimir (all expose compatible `/api/v1/` endpoints).

## Instance Resolution

Resolve target in this order:

1. **URL in message** — user pastes `http://...` → use directly
2. **Instance name in message** — "prod", "staging" → look up in config
3. **Config exists** — read `~/.claude/prometheus-instances.json`, if one instance use it; if multiple, list and ask
4. **Nothing** — ask: "What's the Prometheus endpoint? (e.g. `http://localhost:9090`)"

Config file (`~/.claude/prometheus-instances.json`):
```json
{
  "instances": {
    "prod":    { "url": "http://prometheus-prod:9090", "default": true },
    "staging": { "url": "http://prometheus-staging:9090" },
    "vm":      { "url": "http://victoria:8428", "auth": "user:pass" },
    "thanos":  { "url": "http://thanos-query:9090", "auth_header": "Bearer eyJ..." }
  }
}
```

`auth` → Basic auth (`-u user:pass`)  
`auth_header` → Raw `Authorization` header value  
No auth → unauthenticated

## curl Auth Patterns

```bash
# No auth
curl -s "http://HOST:PORT/api/v1/ENDPOINT"

# Basic auth
curl -s -u "user:pass" "http://HOST:PORT/api/v1/ENDPOINT"

# Bearer / custom header
curl -s -H "Authorization: Bearer TOKEN" "http://HOST:PORT/api/v1/ENDPOINT"
```

Always pipe through `| jq .` or `| python3 -m json.tool` for readability.  
All Prometheus API responses: `{"status":"success","data":{...}}` — check `status` first.

## Operations

### Instant Query (PromQL)

```bash
curl -s [AUTH] \
  "http://HOST:PORT/api/v1/query?query=PROMQL_EXPRESSION&time=$(date +%s)" \
  | jq '.data.result'
```

Use URL encoding for complex expressions — or use `--data-urlencode`:
```bash
curl -s [AUTH] -G "http://HOST:PORT/api/v1/query" \
  --data-urlencode "query=rate(http_requests_total[5m])" \
  | jq '.data.result'
```

### Range Query (time series)

```bash
curl -s [AUTH] -G "http://HOST:PORT/api/v1/query_range" \
  --data-urlencode "query=PROMQL" \
  --data-urlencode "start=$(date -v-1H +%s 2>/dev/null || date -d '1 hour ago' +%s)" \
  --data-urlencode "end=$(date +%s)" \
  --data-urlencode "step=60" \
  | jq '.data.result'
```

Time shorthands for `start`:
- Last 15m: `$(date +%s) - 900`
- Last 1h: `$(date +%s) - 3600`
- Last 24h: `$(date +%s) - 86400`

### List All Metric Names

```bash
curl -s [AUTH] "http://HOST:PORT/api/v1/label/__name__/values" \
  | jq '.data | sort | .[]'
```

Filter by prefix:
```bash
curl -s [AUTH] "http://HOST:PORT/api/v1/label/__name__/values" \
  | jq '[.data[] | select(startswith("http_"))] | sort | .[]'
```

### List Labels

```bash
curl -s [AUTH] "http://HOST:PORT/api/v1/labels" | jq '.data | sort | .[]'
```

### Label Values

```bash
curl -s [AUTH] "http://HOST:PORT/api/v1/label/LABEL_NAME/values" | jq '.data[]'
```

Useful for: `job`, `instance`, `namespace`, `pod`, `service`, `__name__`.

### Series Search (find metrics matching selector)

```bash
curl -s [AUTH] -G "http://HOST:PORT/api/v1/series" \
  --data-urlencode 'match[]=METRIC_NAME{label="value"}' \
  | jq '.data | length'  # count first

curl -s [AUTH] -G "http://HOST:PORT/api/v1/series" \
  --data-urlencode 'match[]=up' \
  | jq '.data[] | {job: .job, instance: .instance}'
```

### Targets (scrape status)

```bash
# All targets
curl -s [AUTH] "http://HOST:PORT/api/v1/targets" \
  | jq '.data.activeTargets[] | {job: .labels.job, instance: .labels.instance, health: .health, lastError: .lastError}'

# Only unhealthy
curl -s [AUTH] "http://HOST:PORT/api/v1/targets" \
  | jq '[.data.activeTargets[] | select(.health != "up")] | .[] | {job: .labels.job, instance: .labels.instance, health: .health, lastError: .lastError}'

# Dropped targets (not being scraped)
curl -s [AUTH] "http://HOST:PORT/api/v1/targets" \
  | jq '.data.droppedTargets | length'
```

### Health Check Pattern

For "are all X up?", "is X down?", or "check if exporters are running" — **use targets API first**, not PromQL `up{}`.

Targets API returns `lastError` and richer metadata in one call. PromQL `up{}` is for trends and alerting rules.

```bash
# Check all targets matching a job name
curl -s [AUTH] "http://HOST:PORT/api/v1/targets" \
  | jq '[.data.activeTargets[] | select(.labels.job | test("JOB_PATTERN"))] | {total: length, down: [.[] | select(.health != "up")] | length, unhealthy: [.[] | select(.health != "up")] | map({instance: .labels.instance, pod: .labels.instance, error: .lastError})}'
```

**Namespace label naming**: Some jobs (e.g. JMX exporters via pod annotations) use `kubernetes_namespace` instead of `namespace`. If `{namespace="..."}` filter returns empty, retry with `kubernetes_namespace`:

```bash
# Try kubernetes_namespace if namespace returns empty
curl -s [AUTH] -G "http://HOST:PORT/api/v1/query" \
  --data-urlencode 'query=up{job="JOB_NAME", kubernetes_namespace="NAMESPACE"}' \
  | jq '.data.result'
```

To discover which label name is used, inspect a known metric:
```bash
curl -s [AUTH] -G "http://HOST:PORT/api/v1/query" \
  --data-urlencode 'query=up{job="JOB_NAME"}' \
  | jq '.data.result[0].metric | keys'
```

### Alerts

```bash
# Firing alerts
curl -s [AUTH] "http://HOST:PORT/api/v1/alerts" \
  | jq '[.data.alerts[] | select(.state == "firing")] | .[] | {alertname: .labels.alertname, severity: .labels.severity, summary: .annotations.summary, since: .activeAt}'

# All alerts with state
curl -s [AUTH] "http://HOST:PORT/api/v1/alerts" \
  | jq '.data.alerts[] | {name: .labels.alertname, state: .state, severity: .labels.severity}'
```

### Rules (recording + alerting)

```bash
# Alerting rules only
curl -s [AUTH] "http://HOST:PORT/api/v1/rules" \
  | jq '[.data.groups[].rules[] | select(.type == "alerting")] | .[] | {name: .name, state: .state, health: .health}'

# Recording rules
curl -s [AUTH] "http://HOST:PORT/api/v1/rules" \
  | jq '[.data.groups[].rules[] | select(.type == "recording")] | .[] | {name: .name, health: .health}'
```

### Runtime Info & Config

```bash
# Build info
curl -s [AUTH] "http://HOST:PORT/api/v1/status/buildinfo" | jq .

# Runtime config
curl -s [AUTH] "http://HOST:PORT/api/v1/status/runtimeinfo" | jq .

# TSDB stats (cardinality)
curl -s [AUTH] "http://HOST:PORT/api/v1/status/tsdb" \
  | jq '{seriesCount: .data.seriesCountByMetricName[:10], labelValueCount: .data.labelValueCountByLabelName[:10]}'
```

### Metadata (metric type + help)

```bash
curl -s [AUTH] "http://HOST:PORT/api/v1/metadata?metric=METRIC_NAME" | jq .
```

## Common PromQL Patterns

Translate user requests into PromQL:

| User says | PromQL |
|---|---|
| "CPU usage" | `rate(process_cpu_seconds_total[5m])` or `100 - avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100` |
| "memory usage" | `process_resident_memory_bytes` or `node_memory_MemUsed_bytes` |
| "request rate" | `rate(http_requests_total[5m])` |
| "error rate" | `rate(http_requests_total{status=~"5.."}[5m])` |
| "error ratio" | `rate(http_requests_total{status=~"5.."}[5m]) / rate(http_requests_total[5m])` |
| "p99 latency" | `histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))` |
| "disk usage" | `1 - node_filesystem_avail_bytes / node_filesystem_size_bytes` |
| "pod restarts" | `increase(kube_pod_container_status_restarts_total[1h])` |
| "up/down targets" | `up` |

## Output Formatting

- Instant query with 1 result → show value + labels
- Instant query with multiple results → table: labels | value
- Range query → summarize: metric name, time range, min/max/last value
- Targets → healthy count / total, list unhealthy with error
- Alerts → firing count, list by severity
- Empty result (`"result":[]`) → say so clearly, suggest checking metric name with label list

## Config Management (`/prometheus config`)

Config file: `~/.claude/prometheus-instances.json`  
Use `Read` + `Write` tools only — no shell commands.

### `/prometheus config add`

Ask for (only what's missing):
- **name** — alias: `prod`, `vm`, `thanos`
- **url** — full base URL: `http://prometheus:9090`
- **auth** (optional) — `user:pass` → `"auth"`, Bearer token → `"auth_header": "Bearer TOKEN"`
- **default** (optional) → `"default": true`, unset on others

### `/prometheus config list`

Read config, display table:

| Name | URL | Auth | Default |
|------|-----|------|---------|
| prod | http://... | — | ✓ |

### `/prometheus config remove <name>`

Remove entry, write back, confirm.

### `/prometheus config test [name]`

```bash
curl -s [AUTH] "http://HOST:PORT/api/v1/query?query=up" | jq '.status'
```

Report: `success` → connected, error/connection refused → diagnose.

### Default Instance

One instance or one marked `"default": true` → use without asking.  
Multiple, none default → list and ask.

## Error Reference

| Error | Cause | Fix |
|---|---|---|
| `connection refused` | Wrong host/port or Prometheus down | Check URL |
| `bad_data` in response | Invalid PromQL syntax | Check expression |
| `"result":[]` | Metric not found, wrong labels, or wrong namespace label name (`namespace` vs `kubernetes_namespace`) | List metrics; inspect a known metric's label keys to find the correct namespace label name |
| `execution: query processing would load too many samples` | Query too broad + long range | Narrow time range or add label filters |
| 401 / 403 | Auth missing or wrong | Check auth config |
| `invalid parameter "query"` | Missing URL encoding | Use `--data-urlencode` |
