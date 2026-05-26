---
name: aws-investigator
model: haiku
description: >
  Read-only AWS investigation using the AWS CLI. Use when the user wants to inspect
  EC2 instances, query CloudWatch Logs/Metrics, search CloudTrail events, check ELB
  target health, ECS task status, RDS instance health, or investigate any AWS
  infrastructure issue. Supports multiple named profiles and regions.
  Never modifies AWS resources.
  Trigger: /aws, "investigate aws", "check ec2", "cloudwatch logs", "cloudtrail",
  "who deleted", "elb health", "ecs task", "rds status", "aws issue", "check alarms",
  "vpc", "subnet", "ip exhaustion", "no ip addresses", "/28", "prefix delegation",
  "vpc cni", "eni limit", "nat gateway", "route table", "vpc issue",
  "asg", "auto scaling", "scale in", "scale out", "eks", "node group", "eks cluster",
  "cloudfront", "cdn", "distribution", "cache hit", "origin error", "cloudfront 5xx",
  "spot", "spot instance", "spot interrupted", "capacity-not-available", "spot price",
  "spot shortage", "insufficient capacity", "spot fleet", "spot termination".
trigger: /aws
---

# AWS Investigator

Read-only AWS diagnostics via AWS CLI. **Never** modify resources.

## Hard Rules

**NEVER run:**
- Any `aws ... create`, `delete`, `put`, `update`, `modify`, `terminate`, `stop`, `start`
- `aws iam` write operations
- `aws s3 rm`, `cp`, `mv`, `sync`
- Any command with `--dry-run` is OK; without it and modifying — skip

If a command is ambiguous, use `--dry-run` or skip it.

## Profile & Region Resolution

Resolve target in this order:

1. **Named config in message** — "prod", "staging" → look up in `~/.claude/aws-instances.json`
2. **Profile in message** — "use profile dev" → `--profile dev`
3. **Region in message** — "eu-west-1" → `--region eu-west-1`
4. **Config default** — `~/.claude/aws-instances.json` default entry
5. **AWS CLI default** — whatever `aws configure` has set

Config file (`~/.claude/aws-instances.json`):
```json
{
  "instances": {
    "prod":    { "profile": "prod-admin",  "region": "eu-west-1", "default": true },
    "staging": { "profile": "staging",     "region": "eu-west-1" },
    "us-prod": { "profile": "prod-us",     "region": "us-east-1" }
  }
}
```

Build CLI prefix from config:
```bash
aws --profile PROFILE --region REGION <subcommand>
# or just
AWS_PROFILE=PROFILE aws --region REGION <subcommand>
```

Always add `--output json` for structured output, pipe through `| jq .` for readability.

## Operations

### Verify CLI & Credentials

```bash
aws sts get-caller-identity [--profile P] [--region R]
aws configure list [--profile P]
```

### EC2 Instances

```bash
# List all instances with state
aws ec2 describe-instances [--profile P] [--region R] \
  --query 'Reservations[].Instances[] | [].{ID:InstanceId, Name:Tags[?Key==`Name`]|[0].Value, State:State.Name, Type:InstanceType, IP:PrivateIpAddress, AZ:Placement.AvailabilityZone}' \
  --output table

# Filter by state
aws ec2 describe-instances [--profile P] [--region R] \
  --filters Name=instance-state-name,Values=running \
  --query 'Reservations[].Instances[].{ID:InstanceId, Name:Tags[?Key==`Name`]|[0].Value, Type:InstanceType, IP:PrivateIpAddress}' \
  --output table

# Single instance detail
aws ec2 describe-instances [--profile P] [--region R] \
  --instance-ids i-XXXXXXXXX \
  --output json | jq '.Reservations[0].Instances[0] | {State, InstanceType, PrivateIpAddress, PublicIpAddress, LaunchTime, Tags, SecurityGroups, IamInstanceProfile}'

# Instance status (system/instance checks)
aws ec2 describe-instance-status [--profile P] [--region R] \
  --instance-ids i-XXXXXXXXX \
  --output json | jq '.InstanceStatuses[]'
```

### Security Groups

```bash
aws ec2 describe-security-groups [--profile P] [--region R] \
  --group-ids sg-XXXXXXXXX \
  --output json | jq '.SecurityGroups[] | {GroupName, Description, InboundRules: .IpPermissions, OutboundRules: .IpPermissionsEgress}'
```

### CloudWatch Alarms

```bash
# All alarms with state
aws cloudwatch describe-alarms [--profile P] [--region R] \
  --query 'MetricAlarms[].{Name:AlarmName, State:StateValue, Reason:StateReason, Metric:MetricName}' \
  --output table

# Only ALARM state
aws cloudwatch describe-alarms [--profile P] [--region R] \
  --state-value ALARM \
  --output json | jq '.MetricAlarms[] | {AlarmName, StateReason, MetricName, Namespace, Threshold}'
```

### CloudWatch Metrics

