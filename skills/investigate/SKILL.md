---
name: investigate
description: >
  Multi-system DevOps incident investigation orchestrator. Spawns parallel Haiku
  subagents for Elasticsearch, Loki, Prometheus, Kubernetes, and AWS — then synthesizes
  a unified root cause report. ALWAYS use when the user reports an incident, alert,
  alarm, service degradation, or asks to investigate an issue in any service or
  infrastructure. This skill handles AWS-only incidents just as well as multi-system ones
  — if only AWS is configured, it spawns just the AWS subagent.
  Trigger: /investigate, "investigate issue", "something is wrong with X",
  "service is down", "investigate incident", "full investigation", "check everything",
  "what's causing errors in Y", "why is service X slow/failing",
  "received alert", "alert firing", "alarm triggered", "CloudWatch alarm",
  "5XX alert", "ApiGateway5XXError", "production alert", "monitoring alert",
  "investigate the issue in", "investigate in", "got an alert", "alert at",
  "received an alert", "pagerduty", "grafana alert", "opsgenie".
trigger: /investigate
allowed-tools:
  - Agent
  - Read
---

# DevOps Investigation Orchestrator

Three-phase investigation: broad sweep → cross-expert validation → unified synthesis.
Experts share findings with each other before the final report.

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
| **AWS** | EC2 health, ECS/EKS task failures, API Gateway 5XX, ELB target health, CloudWatch alarms, CloudTrail events, RDS issues |

Check which instances are configured:
- `~/.claude/prometheus-instances.json`
- `~/.claude/loki-instances.json`
- `~/.claude/elasticsearch-instances.json`
- `~/.claude/aws-instances.json`

Only spawn agents for configured systems (or URL/profile provided in message). Skip unconfigured ones silently — mention in final report if relevant.

**AWS-only mode**: If the incident is clearly AWS-based (API Gateway alert, CloudWatch alarm, ECS/EKS issue, RDS error, ALB health) AND no Loki/Prometheus/ES instances are configured, skip checking those config files and spawn just the AWS subagent immediately. AWS credentials from the current shell session count as "configured" — check with `aws sts get-caller-identity`. Don't block the investigation waiting for systems that aren't relevant.

**Symptom-to-system hints:**
- "ApiGateway5XXError", "5XX alert", API Gateway alarm → AWS subagent (API GW metrics + logs)
- "pod crashlooping", "OOMKilled", "deployment failed" → Kubernetes subagent
- "high latency", "error rate spike" → Prometheus + relevant app system
- "received alert from" + service name → check AWS first, then K8s if EKS involved

## Step 3: Phase 1 — Broad Parallel Sweep

Spawn ALL relevant subagents in a **single message** (one Agent tool call per system).
Each subagent uses `model: haiku` to minimize token cost.

**IMPORTANT:** Each Phase 1 agent must append a `## Cross-Validation Requests` section to its output listing specific questions it needs other systems to answer. Examples:
- "K8s → AWS: Node ip-10-0-1-42 NotReady since 14:32 — confirm EC2 instance health and whether it was spot-terminated"
- "K8s → Prometheus: 3 OOMKilled events on payment-api — confirm memory metric spike before 14:30"
- "AWS → K8s: ELB target 10.0.1.55:8080 marked unhealthy — which pod is this, what is its readiness state?"
- "Loki → Prometheus: error storm started at 14:28 — confirm request rate and error rate metrics at that time"

