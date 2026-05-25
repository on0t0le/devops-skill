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
- Multiple named instances (prod, staging, etc.)
- Supports Basic auth and API key auth

**Install:**

```bash
cp -r elasticsearch ~/.claude/skills/
```

Or symlink for live edits:

```bash
ln -s $(pwd)/elasticsearch ~/.claude/skills/elasticsearch
```

**Instance config** (`~/.claude/elasticsearch-instances.json`):

```json
{
  "instances": {
    "prod":    { "url": "http://prod-es:9200", "auth": "user:pass" },
    "staging": { "url": "http://staging-es:9200" },
    "logs":    { "url": "https://logs.internal:9200", "auth_header": "ApiKey abc123==" }
  }
}
```

**Example prompts:**
- `list indexes on http://localhost:9200`
- `search prod for logs with status 500 in the last hour`
- `show me indexes on both prod and staging`
- `/es search logs-* "OutOfMemoryError" last 24h`
