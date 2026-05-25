---
name: elasticsearch
description: >
  Query Elasticsearch clusters using curl. Use whenever the user wants to list indexes,
  search logs, inspect documents, check cluster health, or query any Elasticsearch/OpenSearch
  instance — even if they don't say "Elasticsearch" but mention log search, index listing,
  or "ES". Supports multiple named instances (prod, staging, etc.). Trigger: /es, /elasticsearch,
  "list indexes", "search logs in elasticsearch", "query my ES cluster", "show me ES logs".
trigger: /elasticsearch
---

# Elasticsearch via curl

All operations use `curl`. No MCP, no SDKs.

## Instance Resolution

Resolve the target instance in this order:

1. **URL in message** — user pastes `http://...` or `https://...` → use it directly
2. **Instance name in message** — user says "prod", "staging", "logs-cluster" → look up in config
3. **Config file** — `~/.claude/elasticsearch-instances.json` exists → list available instances and ask which
4. **Nothing** — ask: "What's the Elasticsearch endpoint? (e.g. `http://localhost:9200`)"

Config file format (`~/.claude/elasticsearch-instances.json`):
```json
{
  "instances": {
    "prod":    { "url": "http://prod-es:9200",    "auth": "user:pass" },
    "staging": { "url": "http://staging-es:9200"  },
    "logs":    { "url": "https://logs.internal:9200", "auth_header": "ApiKey abc123==" }
  }
}
```

`auth` → Basic auth (`-u user:pass`)  
`auth_header` → Raw `Authorization` header value (`-H "Authorization: ..."`)  
No auth field → unauthenticated

## curl Helpers

Build every command from these building blocks:

```bash
# Base call (no auth)
curl -s -X GET "http://HOST:PORT/ENDPOINT" -H "Content-Type: application/json"

# Basic auth
curl -s -u "user:pass" -X GET "http://HOST:PORT/ENDPOINT" -H "Content-Type: application/json"

# API key / custom auth header
curl -s -H "Authorization: ApiKey abc==" -X GET "http://HOST:PORT/ENDPOINT" -H "Content-Type: application/json"

# With body (POST/GET with DSL)
curl -s -u "user:pass" -X GET "http://HOST:PORT/INDEX/_search" \
  -H "Content-Type: application/json" \
  -d '{"query": {...}}'
```

Always use `-s` (silent). Pipe through `| python3 -m json.tool` or `| jq .` for readable output if available.

## Operations

### List Indexes

```bash
curl -s [AUTH] "http://HOST:PORT/_cat/indices?v&s=index&h=index,health,status,docs.count,store.size"
```

Shows: name, health, status, doc count, size. Sorted by name.

To filter: append `&index=logs-*` or pipe through `grep`.

### Cluster Health

```bash
curl -s [AUTH] "http://HOST:PORT/_cluster/health?pretty"
```

### Search Logs

**Simple query string** (user gives keywords/phrases):
```bash
curl -s [AUTH] -X GET "http://HOST:PORT/INDEX/_search?pretty" \
  -H "Content-Type: application/json" \
  -d '{
    "query": { "query_string": { "query": "USER_QUERY" } },
    "size": 20,
    "sort": [{ "@timestamp": { "order": "desc" } }]
  }'
```

**Time-ranged search** (user mentions "last hour", "today", "last 24h", etc.):
```bash
curl -s [AUTH] -X GET "http://HOST:PORT/INDEX/_search?pretty" \
  -H "Content-Type: application/json" \
  -d '{
    "query": {
      "bool": {
        "must": [{ "query_string": { "query": "USER_QUERY" } }],
        "filter": [{ "range": { "@timestamp": { "gte": "now-1h", "lte": "now" } } }]
      }
    },
    "size": 20,
    "sort": [{ "@timestamp": { "order": "desc" } }]
  }'
```

**Field-specific filter** (user says "where status=500", "field X equals Y"):
Add to `bool.filter`:
```json
{ "term": { "FIELD": "VALUE" } }
```

**Full-text + field combo**: combine `must` (query_string) and `filter` (term/range) in bool query.

