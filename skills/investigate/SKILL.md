---
name: investigate
description: >
  Multi-system DevOps incident investigation orchestrator. Spawns parallel Haiku
  subagents for Elasticsearch, Loki, Prometheus, and Kubernetes — then synthesizes
  a unified root cause report. Use when the user reports an incident, service
  degradation, or asks to investigate an issue across multiple systems.
  Trigger: /investigate, "investigate issue", "something is wrong with X",
  "service is down", "investigate incident", "full investigation", "check everything",
  "what's causing errors in Y", "why is service X slow/failing".
trigger: /investigate
---

# DevOps Investigation Orchestrator

Parallel multi-system investigation using specialized Haiku subagents.
Each subagent loads its own skill and runs independently.

## Step 1: Clarify Scope (if needed)

Before spawning agents, ensure you have:
- **Service/app name** — what to investigate
- **Time range** — "last hour", "since 14:00", "last 30 min" (default: last 1 hour)
- **Namespace** — for Kubernetes (ask if unknown)
- **Symptom** — errors, slowness, OOM, pod restarts, no traffic, etc.

If user gave enough context (e.g. "investigate errors in myapp since 15:00"), proceed immediately. Don't ask for what you can infer.

## Step 2: Determine Which Systems to Query

Select based on context. When in doubt, query all configured systems.

| System | Query when |
|---|---|
| **Kubernetes** | Pod failures, restarts, deployments, OOM, Pending — or always for service issues |
| **Prometheus** | Metrics: error rate, latency, saturation, alerts firing |
| **Loki** | Structured log search, Kubernetes-native logs via label selectors |
| **Elasticsearch** | App logs if ES instances are configured |
| **AWS** | EC2 health, ECS task failures, ELB target health, CloudWatch alarms, CloudTrail events |

Check which instances are configured:
- `~/.claude/prometheus-instances.json`
- `~/.claude/loki-instances.json`
- `~/.claude/elasticsearch-instances.json`
- `~/.claude/aws-instances.json`

Only spawn agents for configured systems (or URL/profile provided in message). Skip unconfigured ones silently — mention in final report if relevant.

## Step 3: Spawn Subagents in Parallel

Spawn ALL relevant subagents in a single message (one Agent tool call per system).
Each subagent uses `model: haiku` to minimize token cost.

**Kubernetes subagent prompt template:**
```
You are a Kubernetes diagnostics expert. Use the kubernetes-expert skill.

Task: Investigate [SERVICE/POD] in namespace [NAMESPACE].
Focus: [SYMPTOM — pod restarts, OOM, Pending, etc.]
Time context: [TIME RANGE]

Run read-only kubectl diagnostics. Report:
1. Pod status and recent events
2. Container states and restart count
3. Relevant log lines (last 50)
4. Root cause hypothesis
5. Recommended fix (do not apply)

Be concise. Return structured findings only.
```

**Prometheus subagent prompt template:**
```
You are a Prometheus metrics expert. Use the prometheus skill.

Task: Query metrics for service [SERVICE] in the last [TIME RANGE].
Prometheus instance: [NAME or URL]

Run these queries and report findings:
1. Error rate: rate(http_requests_total{job="[SERVICE]",status=~"5.."}[5m])
2. Request rate: rate(http_requests_total{job="[SERVICE]"}[5m])
3. Any firing alerts related to [SERVICE]
4. P99 latency if histogram metrics exist

Adapt metric names if the above don't return results — list available metrics for [SERVICE] first.
Return: metric values, trends (increasing/stable/decreasing), anomalies.
```

**Loki subagent prompt template:**
```
You are a Loki log search expert. Use the loki skill.

Task: Search logs for service [SERVICE] in the last [TIME RANGE].
Loki instance: [NAME or URL]

Run these searches:
1. All logs: {app="[SERVICE]"} — last 20 lines to understand log format
2. Errors: {app="[SERVICE]"} |= "error" OR {app="[SERVICE]"} | json | level="error"
3. If namespace known: {namespace="[NAMESPACE]", app="[SERVICE]"} |= "error"

Return: error log samples (5-10 most relevant), error patterns, first/last occurrence times.
```

**AWS subagent prompt template:**
```
You are an AWS infrastructure investigator. Use the aws-investigator skill.

Task: Investigate [SERVICE] issue in AWS [PROFILE/REGION].
Focus: [SYMPTOM — EC2 down, ECS task failing, ALB unhealthy targets, alarms firing, etc.]
Time context: [TIME RANGE]

Run these checks and report findings:
1. Any ALARM-state CloudWatch alarms related to [SERVICE]
2. ECS service status if applicable (desired vs running count, recent events)
3. ELB target health if applicable (unhealthy targets + reason)
4. CloudTrail: recent changes to [SERVICE] resources in [TIME RANGE]
5. CloudWatch Logs: errors in relevant log group in [TIME RANGE]

Return: resource state, anomalies, timeline of changes, suspected cause.
```

**Elasticsearch subagent prompt template:**
```
You are an Elasticsearch log search expert. Use the elasticsearch skill.

Task: Search logs for [SERVICE] in the last [TIME RANGE].
ES instance: [NAME or URL]

Search strategy:
1. List indexes matching [SERVICE] or app_logs-* pattern
2. Search for "error" OR "exception" OR "fatal" in relevant index
3. Include time filter for [TIME RANGE]

Return: error count, sample error messages (5-10), any stack traces, first/last occurrence.
```

## Step 4: Synthesize Results

After all subagents complete, write a unified incident report:

```
# Investigation Report: [SERVICE] — [TIME]

## Summary
[2-3 sentence executive summary: what's broken, likely cause, severity]

## Findings by System

### Kubernetes
[Key findings from K8s subagent — pod state, restarts, events]

### AWS Infrastructure
[EC2/ECS/ELB state, CloudWatch alarms, CloudTrail changes]

### Metrics (Prometheus)
[Error rate, latency, any anomalies]

### Logs — Loki / Elasticsearch
[Error patterns, first occurrence, sample messages]

## Root Cause Analysis
[Synthesize across systems: correlate timing of errors with pod restarts/deploys/metrics spikes]

## Timeline
- HH:MM — [first symptom from any system]
- HH:MM — [escalation or pod restart]
- HH:MM — [current state]

## Recommended Actions
1. [Most likely fix — specific, actionable]
2. [Secondary action if root cause unclear]
3. [Monitoring suggestion]

**Note:** No changes were made. All diagnostics are read-only.
```

## Parallelism Rules

- Spawn Kubernetes + Prometheus + Loki + ES agents **all at once** in one message
- Only run sequentially if output of one is needed for another (rare — e.g. need pod name before querying logs by pod label)
- Don't wait for one before starting others

## Context Passing

Pass specific context to each agent — don't make them guess:
- Exact service name, namespace, time range
- Instance name (from config) or URL
- Specific symptom to focus on

Vague subagent prompts produce vague results. Be specific.
