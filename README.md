# devops-skill

Claude Code skills for DevOps workflows.

**Install all skills:**

```bash
claude plugin install https://github.com/on0t0le/devops-skill
```

## Skills

### `prometheus/`

Query Prometheus (and VictoriaMetrics, Thanos, Cortex, Mimir) via `curl`. Multiple named instances with persistent config.

**Features:**
- Instant and range PromQL queries
- List metrics, labels, label values
- Targets health (scrape status)
- Firing alerts and rule status
- TSDB cardinality stats
- Multi-instance config (`~/.claude/prometheus-instances.json`)

**Config commands:**
- `/prometheus config list`
- `/prometheus config add`
- `/prometheus config remove <name>`
- `/prometheus config test [name]`

**Example prompts:**
- `query prometheus for http error rate in the last hour`
- `show firing alerts on prod prometheus`
- `list all metrics starting with kube_pod`
- `check prometheus targets for unhealthy scrapes`

---

### `kubernetes-expert/`

Read-only Kubernetes diagnostics expert. Investigates Pod, Namespace, and Workload issues without touching cluster state.

**Handles:** CrashLoopBackOff, OOMKilled, Pending, ImagePullBackOff, Evicted, Init failures, missing ConfigMaps/Secrets, node pressure, quota exhaustion.

**Safe by design:** `kubectl get/describe/logs/top` only. No exec, no apply, no delete. No mutating HTTP calls.

**Example prompts:**
- `debug pod my-app-xyz in namespace production`
- `why is my deployment crashing`
- `investigate all failing pods in the staging namespace`
- `pod is OOMKilled, what's wrong`

---

### `elasticsearch/`

Query Elasticsearch/OpenSearch clusters via `curl` — no MCP, no SDKs required.

**Features:**
- List indexes with health, doc count, size
- Search logs (query string, field filters, time ranges, aggregations)
- Cluster health check
- Index mapping inspection
- Multiple named instances with persistent config
- Basic auth and API key auth

**Install via Claude Code plugin** (recommended):

```bash
claude plugin install https://github.com/abuhrovyi/devops-skill
```

Or copy manually:

```bash
cp -r skills/elasticsearch ~/.claude/skills/
```

Symlink for live edits:

```bash
ln -s $(pwd)/skills/elasticsearch ~/.claude/skills/elasticsearch
```

**Configure instances** (persisted to `~/.claude/elasticsearch-instances.json`):

```
add elasticsearch instance http://es-prod:9200 named prod
add auth user:pass to prod
```

Or edit directly:

```json
{
  "instances": {
    "prod":    { "url": "http://prod-es:9200", "auth": "user:pass", "default": true },
    "staging": { "url": "http://staging-es:9200" },
    "logs":    { "url": "https://logs.internal:9200", "auth_header": "ApiKey abc123==" }
  }
}
```

**Config commands:**
- `/elasticsearch config list` — show all instances
- `/elasticsearch config add` — guided add
- `/elasticsearch config remove <name>` — remove instance
- `/elasticsearch config test [name]` — check connectivity

**Example prompts:**
- `list indexes on http://localhost:9200`
- `search prod for logs with status 500 in the last hour`
- `show me indexes on both prod and staging`
- `/elasticsearch search logs-* "OutOfMemoryError" last 24h`