```bash
# List available metrics for a service
aws cloudwatch list-metrics [--profile P] [--region R] \
  --namespace AWS/EC2 \
  --dimensions Name=InstanceId,Value=i-XXXXXXXXX \
  --output json | jq '.Metrics[].MetricName'

# Get metric statistics (last 1 hour, 5-min granularity)
aws cloudwatch get-metric-statistics [--profile P] [--region R] \
  --namespace AWS/EC2 \
  --metric-name CPUUtilization \
  --dimensions Name=InstanceId,Value=i-XXXXXXXXX \
  --start-time $(date -u -v-1H +%Y-%m-%dT%H:%M:%SZ 2>/dev/null || date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 300 \
  --statistics Average Maximum \
  --output json | jq '.Datapoints | sort_by(.Timestamp) | .[] | {Time: .Timestamp, Avg: .Average, Max: .Maximum}'

# Common namespaces: AWS/EC2, AWS/ECS, AWS/RDS, AWS/ApplicationELB, AWS/Lambda, AWS/SQS
```

### CloudWatch Logs — Search

```bash
# List log groups
aws logs describe-log-groups [--profile P] [--region R] \
  --query 'logGroups[].{Name:logGroupName, RetentionDays:retentionInDays, SizeMB:storedBytes}' \
  --output table

# Filter by prefix
aws logs describe-log-groups [--profile P] [--region R] \
  --log-group-name-prefix /aws/lambda/

# Simple log search (last 1 hour)
aws logs filter-log-events [--profile P] [--region R] \
  --log-group-name /aws/lambda/my-function \
  --start-time $(( $(date +%s) * 1000 - 3600000 )) \
  --filter-pattern "ERROR" \
  --query 'events[].{Time:timestamp, Message:message}' \
  --output json | jq -r '.[] | (.Time | todate) + " " + .message'
```

### CloudWatch Logs Insights (powerful queries)

```bash
# Start query
QUERY_ID=$(aws logs start-query [--profile P] [--region R] \
  --log-group-name /aws/lambda/my-function \
  --start-time $(( $(date +%s) - 3600 )) \
  --end-time $(date +%s) \
  --query-string 'fields @timestamp, @message | filter @message like /ERROR/ | sort @timestamp desc | limit 20' \
  --output text --query queryId)

# Wait and get results
sleep 3
aws logs get-query-results [--profile P] [--region R] \
  --query-id "$QUERY_ID" \
  --output json | jq '.results[] | map({(.field): .value}) | add'
```

Common Logs Insights patterns:
```
# Error count by message
fields @message | filter @message like /ERROR/ | stats count(*) as errors by @message | sort errors desc | limit 20

# P99 latency (Lambda)
filter @type = "REPORT" | stats pct(@duration, 99) as p99, avg(@duration) as avg by bin(5m)

# Cold starts
filter @type = "REPORT" | filter @initDuration > 0 | stats count(*) as coldStarts by bin(5m)
```

### CloudTrail — Who Did What

```bash
# Recent events (last 1 hour) for a resource
aws cloudtrail lookup-events [--profile P] [--region R] \
  --lookup-attributes AttributeKey=ResourceName,AttributeValue=i-XXXXXXXXX \
  --start-time $(date -u -v-1H +%Y-%m-%dT%H:%M:%SZ 2>/dev/null || date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%SZ) \
  --output json | jq '.Events[] | {Time: .EventTime, Name: .EventName, User: .Username, Source: .EventSource}'

# Find who deleted/stopped something
aws cloudtrail lookup-events [--profile P] [--region R] \
  --lookup-attributes AttributeKey=EventName,AttributeValue=TerminateInstances \
  --output json | jq '.Events[] | {Time: .EventTime, User: .Username, Resources: .Resources}'

# All events by a specific user
aws cloudtrail lookup-events [--profile P] [--region R] \
  --lookup-attributes AttributeKey=Username,AttributeValue=USER_OR_ROLE \
  --output json | jq '.Events[] | {Time: .EventTime, Event: .EventName, Resource: .Resources[0].ResourceName}'

# Common event names to investigate:
# TerminateInstances, StopInstances, DeleteSecurityGroup, AuthorizeSecurityGroupIngress,
# DeleteBucket, PutBucketPolicy, CreateUser, AttachRolePolicy, DeleteStack
```

### ELB / ALB / NLB

`elbv2` covers both ALB (Application) and NLB (Network). Classic ELB uses `elb`.

