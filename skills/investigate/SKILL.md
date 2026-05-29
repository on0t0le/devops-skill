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
  - Bash(date *)
---

# DevOps Investigation Orchestrator

## Step 1: Clarify Scope (if needed)

Before spawning agents, ensure you have:
- **Service/app name** — what to investigate
- **Time range** — "last hour", "since 14:00", "last 30 min" (default: last 1 hour)
- **Namespace** — for Kubernetes (ask if unknown)
- **Symptom** — errors, slowness, OOM, pod restarts, no traffic, etc.

If user gave enough context, proceed immediately. Don't ask for what you can infer.

## Step 2: Determine Which Systems to Query

Check which instances are configured:
- `~/.claude/prometheus-instances.json`
- `~/.claude/loki-instances.json`
- `~/.claude/elasticsearch-instances.json`
- `~/.claude/aws-instances.json`

Only spawn agents for configured systems. Skip unconfigured ones silently.

**System selection rules:**

| System | Query when |
|---|---|
| **Kubernetes** | Service runs in K8s, or namespace given — always start here |
| **Loki** | Loki configured AND namespace confirmed present in Loki (verify first) |
| **Elasticsearch** | ES configured AND service known to ship logs there, or Loki has no data for namespace |
| **Prometheus** | Metrics: error rate, latency, resource saturation, alerts firing |
| **AWS** | EC2/ECS/ELB/RDS/API Gateway issues, CloudWatch alarms, CloudTrail events |

**Log system selection — verify before assuming:**
When namespace is known and Loki is configured, check if logs exist before spawning a full log agent:
```bash
curl -s "LOKI_URL/loki/api/v1/label/namespace/values" | jq '.data[] | select(. == "NAMESPACE")'
```
- Namespace found in Loki → use Loki for log search
- Namespace not found in Loki → fall back to ES (if configured), or ask user where logs are shipped
- Neither configured → note the gap in the final report, do not assume

Do not hardcode "K8s always uses Loki" — some clusters ship pod logs to ES or elsewhere.

**AWS-only mode**: If the incident is AWS-based AND no Loki/Prometheus/ES instances are configured,
skip checking those config files and spawn just the AWS subagent immediately.

**Symptom-to-system hints:**
- Pod crashlooping, OOMKilled, deployment failed → start with K8s
- High latency, error rate spike → start with Prometheus
- AWS alert, CloudWatch alarm → start with AWS
- 5XX errors on K8s service → start with K8s

## Step 3: Investigation Strategy

**One agent at a time. Always.**

Never spawn multiple agents simultaneously. Spawn one, read its findings, decide what to spawn next.
Pass ALL accumulated findings to each subsequent agent so it has full context.

To revisit a system already queried (e.g. K8s again after Prometheus reveals something),
spawn a fresh agent of that type and pass the full accumulated context — same pattern as the first time.

### Decision chain

**K8s service (namespace given):**
1. Spawn K8s agent
2. Read findings:
   - HIGH confidence root cause (OOMKilled, CrashLoop + clear error) → spawn ONE targeted follow-up only (e.g. Prometheus for OOM memory confirmation), then synthesize
   - Ambiguous → spawn Loki (verify namespace first), read findings, then Prometheus if still unclear
   - All pods healthy → spawn Prometheus, then Loki if metrics show nothing

**AWS-based incident (no K8s namespace):**
1. Spawn AWS agent
2. Read findings, spawn Prometheus or K8s if cross-check needed

**Log gap (Loki namespace check failed):**
→ Spawn ES agent instead; note in report that Loki had no data for namespace

**Re-querying a system:**
→ Spawn new agent of that type; pass verbatim findings from all prior agents in the prompt

### When to stop

Stop spawning when:
- Root cause confirmed with evidence from at least one system
- All relevant configured systems have been queried with no new signal
- Two consecutive agents return no new findings

## Step 4: Spawning Agents

⚠️ **Agent tool call requirements — non-negotiable:**
- Set `model: "haiku"` on every Agent call.
- Set `subagent_type: "general-purpose"` on every Agent call.
- One agent per message. Never spawn two agents in the same message.

**Timestamps rule (applies to ALL subagents):**
YOU (the orchestrator) compute timestamps and embed them as integer literals in the subagent prompt.
Do NOT instruct subagents to compute timestamps.

**IMPORTANT — RTK hook constraints:**
- One bash call per timestamp. Never chain with `&&`.
- Never use `$((VAR * N))` arithmetic — RTK blocks variable arithmetic expansion.
- Compute nanoseconds by mentally appending 9 zeros: if START_S=1748498700, START_NS=1748498700000000000.

For absolute times, run TWO separate bash calls:
```bash
# Call 1 — start time
TZ=UTC date -j -f "%Y-%m-%d %H:%M:%S" "YYYY-MM-DD HH:MM:SS" +%s
```
```bash
# Call 2 — end time
TZ=UTC date -j -f "%Y-%m-%d %H:%M:%S" "YYYY-MM-DD HH:MM:SS" +%s
```