### Get Specific Document

```bash
curl -s [AUTH] "http://HOST:PORT/INDEX/_doc/DOC_ID?pretty"
```

### Index Mapping (field names/types)

```bash
curl -s [AUTH] "http://HOST:PORT/INDEX/_mapping?pretty"
```

Useful when user asks "what fields are available" or search returns unexpected results.

### Aggregations (counts, stats)

```bash
curl -s [AUTH] -X GET "http://HOST:PORT/INDEX/_search?pretty" \
  -H "Content-Type: application/json" \
  -d '{
    "size": 0,
    "aggs": {
      "by_field": { "terms": { "field": "FIELD.keyword", "size": 20 } }
    }
  }'
```

## Output Formatting

After running curl:

1. If result is large (>50 hits), summarize: total hits, first N results with key fields
2. For log search: show `@timestamp`, `level`/`severity`, `message`, plus any fields user asked about
3. For errors in response (`"error":` in JSON), explain the error clearly and suggest fixes
4. If index doesn't exist, suggest running list-indexes to find the right name

## Multi-Instance Workflows

When user compares across instances ("check prod and staging") or doesn't specify:

1. If config has multiple instances and none specified → show list, ask which one (or "all")
2. For "all" → run sequentially, label each result block with instance name
3. Never run destructive ops (DELETE, index creation) against "all"

## Wildcard Index Patterns

ES supports glob patterns. Pass through directly:
- `logs-*` matches all logs indexes
- `logs-2024.*` matches date-sharded indexes
- `*` matches everything (warn user this may be slow)

## Config Management (`/elasticsearch config`)

Trigger on: `/elasticsearch config`, "add elasticsearch instance", "save this ES endpoint", "configure ES", "remember this cluster", "set default ES".

Config file: `~/.claude/elasticsearch-instances.json`

### `/elasticsearch config add`

Collect from user (ask only what's missing):
- **name** — short alias, e.g. `prod`, `staging`, `local`
- **url** — full base URL, e.g. `http://es-prod:9200`
- **auth** (optional) — one of:
  - `user:pass` → stored as `"auth": "user:pass"`
  - API key string → stored as `"auth_header": "ApiKey <key>"`
  - nothing → omit auth fields
- **default** (optional) — ask "Make this the default instance?" → stored as `"default": true` (unset on others)

Then write:
```bash
# Read existing or start fresh
python3 -c "
import json, os, sys
path = os.path.expanduser('~/.claude/elasticsearch-instances.json')
cfg = json.load(open(path)) if os.path.exists(path) else {'instances': {}}
# ... apply changes ...
json.dump(cfg, open(path, 'w'), indent=2)
print('Saved.')
"
```

Confirm: "Saved **NAME** (`URL`). Reference it by name in future requests."

### `/elasticsearch config list`

Trigger on: "show ES instances", "what clusters do I have", "list configured ES".

Read and display `~/.claude/elasticsearch-instances.json` as a table:

| Name | URL | Auth | Default |
|------|-----|------|---------|
| prod | http://... | basic | ✓ |

### `/elasticsearch config remove <name>`

Trigger on: "remove ES instance", "delete ES config", "forget cluster NAME".

Remove named entry, write back, confirm.

### `/elasticsearch config test [name]`

Trigger on: "test ES connection", "check if ES is reachable".

Run `_cluster/health` against instance, report green/yellow/red or connection error.

### Default Instance

If config has exactly one instance, or one marked `"default": true` → use it without asking.  
If multiple and none is default → list them and ask which to use.

## Error Reference

| Error | Likely cause | Fix |
|-------|-------------|-----|
| `Connection refused` | ES not running or wrong port | Check URL/port |
| `index_not_found_exception` | Typo or wrong index name | Run list-indexes |
| `illegal_argument_exception` on `.keyword` | Field not mapped as keyword | Use field without `.keyword` |
| `search_phase_execution_exception` | Bad query DSL | Check query syntax |
| `authentication_required` / 401 | Missing or wrong credentials | Check auth config |
| `403 Forbidden` | Insufficient permissions | Check user role in ES |