```bash
# List all load balancers (ALB + NLB)
aws elbv2 describe-load-balancers [--profile P] [--region R] \
  --query 'LoadBalancers[].{Name:LoadBalancerName, DNS:DNSName, State:State.Code, Type:Type, Scheme:Scheme}' \
  --output table

# Single LB detail
aws elbv2 describe-load-balancers [--profile P] [--region R] \
  --names LB_NAME \
  --output json | jq '.LoadBalancers[] | {Name: .LoadBalancerName, Type: .Type, State: .State, DNS: .DNSName, AZs: [.AvailabilityZones[].ZoneName]}'

# List target groups (all or for specific LB)
aws elbv2 describe-target-groups [--profile P] [--region R] \
  --output json | jq '.TargetGroups[] | {Name: .TargetGroupName, ARN: .TargetGroupArn, Protocol: .Protocol, Port: .Port, HealthCheck: .HealthCheckPath}'

# Target health — works for ALB and NLB
aws elbv2 describe-target-health [--profile P] [--region R] \
  --target-group-arn ARN \
  --output json | jq '.TargetHealthDescriptions[] | {Target: .Target.Id, Port: .Target.Port, State: .TargetHealth.State, Reason: .TargetHealth.Reason, Description: .TargetHealth.Description}'

# NLB-specific: describe listener + rules
aws elbv2 describe-listeners [--profile P] [--region R] \
  --load-balancer-arn LB_ARN \
  --output json | jq '.Listeners[] | {Port: .Port, Protocol: .Protocol, DefaultActions: .DefaultActions}'

# ALB access logs location (if enabled)
aws elbv2 describe-load-balancer-attributes [--profile P] [--region R] \
  --load-balancer-arn LB_ARN \
  --output json | jq '[.Attributes[] | select(.Key | startswith("access_logs"))]'

# Classic ELB (legacy)
aws elb describe-load-balancers [--profile P] [--region R] \
  --query 'LoadBalancerDescriptions[].{Name:LoadBalancerName, DNS:DNSName, Instances:length(Instances)}' \
  --output table

aws elb describe-instance-health [--profile P] [--region R] \
  --load-balancer-name ELB_NAME \
  --output json | jq '.InstanceStates[] | {Instance: .InstanceId, State: .State, ReasonCode: .ReasonCode, Description: .Description}'

# CloudWatch metrics for ALB (HTTPCode_ELB_5XX_Count, TargetResponseTime, HealthyHostCount)
aws cloudwatch get-metric-statistics [--profile P] [--region R] \
  --namespace AWS/ApplicationELB \
  --metric-name HTTPCode_ELB_5XX_Count \
  --dimensions Name=LoadBalancer,Value=app/LB_NAME/HASH \
  --start-time $(date -u -v-1H +%Y-%m-%dT%H:%M:%SZ 2>/dev/null || date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 300 --statistics Sum \
  --output json | jq '[.Datapoints | sort_by(.Timestamp)[] | {Time: .Timestamp, Count: .Sum}]'

# NLB metrics (namespace: AWS/NetworkELB)
aws cloudwatch get-metric-statistics [--profile P] [--region R] \
  --namespace AWS/NetworkELB \
  --metric-name UnHealthyHostCount \
  --dimensions Name=LoadBalancer,Value=net/LB_NAME/HASH Name=TargetGroup,Value=targetgroup/TG_NAME/HASH \
  --start-time $(date -u -v-1H +%Y-%m-%dT%H:%M:%SZ 2>/dev/null || date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 300 --statistics Maximum \
  --output json | jq '[.Datapoints | sort_by(.Timestamp)[] | {Time: .Timestamp, Unhealthy: .Maximum}]'
```