For relative times, run TWO separate bash calls (macOS):
```bash
date -v-30M +%s   # start: 30 min ago
```
```bash
date +%s          # end: now
```

Embed the resulting integer literals directly in the subagent prompt — subagents never compute timestamps.

Subagents receive timestamps as literals, e.g.:
- `START_S=1780038600`
- `END_S=1780040400`
- `START_NS=1780038600000000000`
- `END_NS=1780040400000000000`

**One curl per bash call.** Never chain multiple curl commands in a single bash call.
Each query = one small, focused bash call that's easy to review.

---

**Kubernetes subagent prompt template:**
```
FIRST ACTION — before anything else, invoke the Skill tool:
  skill: "devops-skill:kubernetes-investigator"

SECOND ACTION — immediately after, invoke:
  skill: "devops-skill:kubernetes-troubleshooting-flow"

Do NOT run kubectl, bash, or any other tool until both skills are loaded.

Task: Investigate [SERVICE/POD] in namespace [NAMESPACE].
Focus: [SYMPTOM — pod restarts, OOM, Pending, 5XX errors, etc.]
Time context: [TIME RANGE UTC]

If service name given but no pod name: start with namespace-wide triage —
list all non-Running pods, check Warning events, identify worst offenders.

Run read-only kubectl diagnostics per the skill. Report:
1. Pod status and recent events
2. Container states and restart count
3. Relevant log lines (last 50)
4. Root cause hypothesis with evidence (cite specific kubectl output lines)
5. Recommended fix (do not apply)
6. Confidence: HIGH (OOMKilled/CrashLoop/clear error) or LOW (ambiguous)

## Follow-up Needed
End your report with this section. State which system to query next and exactly what to check.
Fill all placeholders with actual values from kubectl output — no blanks.
- OOMKilled → next: Prometheus — confirm memory spike for pod [POD] at [TIMESTAMP], limit [LIMIT]
- CrashLoopBackOff → next: Loki — check error logs for namespace=[NS], app=[APP] around [TIMESTAMP]
- Node NotReady → next: AWS — check EC2 instance [INSTANCE_ID] health at [TIMESTAMP]
- Pending (VPC CNI) → next: AWS — check nodegroup + subnets in region [REGION]
- 5XX + pod failures → next: Prometheus — error rate for [SERVICE] + Loki errors in [NS] at [TIMESTAMP]
- Root cause confirmed, no ambiguity → next: none
```

**Prometheus subagent prompt template:**
```
FIRST ACTION — before anything else, invoke the Skill tool:
  skill: "devops-skill:prometheus"
Do NOT run curl, bash, or any other tool until the skill has loaded.

Task: Query metrics for service [SERVICE] in namespace [NAMESPACE].
Prometheus instance: [NAME or URL]
START_S=[EPOCH]  END_S=[EPOCH]  (pre-computed, use as literals)

Run each query as a SEPARATE bash call:

Call 1 — firing alerts:
curl -s -G "URL/api/v1/query" --data-urlencode 'query=ALERTS{alertstate="firing"}' --data-urlencode "time=END_S"

Call 2 — HTTP 5xx error rate:
curl -s -G "URL/api/v1/query_range" --data-urlencode 'query=sum by(job)(increase(http_requests_total{status=~"5.."}[10m]))' --data-urlencode "start=START_S" --data-urlencode "end=END_S" --data-urlencode "step=60"

Call 3 — pod restarts:
curl -s -G "URL/api/v1/query" --data-urlencode 'query=topk(5,increase(kube_pod_container_status_restarts_total{namespace="[NAMESPACE]"}[30m]))' --data-urlencode "time=END_S"

Call 4 — if 5xx found, check error rate by ingress/service:
curl -s -G "URL/api/v1/query_range" --data-urlencode 'query=sum by(ingress,service)(increase(nginx_ingress_controller_requests{status=~"5.."}[10m]))' --data-urlencode "start=START_S" --data-urlencode "end=END_S" --data-urlencode "step=60"

Adapt metric names if empty — discover with label/job/values first.
Return: metric values, trends, anomalies, exact time of any spike.

## Follow-up Needed
End with this section. State next system to query and exact question, or "none" if root cause confirmed.
```

