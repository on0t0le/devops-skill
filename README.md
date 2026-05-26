# devops-skill

Claude Code skills for DevOps workflows.

## Installation

**Via Claude Code CLI** (terminal):

```bash
claude plugin install https://github.com/on0t0le/devops-skill
```

**From within a Claude Code session** — type this as a prompt:

```
! claude plugin install https://github.com/on0t0le/devops-skill
```

The `!` prefix runs the command in your shell and pipes output back into the conversation.

**Update to latest:**

```bash
claude plugin update devops-skill
```

**Manual (symlink for live edits):**

```bash
git clone https://github.com/on0t0le/devops-skill
ln -s $(pwd)/devops-skill/skills/elasticsearch ~/.claude/skills/elasticsearch
ln -s $(pwd)/devops-skill/skills/loki ~/.claude/skills/loki
ln -s $(pwd)/devops-skill/skills/prometheus ~/.claude/skills/prometheus
ln -s $(pwd)/devops-skill/skills/kubernetes-investigator ~/.claude/skills/kubernetes-investigator
ln -s $(pwd)/devops-skill/skills/aws-investigator ~/.claude/skills/aws-investigator
ln -s $(pwd)/devops-skill/skills/investigate ~/.claude/skills/investigate
```

## Skills

### `investigate/` ⭐

Multi-system incident investigation orchestrator. Spawns parallel Haiku subagents for Kubernetes, Prometheus, Loki, and Elasticsearch — synthesizes a unified root cause report.

**Example prompts:**
- `/investigate errors in myapp since 15:00`
- `investigate why service payments is slow`
- `something is wrong with the orders pod in production`

---

### `aws-investigator/`

Read-only AWS investigation via AWS CLI. Multiple named profiles/regions with persistent config.

**Covers:** EC2, CloudWatch Logs/Metrics/Alarms, CloudTrail, ELB/ALB/NLB target health, ASG scaling activity, EKS cluster/nodegroup health, CloudFront distributions + metrics, ECS tasks, RDS, Lambda.

**Config:** `~/.claude/aws-instances.json` (profile + region per named env)

**Example prompts:**
- `check EC2 instances in prod`
- `who deleted the security group yesterday`
- `show firing CloudWatch alarms`
- `check ECS service payments in staging`
- `query CloudWatch logs for errors in /aws/lambda/my-function last hour`

---

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

### `loki/`

Query Grafana Loki (and Grafana Cloud Logs) via `curl`. Multiple named instances, multi-tenant support.

**Features:**
- Range and instant LogQL queries
- List labels and label values
- List log streams
- Log rate / count metric queries
- Multi-tenant (`X-Scope-OrgID`) support
- Multi-instance config (`~/.claude/loki-instances.json`)

**Config commands:**
- `/loki config list`
- `/loki config add`
- `/loki config remove <name>`
- `/loki config test [name]`

**Example prompts:**
- `search loki for errors in app myapp in the last hour`
- `show loki labels`
- `query loki for logs from namespace production containing "timeout"`
- `count error logs per minute in loki prod`

---

### `kubernetes-investigator/`

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