**Kubernetes subagent prompt template:**
```
You are a Kubernetes diagnostics expert. Use the kubernetes-investigator skill.

Task: Investigate [SERVICE/POD] in namespace [NAMESPACE].
Focus: [SYMPTOM — pod restarts, OOM, Pending, etc.]
Time context: [TIME RANGE]

Run read-only kubectl diagnostics. Report:
1. Pod status and recent events
2. Container states and restart count
3. Relevant log lines (last 50)
4. Root cause hypothesis
5. Recommended fix (do not apply)

## Cross-Validation Requests
End your report with this section listing specific questions for other systems.
Format: "[Your system] → [Target system]: [What you found] — [What you need confirmed]"
Examples of things to cross-validate:
- Node issues → ask AWS to check EC2 health / spot termination
- OOMKilled → ask Prometheus to confirm memory spike timing
- ImagePullBackOff → ask AWS to check ECR image/permissions
- Deployment at specific time → ask Loki/ES to correlate error onset

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
Return: metric values, trends (increasing/stable/decreasing), anomalies, exact time of any spike.

## Cross-Validation Requests
End your report with this section listing specific questions for other systems.
Format: "[Your system] → [Target system]: [What you found] — [What you need confirmed]"
Examples:
- Spike at specific time → ask K8s if a deployment/restart happened then
- Memory saturation → ask K8s for OOMKilled events
- Error rate spike → ask Loki/ES for log errors at that timestamp
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

## Cross-Validation Requests
End your report with this section listing specific questions for other systems.
Format: "[Your system] → [Target system]: [What you found] — [What you need confirmed]"
Examples:
- Error onset time → ask Prometheus to confirm metric spike at same time
- Connection refused errors → ask K8s if target pod was restarting
- AWS API errors in logs → ask AWS to check IAM/resource state
```

**AWS subagent prompt template:**
```
You are an AWS infrastructure investigator. Use the aws-investigator skill.

Task: Investigate [SERVICE] issue in AWS [PROFILE/REGION].
Focus: [SYMPTOM — EC2 down, ECS task failing, ALB unhealthy targets, alarms firing,
        API Gateway 5XX errors, EKS node issues, RDS errors, etc.]
Time context: [TIME RANGE]

IMPORTANT: Use the correct UTC timestamps for all CloudWatch/Logs queries.
The user's local time is [LOCAL TIME + TIMEZONE]. Convert to UTC before querying:
- CloudWatch metric queries: use ISO 8601 UTC strings ("2026-05-27T00:00:00Z")
- CloudWatch Logs Insights: use `date -u` or Python to compute Unix timestamps in UTC
  → python3 -c "import datetime; print(int(datetime.datetime(YYYY,MM,DD,HH,MM, tzinfo=datetime.timezone.utc).timestamp()))"
  Do NOT use macOS `date -j` with a Z-suffix input — it ignores the Z and uses local time.

Run these checks and report findings:
1. Any ALARM-state CloudWatch alarms related to [SERVICE]
2. API Gateway 5XX metrics if applicable — check both `5XXError` metric and execution logs
   - List REST APIs: `aws apigateway get-rest-apis`
   - Get 5XX metric per API by ApiName dimension
   - If execution logs enabled: query with correct UTC timestamps for status 5XX entries
3. ECS/EKS service status if applicable (desired vs running count, recent events)
4. NLB/ALB target health if applicable (unhealthy targets + reason)
5. CloudTrail: recent changes to [SERVICE] resources in [TIME RANGE]
6. RDS events/errors if database is involved

Return: resource state, anomalies, timeline of changes, root cause hypothesis.

## Cross-Validation Requests
End your report with this section listing specific questions for other systems.
Format: "[Your system] → [Target system]: [What you found] — [What you need confirmed]"
Examples:
- ELB unhealthy target IP → ask K8s which pod has that IP, check its readiness
- CloudTrail config change at time T → ask Loki/ES for errors starting at T
- Spot termination at time T → ask K8s for node evictions at T
- ASG scale-in event → ask Prometheus if traffic/load changed before it
- API GW 504 on health endpoint → ask K8s if pods were restarting at that time
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

## Cross-Validation Requests
End your report with this section listing specific questions for other systems.
Format: "[Your system] → [Target system]: [What you found] — [What you need confirmed]"
Examples:
- Error onset time → ask Prometheus for metric spike at same time
- DB connection errors → ask AWS to check RDS health at that time
- Error started after deployment → ask K8s for deployment events at that time
```