Key LB issues:
- `TargetHealth.State != healthy` → check `Reason`: `Target.FailedHealthChecks`, `Target.NotRegistered`, `Target.DeregistrationInProgress`
- NLB unhealthy target with no `Reason` → security group blocking health check port (NLBs don't modify SGs)
- ALB `HTTPCode_ELB_5XX` → LB itself erroring (504=origin timeout, 502=bad response)
- `HealthyHostCount = 0` → all targets down — check ASG, ECS, or EC2 state

### ECS

```bash
# List clusters
aws ecs list-clusters [--profile P] [--region R] --output json | jq '.clusterArns[]'

# Services in cluster
aws ecs list-services [--profile P] [--region R] \
  --cluster CLUSTER_NAME \
  --output json | jq '.serviceArns[]'

# Service status (desired vs running)
aws ecs describe-services [--profile P] [--region R] \
  --cluster CLUSTER_NAME \
  --services SERVICE_NAME \
  --output json | jq '.services[] | {Name: .serviceName, Desired: .desiredCount, Running: .runningCount, Pending: .pendingCount, Status: .status, Events: .events[:5]}'

# Task details
aws ecs list-tasks [--profile P] [--region R] \
  --cluster CLUSTER_NAME --service-name SERVICE_NAME \
  --output json | jq '.taskArns[]'

aws ecs describe-tasks [--profile P] [--region R] \
  --cluster CLUSTER_NAME \
  --tasks TASK_ARN \
  --output json | jq '.tasks[] | {Status: .lastStatus, Health: .healthStatus, StopCode: .stopCode, StopReason: .stoppedReason, Containers: [.containers[] | {Name: .name, Status: .lastStatus, ExitCode: .exitCode, Reason: .reason}]}'
```

### RDS

```bash
# List DB instances
aws rds describe-db-instances [--profile P] [--region R] \
  --query 'DBInstances[].{ID:DBInstanceIdentifier, Class:DBInstanceClass, Engine:Engine, Status:DBInstanceStatus, Endpoint:Endpoint.Address}' \
  --output table

# Recent events (last 1 hour)
aws rds describe-events [--profile P] [--region R] \
  --duration 60 \
  --output json | jq '.Events[] | {Time: .Date, Source: .SourceIdentifier, Message: .Message}'

# Cluster status (Aurora)
aws rds describe-db-clusters [--profile P] [--region R] \
  --query 'DBClusters[].{ID:DBClusterIdentifier, Status:Status, Engine:Engine, Members:DBClusterMembers}' \
  --output json | jq .
```

### Lambda

```bash
# List functions
aws lambda list-functions [--profile P] [--region R] \
  --query 'Functions[].{Name:FunctionName, Runtime:Runtime, Memory:MemorySize, Timeout:Timeout, Modified:LastModified}' \
  --output table

# Function config + last error
aws lambda get-function [--profile P] [--region R] \
  --function-name FUNCTION_NAME \
  --output json | jq '{Config: .Configuration | {Runtime, MemorySize, Timeout, LastModified, State, StateReason}}'

# Recent throttles / errors (via CloudWatch)
aws cloudwatch get-metric-statistics [--profile P] [--region R] \
  --namespace AWS/Lambda \
  --metric-name Errors \
  --dimensions Name=FunctionName,Value=FUNCTION_NAME \
  --start-time $(date -u -v-1H +%Y-%m-%dT%H:%M:%SZ 2>/dev/null || date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 300 --statistics Sum \
  --output json | jq '[.Datapoints[] | {Time: .Timestamp, Errors: .Sum}] | sort_by(.Time)'
```

### VPC & Subnets (including EKS CNI /28 prefix issues)

```bash
# List VPCs
aws ec2 describe-vpcs [--profile P] [--region R] \
  --output json | jq '.Vpcs[] | {ID: .VpcId, CIDR: .CidrBlock, State: .State, Default: .IsDefault, Name: (.Tags[]? | select(.Key=="Name") | .Value)}'

# List subnets with available IPs (critical for EKS CNI exhaustion)
aws ec2 describe-subnets [--profile P] [--region R] \
  --output json | jq '.Subnets[] | {
    ID: .SubnetId,
    Name: (.Tags[]? | select(.Key=="Name") | .Value),
    AZ: .AvailabilityZone,
    CIDR: .CidrBlock,
    AvailableIPs: .AvailableIpAddressCount,
    TotalIPs: ((.CidrBlock | split("/")[1] | tonumber) as $prefix | pow(2; 32 - $prefix) | floor - 5),
    State: .State
  }' | jq -s 'sort_by(.AvailableIPs)'

# Filter subnets by VPC
aws ec2 describe-subnets [--profile P] [--region R] \
  --filters Name=vpc-id,Values=vpc-XXXXXXXXX \
  --output json | jq '.Subnets[] | {ID: .SubnetId, AZ: .AvailabilityZone, CIDR: .CidrBlock, AvailableIPs: .AvailableIpAddressCount}'

# Check /28 prefix delegations in subnet (EKS VPC CNI prefix mode)
# Prefix delegation carves /28 blocks (16 IPs each) from the subnet
aws ec2 describe-subnets [--profile P] [--region R] \
  --subnet-ids subnet-XXXXXXXXX \
  --output json | jq '.Subnets[] | {
    CIDR: .CidrBlock,
    AvailableIPs: .AvailableIpAddressCount,
    PrefixesOf28Possible: (.AvailableIpAddressCount / 16 | floor),
    Comment: (if .AvailableIpAddressCount < 16 then "CRITICAL: cannot allocate /28 prefix" elif .AvailableIpAddressCount < 64 then "WARNING: fewer than 4 /28 prefixes available" else "OK" end)
  }'

# List all ENIs in subnet (to find what's consuming IPs)
aws ec2 describe-network-interfaces [--profile P] [--region R] \
  --filters Name=subnet-id,Values=subnet-XXXXXXXXX \
  --output json | jq '[.NetworkInterfaces[] | {ID: .NetworkInterfaceId, Description: .Description, Status: .Status, PrivateIP: .PrivateIpAddress, Attachment: .Attachment.InstanceId}] | length as $total | {TotalENIs: $total, ENIs: .}'

# Find ENIs with prefix delegations (/28 blocks assigned to them)
aws ec2 describe-network-interfaces [--profile P] [--region R] \
  --filters Name=subnet-id,Values=subnet-XXXXXXXXX \
  --output json | jq '[.NetworkInterfaces[] | select(.Ipv4Prefixes | length > 0) | {ID: .NetworkInterfaceId, Instance: .Attachment.InstanceId, Prefixes: [.Ipv4Prefixes[].Ipv4Prefix]}]'

# Check VPC CNI prefix delegation config on EKS nodegroup
# (reads DaemonSet env var — use kubectl, not AWS CLI)
# kubectl get ds aws-node -n kube-system -o json | jq '.spec.template.spec.containers[0].env[] | select(.name == "ENABLE_PREFIX_DELEGATION")'

# ENI/IP limits per instance type (important for VPC CNI capacity planning)
aws ec2 describe-instance-types [--profile P] [--region R] \
  --instance-types m5.large m5.xlarge m5.2xlarge \
  --output json | jq '.InstanceTypes[] | {
    Type: .InstanceType,
    MaxENIs: .NetworkInfo.MaximumNetworkInterfaces,
    MaxIPsPerENI: .NetworkInfo.Ipv4AddressesPerInterface,
    MaxPrefixesPerENI: .NetworkInfo.MaximumNetworkCards,
    TotalIPsWithoutPrefix: (.NetworkInfo.MaximumNetworkInterfaces * .NetworkInfo.Ipv4AddressesPerInterface),
    TotalIPsWithPrefix: (.NetworkInfo.MaximumNetworkInterfaces * .NetworkInfo.Ipv4AddressesPerInterface * 16)
  }'

# VPC CIDR associations (secondary CIDRs)
aws ec2 describe-vpcs [--profile P] [--region R] \
  --vpc-ids vpc-XXXXXXXXX \
  --output json | jq '.Vpcs[] | {CidrBlock: .CidrBlock, CidrAssociations: [.CidrBlockAssociationSet[] | {CIDR: .CidrBlock, State: .CidrBlockState.State}]}'

# VPC flow logs (are they enabled? where do they go?)
aws ec2 describe-flow-logs [--profile P] [--region R] \
  --filter Name=resource-id,Values=vpc-XXXXXXXXX \
  --output json | jq '.FlowLogs[] | {ID: .FlowLogId, Status: .FlowLogStatus, Destination: .LogDestination, TrafficType: .TrafficType}'

# NAT Gateways state
aws ec2 describe-nat-gateways [--profile P] [--region R] \
  --output json | jq '.NatGateways[] | {ID: .NatGatewayId, VPC: .VpcId, Subnet: .SubnetId, State: .State, Type: .ConnectivityType}'

# Route tables — check default route (0.0.0.0/0) exists
aws ec2 describe-route-tables [--profile P] [--region R] \
  --filters Name=vpc-id,Values=vpc-XXXXXXXXX \
  --output json | jq '.RouteTables[] | {ID: .RouteTableId, Associations: [.Associations[].SubnetId], DefaultRoute: (.Routes[] | select(.DestinationCidrBlock == "0.0.0.0/0") | {Via: (.GatewayId // .NatGatewayId // .TransitGatewayId), State: .State})}'

# Internet Gateway
aws ec2 describe-internet-gateways [--profile P] [--region R] \
  --filters Name=attachment.vpc-id,Values=vpc-XXXXXXXXX \
  --output json | jq '.InternetGateways[] | {ID: .InternetGatewayId, State: .Attachments[0].State}'
```

**EKS VPC CNI /28 prefix investigation checklist:**

```bash
# Step 1: check which subnets EKS nodes use
aws eks describe-nodegroup [--profile P] [--region R] \
  --cluster-name CLUSTER --nodegroup-name NODEGROUP \
  --output json | jq '.nodegroup.subnets'

# Step 2: for each subnet — check available IPs and /28 capacity
aws ec2 describe-subnets [--profile P] [--region R] \
  --subnet-ids subnet-AAA subnet-BBB \
  --output json | jq '.Subnets[] | {
    ID: .SubnetId, AZ: .AvailabilityZone,
    AvailableIPs: .AvailableIpAddressCount,
    CanAllocate28: (.AvailableIpAddressCount >= 16),
    Prefixes28Available: (.AvailableIpAddressCount / 16 | floor)
  }'

# Step 3: check if prefix delegation is enabled on VPC CNI
# (requires kubectl — mention to user if AWS CLI insufficient)

# Step 4: check ENIs already holding prefixes in those subnets
aws ec2 describe-network-interfaces [--profile P] [--region R] \
  --filters Name=subnet-id,Values=subnet-AAA \
  --output json | jq '[.NetworkInterfaces[] | select(.Ipv4Prefixes | length > 0)] | length as $count | "ENIs with /28 prefixes: \($count)"'

# Step 5: check if subnet is in RFC1918 range that supports /28
# /28 requires at least 16 contiguous free IPs — fragmented subnets may fail
# even with AvailableIpAddressCount > 16
```

Key /28 prefix issues:
- `AvailableIpAddressCount < 16` → subnet exhausted, **cannot allocate /28**, pods won't get IPs
- `AvailableIpAddressCount >= 16` but still failing → fragmentation (free IPs not contiguous) or AWS internal reservation
- Each /28 = 16 IPs, supports 16 pods per prefix; each ENI can hold multiple prefixes
- Fix options: add secondary CIDR to VPC, expand subnet, add new subnet to nodegroup
- Check `ENABLE_PREFIX_DELEGATION=true` in `aws-node` DaemonSet via `kubectl`



```bash
# List active spot instance requests
aws ec2 describe-spot-instance-requests [--profile P] [--region R] \
  --query 'SpotInstanceRequests[].{ID:SpotInstanceRequestId, State:State, Status:Status.Code, Message:Status.Message, Type:Type, AZ:LaunchedAvailabilityZone, InstanceID:InstanceId, Price:SpotPrice}' \
  --output table

# Filter failed/interrupted requests
aws ec2 describe-spot-instance-requests [--profile P] [--region R] \
  --filters Name=state,Values=failed,cancelled,closed \
  --output json | jq '.SpotInstanceRequests[] | {ID: .SpotInstanceRequestId, State: .State, Code: .Status.Code, Message: .Status.Message, AZ: .LaunchedAvailabilityZone, Time: .Status.UpdateTime}'

# Status codes that indicate problems:
# capacity-not-available      → no capacity in this AZ/instance type
# price-too-low               → bid below current spot price
# instance-terminated-by-spot → AWS reclaimed (interruption)
# instance-stopped-by-spot    → stopped due to interruption
# constraint-not-fulfillable  → AZ or instance type constraints too narrow

# Spot price history (last 6 hours, compare AZs)
aws ec2 describe-spot-price-history [--profile P] [--region R] \
  --instance-types m5.large m5.xlarge \
  --product-descriptions "Linux/UNIX" \
  --start-time $(date -u -v-6H +%Y-%m-%dT%H:%M:%SZ 2>/dev/null || date -u -d '6 hours ago' +%Y-%m-%dT%H:%M:%SZ) \
  --output json | jq '[.SpotPriceHistory[] | {AZ: .AvailabilityZone, Type: .InstanceType, Price: .SpotPrice, Time: .Timestamp}] | sort_by(.Time) | reverse | .[:20]'

# Spot fleet requests
aws ec2 describe-spot-fleet-requests [--profile P] [--region R] \
  --output json | jq '.SpotFleetRequestConfigs[] | {ID: .SpotFleetRequestId, State: .SpotFleetRequestState, Target: .SpotFleetRequestConfig.TargetCapacity, Fulfilled: .SpotFleetRequestConfig.FulfilledCapacity, ExcessCapacity: .SpotFleetRequestConfig.ExcessCapacityTerminationPolicy}'

# Spot fleet instances + their health
aws ec2 describe-spot-fleet-instances [--profile P] [--region R] \
  --spot-fleet-request-id sfr-XXXXXXXX \
  --output json | jq '.ActiveInstances[] | {ID: .InstanceId, Type: .InstanceType, AZ: .InstanceHealth}'

# Spot interruption rebalance recommendations (EC2 signals before interruption)
aws ec2 describe-instances [--profile P] [--region R] \
  --filters Name=instance-lifecycle,Values=spot Name=instance-state-name,Values=running \
  --output json | jq '.Reservations[].Instances[] | {ID: .InstanceId, Type: .InstanceType, AZ: .Placement.AvailabilityZone, RebalanceRecommendation: .SpotInstanceDetails}'

# CloudTrail: spot interruptions and capacity errors
aws cloudtrail lookup-events [--profile P] [--region R] \
  --lookup-attributes AttributeKey=EventName,AttributeValue=BidEvictedEvent \
  --output json | jq '.Events[] | {Time: .EventTime, Resources: .Resources}'

# InsufficientInstanceCapacity in CloudTrail (spot OR on-demand capacity shortage)
aws cloudtrail lookup-events [--profile P] [--region R] \
  --lookup-attributes AttributeKey=EventName,AttributeValue=RunInstances \
  --output json | jq '[.Events[] | select(.CloudTrailEvent | fromjson | .errorCode == "InsufficientInstanceCapacity")] | .[] | {Time: .EventTime, Error: (.CloudTrailEvent | fromjson | .errorMessage)}'

# ASG mixed instances policy (spot + on-demand split)
aws autoscaling describe-auto-scaling-groups [--profile P] [--region R] \
  --auto-scaling-group-names ASG_NAME \
  --output json | jq '.AutoScalingGroups[] | {
    MixedPolicy: .MixedInstancesPolicy | {
      OnDemandBase: .InstancesDistribution.OnDemandBaseCapacity,
      OnDemandPct: .InstancesDistribution.OnDemandPercentageAboveBaseCapacity,
      SpotAllocation: .InstancesDistribution.SpotAllocationStrategy,
      SpotMaxPrice: .InstancesDistribution.SpotMaxPrice,
      InstanceOverrides: [.LaunchTemplate.Overrides[] | .InstanceType]
    },
    Instances: [.Instances[] | {ID: .InstanceId, Type: .InstanceType, Market: .Market, AZ: .AvailabilityZone, Health: .HealthStatus}]
  }'
```

Key Spot issues:
- `capacity-not-available` → switch AZ or instance type; check price history across AZs
- `price-too-low` → bid too low; use `on-demand` or check current spot price
- `instance-terminated-by-spot` → normal interruption; ensure workload is interruption-tolerant
- High spot price in all AZs → capacity crunch in region; consider different instance family
- ASG `SpotAllocationStrategy` — prefer `capacity-optimized` over `lowest-price` for availability
- `SpotMaxPrice` set → can cause `price-too-low`; remove cap to use on-demand price ceiling



```bash
# List ASGs
aws autoscaling describe-auto-scaling-groups [--profile P] [--region R] \
  --query 'AutoScalingGroups[].{Name:AutoScalingGroupName, Min:MinSize, Max:MaxSize, Desired:DesiredCapacity, Running:length(Instances[?LifecycleState==`InService`])}' \
  --output table

# Single ASG detail (instances + health)
aws autoscaling describe-auto-scaling-groups [--profile P] [--region R] \
  --auto-scaling-group-names ASG_NAME \
  --output json | jq '.AutoScalingGroups[] | {
    Name: .AutoScalingGroupName,
    Desired: .DesiredCapacity, Min: .MinSize, Max: .MaxSize,
    Instances: [.Instances[] | {ID: .InstanceId, AZ: .AvailabilityZone, State: .LifecycleState, Health: .HealthStatus}],
    SuspendedProcesses: [.SuspendedProcesses[].ProcessName]
  }'

# Recent scaling activity (last 20 events)
aws autoscaling describe-scaling-activities [--profile P] [--region R] \
  --auto-scaling-group-name ASG_NAME \
  --max-items 20 \
  --output json | jq '.Activities[] | {Time: .StartTime, Status: .StatusCode, Description: .Description, Cause: .Cause}'

# Instances failing health checks
aws autoscaling describe-auto-scaling-instances [--profile P] [--region R] \
  --output json | jq '[.AutoScalingInstances[] | select(.HealthStatus != "HEALTHY")] | .[] | {ID: .InstanceId, ASG: .AutoScalingGroupName, State: .LifecycleState, Health: .HealthStatus}'

# Scheduled scaling actions
aws autoscaling describe-scheduled-actions [--profile P] [--region R] \
  --auto-scaling-group-name ASG_NAME \
  --output table
```

Key ASG issues to check:
- `SuspendedProcesses` — Launch/Terminate suspended → ASG won't scale
- `HealthStatus != HEALTHY` → instances being replaced
- Scaling activity `WaitingForELBConnectionDraining` → slow scale-in
- Activity `StatusCode: Failed` → launch failure (bad AMI, quota, insufficient capacity)

### EKS

```bash
# List clusters
aws eks list-clusters [--profile P] [--region R] --output json | jq '.clusters[]'

# Cluster status + version
aws eks describe-cluster [--profile P] [--region R] \
  --name CLUSTER_NAME \
  --output json | jq '.cluster | {Name: .name, Status: .status, Version: .version, Endpoint: .endpoint, Health: .health, PlatformVersion: .platformVersion}'

# List node groups
aws eks list-nodegroups [--profile P] [--region R] \
  --cluster-name CLUSTER_NAME \
  --output json | jq '.nodegroups[]'

# Node group status + scaling
aws eks describe-nodegroup [--profile P] [--region R] \
  --cluster-name CLUSTER_NAME \
  --nodegroup-name NODEGROUP_NAME \
  --output json | jq '.nodegroup | {
    Name: .nodegroupName, Status: .status,
    Desired: .scalingConfig.desiredSize, Min: .scalingConfig.minSize, Max: .scalingConfig.maxSize,
    InstanceType: .instanceTypes,
    AmiType: .amiType, Version: .version, ReleaseVersion: .releaseVersion,
    Health: .health
  }'

# Node group health issues
aws eks describe-nodegroup [--profile P] [--region R] \
  --cluster-name CLUSTER_NAME \
  --nodegroup-name NODEGROUP_NAME \
  --output json | jq '.nodegroup.health.issues'

# List Fargate profiles
aws eks list-fargate-profiles [--profile P] [--region R] \
  --cluster-name CLUSTER_NAME \
  --output json | jq '.fargateProfileNames[]'

aws eks describe-fargate-profile [--profile P] [--region R] \
  --cluster-name CLUSTER_NAME \
  --fargate-profile-name PROFILE_NAME \
  --output json | jq '.fargateProfile | {Status: .status, Selectors: .selectors, Health: .health}'

# Add-ons status
aws eks list-addons [--profile P] [--region R] \
  --cluster-name CLUSTER_NAME \
  --output json | jq '.addons[]'

aws eks describe-addon [--profile P] [--region R] \
  --cluster-name CLUSTER_NAME \
  --addon-name ADDON_NAME \
  --output json | jq '.addon | {Name: .addonName, Status: .status, Version: .addonVersion, Health: .health}'

# CloudWatch metrics for EKS node groups
aws cloudwatch get-metric-statistics [--profile P] [--region R] \
  --namespace AWS/EKS \
  --metric-name node_cpu_utilization \
  --dimensions Name=ClusterName,Value=CLUSTER_NAME \
  --start-time $(date -u -v-1H +%Y-%m-%dT%H:%M:%SZ 2>/dev/null || date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 300 --statistics Average \
  --output json | jq '[.Datapoints | sort_by(.Timestamp)[] | {Time: .Timestamp, CPU: .Average}]'
```

Key EKS issues to check:
- Cluster `status != ACTIVE` → upgrade or degraded
- Nodegroup `status != ACTIVE` → `health.issues` has the reason (e.g. `Ec2LaunchTemplateNotFound`, `NodeCreationFailure`)
- Add-on `status != ACTIVE` → version conflict or IAM issue
- Node group `desiredSize > maxSize` → can't scale (quota or config)

### CloudFront

```bash
# List distributions
aws cloudfront list-distributions [--profile P] \
  --output json | jq '.DistributionList.Items[] | {
    ID: .Id, Domain: .DomainName, Status: .Status, Enabled: .Enabled,
    Origins: [.Origins.Items[].DomainName],
    Aliases: (.Aliases.Items // [])
  }'

# Single distribution detail
aws cloudfront get-distribution [--profile P] \
  --id DISTRIBUTION_ID \
  --output json | jq '.Distribution | {
    Status: .Status,
    DomainName: .DomainName,
    Config: .DistributionConfig | {
      Enabled: .Enabled,
      PriceClass: .PriceClass,
      HttpVersion: .HttpVersion,
      DefaultCacheBehavior: .DefaultCacheBehavior | {ViewerProtocol: .ViewerProtocolPolicy, CachePolicy: .CachePolicyId, OriginRequestPolicy: .OriginRequestPolicyId},
      Origins: [.Origins.Items[] | {ID: .Id, Domain: .DomainName, Protocol: .CustomOriginConfig.OriginProtocolPolicy}]
    }
  }'

# CloudFront metrics (note: must use us-east-1 for CloudFront metrics)
aws cloudwatch get-metric-statistics --region us-east-1 [--profile P] \
  --namespace AWS/CloudFront \
  --metric-name 5xxErrorRate \
  --dimensions Name=DistributionId,Value=DIST_ID Name=Region,Value=Global \
  --start-time $(date -u -v-1H +%Y-%m-%dT%H:%M:%SZ 2>/dev/null || date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 300 --statistics Average \
  --output json | jq '[.Datapoints | sort_by(.Timestamp)[] | {Time: .Timestamp, ErrorRate: .Average}]'

# All CloudFront metrics: Requests, BytesDownloaded, BytesUploaded,
# 4xxErrorRate, 5xxErrorRate, TotalErrorRate, CacheHitRate, OriginLatency

# Cache hit rate
aws cloudwatch get-metric-statistics --region us-east-1 [--profile P] \
  --namespace AWS/CloudFront \
  --metric-name CacheHitRate \
  --dimensions Name=DistributionId,Value=DIST_ID Name=Region,Value=Global \
  --start-time $(date -u -v-1H +%Y-%m-%dT%H:%M:%SZ 2>/dev/null || date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 300 --statistics Average \
  --output json | jq '[.Datapoints | sort_by(.Timestamp)[] | {Time: .Timestamp, HitRate: .Average}]'

# Invalidation status (recent)
aws cloudfront list-invalidations [--profile P] \
  --distribution-id DISTRIBUTION_ID \
  --output json | jq '.InvalidationList.Items[:5] | .[] | {ID: .Id, Status: .Status, Created: .CreateTime}'

# CloudTrail: recent distribution changes (use us-east-1 — CF events go there)
aws cloudtrail lookup-events --region us-east-1 [--profile P] \
  --lookup-attributes AttributeKey=ResourceName,AttributeValue=DISTRIBUTION_ID \
  --output json | jq '.Events[] | {Time: .EventTime, Event: .EventName, User: .Username}'
```

Key CloudFront issues:
- `5xxErrorRate` spike → origin returning errors; check origin (EC2/ALB/S3) health
- `CacheHitRate` drop → cache-busting headers, TTL=0, or query string variation
- `Status: InProgress` → distribution updating (takes 5-15 min)
- Invalidation `InProgress` → cache clearing in flight
- **Note:** CloudFront metrics only available in `us-east-1` regardless of distribution region

## Output Formatting

- Tables for lists (use `--output table` or `jq` table-like format)
- JSON detail for single resource deep-dives
- CloudTrail: show time, actor, action, resource — sorted by time
- Alarms: ALARM state first, then INSUFFICIENT_DATA, then OK
- ECS: highlight Desired≠Running mismatches
- ASG: highlight SuspendedProcesses, Failed activities, unhealthy instances
- EKS: highlight non-ACTIVE nodegroups with health.issues details
- CloudFront: always use `us-east-1` for metrics; note deployment takes 5-15 min
- NLB: check SG rules when targets show unhealthy with no reason
- Spot: compare price history across AZs side-by-side to identify cheapest AZ
- Spot: `capacity-not-available` → widen instance type pool, don't just retry same type
- VPC CNI: always sort subnets by `AvailableIPs` ascending — exhausted ones surface immediately
- VPC CNI /28: `AvailableIPs >= 16` necessary but not sufficient — fragmentation can still block allocation
- If command fails with error → quote exact error, suggest fix (wrong region, missing permission, resource not found)

## Config Management (`/aws config`)

Config file: `~/.claude/aws-instances.json`
Use `Read` + `Write` tools only — no shell commands.

### `/aws config add`

Ask for (only what's missing):
- **name** — alias: `prod`, `staging`
- **profile** — AWS CLI profile name (from `~/.aws/credentials`)
- **region** — e.g. `eu-west-1`
- **default** (optional)

### `/aws config list`

| Name | Profile | Region | Default |
|------|---------|--------|---------|
| prod | prod-admin | eu-west-1 | ✓ |

### `/aws config test [name]`

```bash
aws sts get-caller-identity --profile PROFILE --region REGION
```

## Error Reference

| Error | Cause | Fix |
|---|---|---|
| `Unable to locate credentials` | No profile configured | Run `aws configure --profile NAME` |
| `InvalidClientTokenId` | Wrong region or expired token | Check `--region`, refresh SSO: `aws sso login` |
| `AccessDenied` | IAM permission missing | Check IAM policy for the action |
| `UnauthorizedOperation` | Same as AccessDenied (EC2) | Check IAM |
| `ResourceNotFoundException` | Wrong ID or wrong region | Verify resource ID and region |
| `ExpiredTokenException` | SSO/assumed role expired | `aws sso login --profile NAME` |
| `could not connect to endpoint` | Wrong region or VPC endpoint issue | Check `--region` |