**Loki subagent prompt template:**
```
FIRST ACTION — before anything else, invoke the Skill tool:
  skill: "devops-skill:loki"
Do NOT run curl, bash, or any other tool until the skill has loaded.

Task: Search logs for [SERVICE] in namespace [NAMESPACE].
Loki instance: [NAME or URL]
START_NS=[EPOCH_NS]  END_NS=[EPOCH_NS]  (pre-computed nanoseconds, use as literals)

Run each step as a SEPARATE bash call:

Call 1 — verify namespace exists:
curl -s "LOKI_URL/loki/api/v1/label/namespace/values"
(If [NAMESPACE] not in result, stop and report "namespace not found in Loki".)

Call 2 — recent log sample (format discovery):
curl -s -G "LOKI_URL/loki/api/v1/query_range" --data-urlencode 'query={namespace="[NAMESPACE]",app=~"[SERVICE].*"}' --data-urlencode "start=START_NS" --data-urlencode "end=END_NS" --data-urlencode "limit=20" --data-urlencode "direction=backward"

Call 3 — error logs:
curl -s -G "LOKI_URL/loki/api/v1/query_range" --data-urlencode 'query={namespace="[NAMESPACE]",app=~"[SERVICE].*"} |= "error"' --data-urlencode "start=START_NS" --data-urlencode "end=END_NS" --data-urlencode "limit=50"

Return: error log samples (5-10 most relevant), error patterns, first/last occurrence times.

## Follow-up Needed
End with this section. State next system to query and exact question, or "none" if root cause confirmed.
```

**AWS subagent prompt template:**
```
FIRST ACTION — before anything else, invoke the Skill tool:
  skill: "devops-skill:aws-investigator"
Do NOT run aws CLI, bash, or any other tool until the skill has loaded.

Task: Investigate [SERVICE] issue in AWS [PROFILE/REGION].
Focus: [SYMPTOM]
START_ISO=[ISO8601_UTC]  END_ISO=[ISO8601_UTC]  (pre-computed, use as literals)

Run each check as a SEPARATE bash call:

Call 1 — CloudWatch alarms in ALARM state:
aws cloudwatch describe-alarms --state-value ALARM --profile [PROFILE] --region [REGION]

Call 2 — ECS service status (if applicable):
aws ecs describe-services --cluster [CLUSTER] --services [SERVICE] --profile [PROFILE] --region [REGION]

Call 3 — ALB/NLB target health:
aws elbv2 describe-target-health --target-group-arn [ARN] --profile [PROFILE] --region [REGION]

Call 4 — CloudTrail recent events for service:
aws cloudtrail lookup-events --lookup-attributes AttributeKey=ResourceName,AttributeValue=[SERVICE] --start-time START_ISO --end-time END_ISO --profile [PROFILE] --region [REGION]

Skip calls where resource names are unknown — note the gap.

Return: resource state, anomalies, timeline of changes, root cause hypothesis.

## Follow-up Needed
End with this section. State next system to query and exact question, or "none" if root cause confirmed.
```

**Elasticsearch subagent prompt template:**
```
FIRST ACTION — before anything else, invoke the Skill tool:
  skill: "devops-skill:elasticsearch"
Do NOT run curl, bash, or any other tool until the skill has loaded.

NOTE: Use ES only for application business logs — NOT for K8s pod/infrastructure logs
(those are in Loki). Scope search to [SERVICE]-specific application indexes only.

Task: Search logs for [SERVICE] in the last [TIME RANGE].
ES instance: [NAME or URL]
Time window: [START UTC] to [END UTC]

START_ISO=[ISO8601_UTC]  END_ISO=[ISO8601_UTC]  (pre-computed, use as literals)

Run each step as a SEPARATE bash call:

Call 1 — list indexes matching service:
curl -s "ES_URL/_cat/indices/[SERVICE]*?h=index,docs.count&s=index"

Call 2 — search errors in time window:
curl -s -X GET "ES_URL/[INDEX]/_search" -H "Content-Type: application/json" -d '{"query":{"bool":{"must":[{"range":{"@timestamp":{"gte":"START_ISO","lte":"END_ISO"}}},{"query_string":{"query":"error OR exception OR fatal"}}]}},"size":10,"sort":[{"@timestamp":{"order":"desc"}}]}'

Return: error count, sample error messages (5-10), any stack traces, first/last occurrence.

## Follow-up Needed
End with this section. State next system to query and exact question, or "none" if root cause confirmed.
```

## Step 5: Synthesize Results

Write a unified incident report:

```
# Investigation Report: [SERVICE] — [TIME]

## Summary
[2-3 sentence executive summary: what's broken, likely cause, severity]

## Findings

### Kubernetes
[Pod state, restarts, events — with specific timestamps]

### Metrics (Prometheus)
[Error rate, memory/CPU, anomalies — only if queried]

### Logs — Loki
[Error patterns, first occurrence, sample messages — only if queried]

### AWS Infrastructure
[EC2/ECS/ELB state, alarms, CloudTrail — only if queried]

## Root Cause Analysis
[Synthesize: correlate timing across systems. Cite specific evidence.]

## Timeline
- HH:MM UTC — [first symptom]
- HH:MM UTC — [escalation / restart / spike]
- HH:MM UTC — [current state]

## Recommended Actions
1. [Most likely fix — specific and actionable]
2. [Secondary action if root cause unclear]
3. [Monitoring/alerting improvement]
```

Omit sections for systems that were not queried.

**Note:** No changes were made. All diagnostics are read-only.
