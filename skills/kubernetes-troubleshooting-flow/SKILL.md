---
name: kubernetes-troubleshooting-flow
description: >
  Knowledge skill: Kubernetes failure-mode to observability correlation patterns.
  Maps kubectl findings to specific PromQL queries, LogQL searches, and AWS cross-checks.
  Loaded by kubernetes-investigator before generating cross-validation requests.
  Not for direct invocation — used internally by the investigation workflow.
---

# Kubernetes → Observability Correlation Patterns

## Failure Mode → Prometheus PromQL

Use these templates when kubectl reveals the failure. Replace NS/POD/WORKLOAD/SERVICE placeholders.

| kubectl finding | PromQL |
|---|---|
| OOMKilled — memory check | `container_memory_working_set_bytes{namespace="NS",pod="POD"}` |
| OOMKilled — limit vs usage | `kube_pod_container_resource_limits{resource="memory",namespace="NS",pod=~"WORKLOAD.*"}` |
| High restarts | `increase(kube_pod_container_status_restarts_total{namespace="NS",pod="POD"}[1h])` |
| 5XX / service errors | `rate(http_requests_total{namespace="NS",status=~"5.."}[5m])` |
| Node NotReady | `kube_node_status_condition{condition="Ready",status="true",node="NODE"}` |
| Pending pods count | `kube_pod_status_phase{namespace="NS",phase="Pending"}` |
| CPU throttling | `rate(container_cpu_cfs_throttled_seconds_total{namespace="NS",pod="POD"}[5m])` |

Namespace label varies — try both if empty result:
- `{namespace="NS"}` — Prometheus Operator / ServiceMonitor default
- `{kubernetes_namespace="NS"}` — custom scrape configs, some exporters

## Failure Mode → Loki LogQL

| kubectl finding | LogQL |
|---|---|
| CrashLoopBackOff | `{namespace="NS",pod="POD"} \| json \| level="error"` |
| CrashLoopBackOff — raw | `{namespace="NS",pod="POD"} \|= "error"` |
| OOMKilled — app-level OOM | `{namespace="NS",pod="POD"} \|~ "out of memory\|OOM\|killed\|signal 9"` |
| Init container failing | `{namespace="NS",container="INIT_CONTAINER_NAME"}` |
| Service errors (namespace) | `{namespace="NS",app="SERVICE"} \|= "error"` |
| 5XX errors in logs | `{namespace="NS"} \|~ "5[0-9]{2}\|internal server error\|panic"` |

Time range (nanosecond epoch):
- Last 1h: `start=$(( $(date +%s) - 3600 ))000000000`
- Last 15m: `start=$(( $(date +%s) - 900 ))000000000`

## Failure Mode → AWS Cross-Check

| kubectl finding | AWS investigation |
|---|---|
| Node NotReady / ContainerStatusUnknown | EC2 instance health (system + instance checks); spot interruption events in CloudTrail |
| Pending — no schedulable node | EKS nodegroup status + health.issues; ASG desired vs actual; ASG launch failures |
| Pending — VPC CNI IP exhaustion | Subnet `AvailableIpAddressCount`; `/28` prefix capacity per subnet |
| ImagePullBackOff (ECR) | ECR repo exists; node IAM role has `ecr:GetAuthorizationToken` |
| Node capacity / OOM at node level | ASG instance health; CloudWatch `MemoryUtilization` on EC2 |
| Spot interruption | CloudTrail `BidEvictedEvent`; ASG scaling activity with `Spot` termination |
| Repeated node issues in one AZ | Spot price history across AZs; ASG subnet configuration |

## Extract EC2 Instance ID from Node Name

```bash
# Node ProviderID contains the instance ID
kubectl get node <NODE_NAME> -o json | jq '{
  InternalIP: (.status.addresses[] | select(.type=="InternalIP") | .address),
  InstanceID: (.spec.providerID | split("/") | last)
}'
# ProviderID format: aws:///eu-central-1a/i-XXXXXXXXX
```

## Cross-Validation Request Templates

Fill these with actual values from kubectl output. Include in `## Cross-Validation Requests`.

### Node issue → AWS

```
K8s → AWS: Node <NODE_NAME> (IP: <INTERNAL_IP>, instance: <i-XXXXXXXXX>)
status <NotReady|ContainerStatusUnknown> since <TIMESTAMP>.
Check: EC2 instance state + system/instance status checks, spot termination CloudTrail,
ASG scale-in activity, CloudWatch alarms on that instance.
Profile: <PROFILE>, Region: <REGION>
```

### OOMKilled → Prometheus

```
K8s → Prometheus: Pod <POD_NAME> in <NAMESPACE> OOMKilled at <TIMESTAMP>.
Memory limit: <LIMIT_MiB>. Restart count: <N>.
Confirm: container_memory_working_set_bytes spike approaching limit before <TIMESTAMP>.
Also check: kube_pod_container_resource_limits for that workload.
```

### CrashLoopBackOff → Loki

```
K8s → Loki: Pod <POD_NAME> in <NAMESPACE> restarted <N> times.
Last crash at ~<TIMESTAMP>. Container: <CONTAINER_NAME>.
Confirm: error log lines in that namespace/pod around that time.
Look for: panic, fatal, connection refused, timeout, or any exception stack trace.
```

### Pending (no capacity) → AWS

```
K8s → AWS: Pods Pending in <NAMESPACE> — reason: <REASON from describe events>.
Check: EKS nodegroup <NODEGROUP_NAME> desired vs actual count, health.issues,
ASG launch failures in last 1h, subnet AvailableIpAddressCount for nodegroup subnets.
Cluster: <CLUSTER_NAME>, Profile: <PROFILE>, Region: <REGION>
```

### 5XX errors → Prometheus + Loki

```
K8s → Prometheus: <N> pods of <WORKLOAD> in <NAMESPACE> have restartCount=<N> / failing.
Confirm: rate(http_requests_total{namespace="<NS>",status=~"5.."}) spike timing.
Identify: when error rate began, whether it correlates with pod restart events.

K8s → Loki: Service <SERVICE> in <NAMESPACE> pods crashing/erroring.
Confirm: error log patterns in {namespace="<NS>",app="<SERVICE>"} in last <TIMERANGE>.
Return: first occurrence time, top 5 error message patterns.
```

### ImagePullBackOff → AWS

```
K8s → AWS: Pod <POD_NAME> in <NAMESPACE> ImagePullBackOff.
Image: <FULL_IMAGE_URI> (ECR registry detected).
Check: ECR repo <REPO_NAME> exists in <REGION>; node IAM role allows ecr:GetAuthorizationToken
and ecr:BatchGetImage. Profile: <PROFILE>, Region: <REGION>
```
