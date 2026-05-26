# devops-skill

Claude Code skills for DevOps workflows.

## Skills

### `elasticsearch/`

Query Elasticsearch/OpenSearch clusters via `curl` — no MCP, no SDKs required.

**Features:**
- List indexes with health, doc count, size
- Search logs (query string, field filters, time ranges, aggregations)
- Cluster health check
- Index mapping inspection
- Multiple named instances with persistent config
- Basic auth and API key auth

**Install:**

```bash
cp -r elasticsearch ~/.claude/skills/
```

Symlink for live edits:

```bash
ln -s $(pwd)/elasticsearch ~/.claude/skills/elasticsearch
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