## Step 4: Phase 2 — Cross-Expert Validation

After all Phase 1 agents complete:

1. **Collect all `## Cross-Validation Requests`** from every agent's output.
2. **Group by target system** — deduplicate overlapping questions.
3. **Spawn targeted follow-up agents** (in parallel, one per target system that has requests).

Each follow-up agent receives:
- The original context (service, namespace, time range)
- The specific findings from the requesting expert verbatim
- The exact questions to answer

**Phase 2 subagent prompt template:**
```
You are a [TARGET SYSTEM] expert. Use the [TARGET SKILL] skill.

## Context from Phase 1 Investigation

The following findings were reported by other expert agents and require your validation:

### From Kubernetes expert:
[Paste K8s Cross-Validation Requests targeting this system, if any]

### From AWS expert:
[Paste AWS Cross-Validation Requests targeting this system, if any]

### From Prometheus expert:
[Paste Prometheus Cross-Validation Requests targeting this system, if any]

### From Loki expert:
[Paste Loki Cross-Validation Requests targeting this system, if any]

### From Elasticsearch expert:
[Paste ES Cross-Validation Requests targeting this system, if any]

## Your Task

Answer each question above with targeted queries. For each request:
1. State the question you're answering
2. Show the command/query you ran
3. Give the result (confirmed / refuted / inconclusive)
4. Add any new findings the question led you to discover

Service: [SERVICE] | Namespace: [NAMESPACE] | Time: [TIME RANGE]

Be concise. Don't re-investigate what wasn't asked — focus on the cross-validation questions only.
```

**Skip Phase 2** if no agent produced any Cross-Validation Requests, or all requests target unconfigured systems — proceed directly to Step 5.

**Announce Phase 2** to the user before spawning: `🔄 Phase 2: Cross-validating findings between experts…`

## Step 5: Synthesize Results

After Phase 1 and Phase 2 complete, write a unified incident report:

```
# Investigation Report: [SERVICE] — [TIME]

## Summary
[2-3 sentence executive summary: what's broken, likely cause, severity]

## Findings by System

### Kubernetes
[Key findings from K8s — pod state, restarts, events]

### AWS Infrastructure
[EC2/ECS/ELB state, CloudWatch alarms, CloudTrail changes]

### Metrics (Prometheus)
[Error rate, latency, anomalies, exact timestamps]

### Logs — Loki / Elasticsearch
[Error patterns, first occurrence, sample messages]

## Cross-Expert Validation Results
[What each expert confirmed or refuted for others — only include if Phase 2 ran]
- K8s → AWS confirmed: [finding]
- AWS → K8s confirmed: [finding]
- etc.

## Root Cause Analysis
[Synthesize across ALL phases: correlate timing of errors with pod restarts/deploys/metrics spikes.
Highlight where multiple systems independently confirm the same cause.]

## Timeline
- HH:MM — [first symptom from any system]
- HH:MM — [escalation or pod restart / AWS event / metric spike]
- HH:MM — [cross-validation confirms cause]
- HH:MM — [current state]

## Recommended Actions
1. [Most likely fix — specific, actionable, informed by cross-validation]
2. [Secondary action if root cause unclear]
3. [Monitoring suggestion]

**Note:** No changes were made. All diagnostics are read-only.
```

## Parallelism Rules

- Phase 1: spawn all agents **simultaneously** in one message
- Phase 2: spawn all follow-up agents **simultaneously** in one message (grouped by target system)
- Never wait for one Phase 1 agent before starting others
- Only run sequentially when output of one is strictly required for another (e.g., need pod name from K8s before querying logs by pod label)

## Context Passing

Pass specific context to each agent — don't make them guess:
- Exact service name, namespace, time range
- Instance name (from config) or URL
- Specific symptom to focus on
- **Phase 2:** verbatim findings from the requesting expert

Vague subagent prompts produce vague results. Be specific.
