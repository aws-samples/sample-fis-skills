---
name: app-service-log-analysis
description: >
  Use when the user wants to analyze application and managed service logs during or after a FIS experiment.
  Triggers on "analyze app logs", "application log analysis", "check application behavior",
  "分析应用日志", "查看应用表现", "应用日志分析". Supports two modes: real-time monitoring
  (during experiment) and post-hoc analysis (after experiment). Reads experiment context
  from aws-fis-experiment-prepare/execute outputs.
---

# App & Service Log Analysis

Analyze application and managed service logs during FIS fault injection experiments to understand how
applications respond to infrastructure failures. Supports real-time monitoring and
post-hoc analysis modes.

## Output Language Rule

Detect the language of the user's conversation and use the **same language** for all output.
- Chinese input -> Chinese output
- English input -> English output

## Prerequisites

Required tools:
- **kubectl** — configured with access to target EKS cluster(s)
- **AWS CLI** — for querying FIS experiment status and EKS cluster discovery
- A prepared/executed FIS experiment directory (from aws-fis-experiment-prepare or aws-fis-experiment-execute)

## Multi-Cluster EKS Discovery and Kubeconfig Isolation

When the environment contains multiple EKS clusters, the skill discovers ALL clusters
in the target region and scans each one for applications depending on the affected
services. Each cluster gets its own kubeconfig file to avoid overwriting the user's
existing `~/.kube/config`.

### Kubeconfig Isolation

**CRITICAL: Never overwrite `~/.kube/config`.** Generate a dedicated kubeconfig file
per cluster in the log directory:

```bash
KUBECONFIG_DIR="${LOG_DIR}/kubeconfigs"
mkdir -p "${KUBECONFIG_DIR}"

# For each EKS cluster, generate an isolated kubeconfig file
aws eks update-kubeconfig \
  --name "{CLUSTER_NAME}" \
  --region ${TARGET_REGION} \
  --kubeconfig "${KUBECONFIG_DIR}/${CLUSTER_NAME}.kubeconfig"
```

All subsequent `kubectl` commands for that cluster MUST use the `--kubeconfig` flag:
```bash
kubectl --kubeconfig "${KUBECONFIG_DIR}/${CLUSTER_NAME}.kubeconfig" get pods -A
```

Or set `KUBECONFIG` env var per-command (preferred for background log processes):
```bash
KUBECONFIG="${KUBECONFIG_DIR}/${CLUSTER_NAME}.kubeconfig" kubectl logs -f ...
```

### Cluster Discovery

1. **List all EKS clusters** in the target region:
   ```bash
   aws eks list-clusters --region ${TARGET_REGION} --query 'clusters[]' --output json
   ```

2. **Filter relevant clusters** — If the experiment already targets a specific EKS
   cluster (e.g., from the experiment README), start with that cluster. Then check
   remaining clusters in the region, as applications on other clusters may also
   depend on the affected service (e.g., a Redis cluster shared by apps across
   multiple EKS clusters).

3. **Generate isolated kubeconfig** for each cluster (see above).

4. **Verify access** before scanning — if `kubectl get nodes` fails for a cluster,
   skip it and inform the user:
   ```
   Cluster discovery:
     ✅ eks-cluster-prod: accessible (12 nodes)
     ✅ eks-cluster-staging: accessible (4 nodes)
     ❌ eks-cluster-private: not accessible (check VPC/IAM config)
   ```

5. **Parallel scanning** — scan all accessible clusters concurrently (one agent per
   cluster) for dependency discovery in Step 3a.

## Workflow

```dot
digraph log_analysis_flow {
    "Receive input path" [shape=box];
    "Detect mode" [shape=diamond];
    "Real-time mode" [shape=box];
    "Post-hoc mode" [shape=box];
    "Read service list" [shape=box];
    "Auto-discover + confirm app dependencies" [shape=box];
    "Detect + collect managed service logs" [shape=box];
    "Start background log collection" [shape=box];
    "Batch fetch historical logs" [shape=box];
    "Frontend polling + insight display" [shape=box];
    "Experiment complete?" [shape=diamond];
    "Generate analysis report" [shape=box];

    "Receive input path" -> "Detect mode";
    "Detect mode" -> "Real-time mode" [label="directory with README"];
    "Detect mode" -> "Post-hoc mode" [label="*-experiment-results.md"];
    "Real-time mode" -> "Read service list";
    "Post-hoc mode" -> "Read service list";
    "Read service list" -> "Detect + collect managed service logs";
    "Detect + collect managed service logs" -> "Auto-discover + confirm app dependencies";
    "Auto-discover + confirm app dependencies" -> "Start background log collection" [label="real-time"];
    "Auto-discover + confirm app dependencies" -> "Batch fetch historical logs" [label="post-hoc"];
    "Start background log collection" -> "Frontend polling + insight display";
    "Frontend polling + insight display" -> "Experiment complete?";
    "Experiment complete?" -> "Frontend polling + insight display" [label="No, continue"];
    "Experiment complete?" -> "Generate analysis report" [label="Yes"];
    "Batch fetch historical logs" -> "Generate analysis report";
}
```

### Step 1: Detect Mode and Load Context

The user provides either:
- **Directory path** (e.g., `./2026-03-31-14-30-22-az-power-interruption-my-cluster/`) → Real-time mode
- **Report file path** (e.g., `./2026-03-31-...-experiment-results.md`) → Post-hoc mode

**Real-time mode:** The directory contains a `README.md` from the prepare skill.
Extract the experiment template ID and region from it.

**Post-hoc mode:** The file is an experiment results report (contains "FIS Experiment
Results"). Extract experiment ID, start time, end time, and region from it.

### Step 2: Read Service List

Extract affected AWS services from:
- `expected-behavior.md` in the experiment directory (real-time mode), or
- the experiment results report (post-hoc mode)

Look for service name headings (e.g., "### RDS (cluster-xxx)") to build the list.
Present the detected service list to the user.


### Step 2.5: Detect and Collect Managed Service Logs

For each affected AWS service identified in Step 2, check whether it has CloudWatch
logging enabled. If enabled, record the log group names for later collection in Step 7.
If not enabled, skip and note in the final report as a recommendation.

**Time window note:** When called from `aws-fis-experiment-execute`, the end time
includes a 3-minute post-experiment baseline window. Use `EXPERIMENT_END_TIME + 3 minutes`
as the query end time to capture recovery behavior in managed service logs.

**Supported managed services:** EKS Control Plane, RDS/Aurora, ElastiCache, MSK, OpenSearch.
See `references/managed-service-log-commands.md` for check commands and log group formats.

**Workflow:**

1. For each service in the affected service list, extract the resource identifier from
   the experiment template or README (cluster name, cluster ID, replication group ID, etc.)
2. Run the check command. If logging is not enabled or the service is not present
   in the experiment, skip it
3. For enabled services, record the log group name(s) in `MANAGED_LOG_GROUPS` map
   (service → list of log group names) for later use in Step 7
4. Present detection results to the user:
   ```
   Managed service log detection:
     ✅ EKS Control Plane: enabled (api, audit, scheduler) → /aws/eks/{cluster}/cluster
     ✅ RDS Aurora: enabled (error, slowquery) → /aws/rds/cluster/{id}/error, .../slowquery
     ❌ ElastiCache: logging not enabled (recommend enabling slow-log, engine-log)
     ⬚ MSK: not involved in this experiment
   ```

**If logging is not enabled for a service**, record in `MANAGED_LOG_RECOMMENDATIONS`
for the report's Recommendations section:
```
**{Service}:** CloudWatch logging is not enabled. Enable {log-types} for better
fault injection analysis. Without these logs, only application-side impact is visible.
```


### Step 3: Collect Application Dependencies

#### 3a. Auto-Discover Potential Dependencies (Deep Scan)

For each affected AWS service, automatically discover EKS applications that may
depend on it. **Scan ALL accessible EKS clusters** (from Multi-Cluster Discovery
above) in parallel.

**Step 3a-1: Resolve service endpoints**

Get the service's endpoint(s) and identifiers for matching:

| Service | CLI Command | Match Targets |
|---|---|---|
| RDS/Aurora | `aws rds describe-db-clusters --db-cluster-identifier {ID} --query 'DBClusters[0].{Endpoint:Endpoint,ReaderEndpoint:ReaderEndpoint,Port:Port}'` | Endpoint hostname, reader endpoint, port |
| ElastiCache (Redis) | `aws elasticache describe-replication-groups --replication-group-id {ID} --query 'ReplicationGroups[0].{Primary:NodeGroups[0].PrimaryEndpoint,Reader:NodeGroups[0].ReaderEndpoint}'` | Primary endpoint, reader endpoint, port (6379) |
| MSK (Kafka) | `aws kafka get-bootstrap-brokers --cluster-arn {ARN}` | Bootstrap broker endpoints, port (9092/9094) |
| OpenSearch | `aws opensearch describe-domain --domain-name {DOMAIN} --query 'DomainStatus.Endpoints'` | Domain endpoint, VPC endpoint |
| EC2 instance | `aws ec2 describe-instances --instance-ids {ID} --query 'Reservations[0].Instances[0].{PrivateIp:PrivateIpAddress,PrivateDns:PrivateDnsName}'` | Private IP, private DNS |

Build a `SERVICE_ENDPOINTS` map (service → list of endpoint strings to search for).
Include both the full hostname and partial matches (e.g., cluster identifier without
domain suffix) to catch applications that construct endpoints dynamically.

**Step 3a-2: Deep scan across all clusters**

For each accessible EKS cluster, search the following sources (ordered by reliability):

| Priority | Source | Command | What It Catches |
|---|---|---|---|
| 1 | **Pod environment variables** | `kubectl get pods -A -o json \| jq '.items[].spec.containers[].env[]?'` | Direct endpoint references in env vars (e.g., `DB_HOST`, `REDIS_URL`, `KAFKA_BROKERS`) |
| 2 | **ConfigMaps** | `kubectl get configmaps -A -o json \| jq '.items[].data'` | Endpoint references in configuration files (application.yml, .env, etc.) |
| 3 | **Secrets** (metadata only) | `kubectl get secrets -A -o json \| jq '.items[] \| {name: .metadata.name, namespace: .metadata.namespace, keys: (.data \| keys)}'` | Secret key names hinting at service connections (e.g., `db-password`, `redis-auth`, `kafka-credentials`). Do NOT decode secret values — only match key names. |
| 4 | **EnvFrom references** | `kubectl get pods -A -o json \| jq '.items[].spec.containers[].envFrom[]?'` | Pods referencing ConfigMaps/Secrets that contain endpoints (follow the reference to check contents) |
| 5 | **Service ExternalName** | `kubectl get services -A -o json \| jq '.items[] \| select(.spec.type=="ExternalName") \| {name: .metadata.name, ns: .metadata.namespace, externalName: .spec.externalName}'` | K8s Services that point to external AWS endpoints (e.g., `mydb.cluster-xxx.region.rds.amazonaws.com`) |
| 6 | **Pod volume mounts** (projected/CSI) | `kubectl get pods -A -o json \| jq '.items[].spec.volumes[]? \| select(.projected or .csi)'` | IRSA-based or CSI-based service connections (e.g., Secrets Store CSI for RDS credentials) |

**Matching logic:**
- For each source, search for ANY string from `SERVICE_ENDPOINTS` (case-insensitive)
- Also match common service identifier patterns:
  - RDS: preferred cluster identifier
  - ElastiCache: preferred cluster name
  - MSK: preferred broker endpoints
  - OpenSearch: preferred domain name
- **DO NOT match** domain suffixes (e.g., `cvoce4scuiue`), ports (e.g., `3306`, `6379`),
  or generic keywords (e.g., `rds`, `mysql`, `redis`). These are shared across multiple
  instances and will cause false positives.
- When a match is found, trace back to the owning **Deployment/StatefulSet/DaemonSet**
  via the pod's `ownerReferences`

**Step 3a-3: Aggregate, validate, and deduplicate results**

For each discovered match, **validate** by reading the actual endpoint value from the
source (ConfigMap data, env var value, etc.) and confirming it contains the target
resource identifier:

1. Read the actual value from the matched source (ConfigMap key, env var, etc.)
2. Check if the value contains `{RESOURCE_ID}.` (dot anchor prevents matching
   `cluster-xxx-replica` when target is `cluster-xxx`)
3. Mark validated matches as "verified", discard false positives

**Note on Secrets:** For Priority 3 (Secrets), only inspect key names, never decode
values. Mark matches as "⚠️ may reference" (uncertain) since key names alone cannot
confirm the actual endpoint — these skip validation and require user confirmation.

Merge validated results from all clusters into a single table:

```
Dependency discovery results (scanned 3 clusters):

EKS Cluster: eks-cluster-prod
  RDS (cluster-xxx):
    ✅ payments/payment-api     — env: DB_HOST
       Verified: cluster-xxx.abc.us-east-1.rds.amazonaws.com
    ✅ orders/order-service     — configmap: orders/app-config (key: spring.datasource.url)
       Verified: jdbc:mysql://cluster-xxx.abc.us-east-1.rds.amazonaws.com:3306/...
    ❌ users/user-service       — discarded (connects to cluster-yyy, different instance)
    ⚠️  billing/billing-api     — secret key: billing/db-credentials (key: DB_HOST) — may reference this service

  ElastiCache (my-redis):
    ✅ payments/payment-api     — env: REDIS_HOST
       Verified: my-redis.abc.use1.cache.amazonaws.com
    ✅ sessions/session-mgr     — service: sessions/redis-svc (ExternalName → my-redis.abc...)

EKS Cluster: eks-cluster-staging
    ⬚ No dependencies found on affected services

Total: 4 verified, 1 discarded, 1 unverified (secret)
```

#### 3b. User Confirmation and Manual Supplement

Ask the user to confirm the auto-discovered dependencies and add any that were
missed. Store the final mapping as `SERVICE_APP_MAP` (service → list of
`{cluster}/{namespace}/{deployment}` tuples).

For multi-cluster setups, the map includes the cluster name:
```
SERVICE_APP_MAP:
  RDS (cluster-xxx):
    - eks-cluster-prod/payments/payment-api
    - eks-cluster-prod/orders/order-service
  ElastiCache (my-redis):
    - eks-cluster-prod/payments/payment-api
    - eks-cluster-prod/sessions/session-mgr
```

### Step 4: Log Collection

> **Shell scripting rule:** Use multi-line scripts. Do NOT chain commands with `&&`
> on a single line — variables get lost after background `&` processes.

All logs should be saved to a temp directory: `/tmp/{timestamp}-fis-app-logs/`,
organized by service name subdirectories.

#### Real-time Mode: Background Collection

For each application in `SERVICE_APP_MAP`, start background `kubectl logs -f` processes
for **regular containers only** (excluding FIS-injected ephemeral containers).

**Multi-cluster note:** For each application, use the kubeconfig for its cluster:
```bash
KUBECONFIG="${KUBECONFIG_DIR}/${CLUSTER_NAME}.kubeconfig" kubectl ...
```
All `kubectl` commands below implicitly use the correct cluster's kubeconfig.

1. Resolve the deployment's pod label selector from `.spec.selector.matchLabels`
2. Get the list of **regular container names** from the deployment spec:
   ```bash
   kubectl get deployment {DEPLOYMENT} -n {NAMESPACE} \
     -o jsonpath='{.spec.template.spec.containers[*].name}'
   ```
   Do NOT use `--all-containers=true` — FIS pod-level fault injection (e.g.,
   `pod-network-latency`, `pod-cpu-stress`) injects ephemeral containers into target
   pods. Using `--all-containers` would pull in FIS agent logs (noise) alongside
   application logs. Always use `--container={name}` to collect only regular containers.
3. For **each** regular container, start a background log stream:
   ```bash
   kubectl logs -f --selector={labels} -n {NAMESPACE} \
     --container={CONTAINER_NAME} --timestamps --prefix=true \
     --max-log-requests=20 \
     >> {LOG_DIR}/{service-name}/{deployment}.log &
   ```
   Use `--selector={labels}` (NOT `deployment/xxx`) — this captures logs from all
   matching pods, including those recreated during the experiment.
4. Record each background PID to `{LOG_DIR}/.pids` for cleanup

#### Post-hoc Mode: Batch Fetch

In post-hoc mode, pods may have been terminated during the experiment. First detect
whether Container Insights is available, then choose the log source accordingly.

**Step 4a: Detect Container Insights**

Check whether the EKS cluster has Container Insights enabled:
- Look for `amazon-cloudwatch-observability` EKS addon (via `aws eks describe-addon`)
- Or check for CloudWatch agent / Fluent Bit daemonset in `amazon-cloudwatch` namespace

**Step 4b: CloudWatch Logs (preferred, if Container Insights is enabled)**

Query CloudWatch Logs Insights against the log group
`/aws/containerinsights/{CLUSTER_NAME}/application` for the experiment time window
(`START_TIME` to `END_TIME`). Filter by `kubernetes.namespace_name` and
`kubernetes.labels.app` (or pod name pattern) for each deployment. This captures
complete logs including from pods that no longer exist.

**Step 4c: kubectl logs (fallback, no Container Insights)**

Use `kubectl logs --selector={labels} --since-time={START_TIME}` with
`--container={CONTAINER_NAME} --timestamps --prefix=true` for each regular container
(same container discovery as real-time mode Step 2). Do NOT use `--all-containers`.
Note: this only retrieves logs from currently running pods — logs from pods terminated
during the experiment are lost.

### Step 5: Real-time Monitoring Display

Poll every 30 seconds while the experiment is running. For each service group and
each application:

1. Read the last 30 seconds of collected logs from the log file
2. Count error-level entries (match: `error`, `exception`, `fail`, `refused`, `timeout`)
   and warning-level entries (match: `warn`, `retry`)
3. Display a per-app summary: error count, warning count, last 5 error lines
4. Detect recovery signals (`connected`, `restored`, `success`, `recovered`) in
   recent lines and report if found

### Step 6: Check Experiment Status (Real-time Mode)

Use `aws fis list-experiments` to check if the experiment with the matching
template ID is still in `running` state. When the experiment completes (or is not
found), proceed to report generation.

### Step 7: Generate Analysis Report

After experiment completes (or immediately in post-hoc mode):

#### Step 7a: Collect Managed Service Logs

If `MANAGED_LOG_GROUPS` is non-empty (from Step 2.5), query CloudWatch Logs Insights
for each recorded log group using the experiment time window. See
`references/managed-service-log-commands.md` for the query script and ASG activity
collection commands.

This step only collects and saves logs — analysis is done in Step 7b together with
application logs.

#### Step 7b: Analyze All Logs and Generate Report

Read **all** log files from `{LOG_DIR}/` — both application logs (`{app}.log`) and
managed service logs (`managed-service-logs.log`). Analyze them together to produce
a unified report with cross-correlation between application-level errors and
infrastructure-level events.

See `references/report-template.md` for the complete report structure and file naming.

### Step 8: Cleanup (Real-time Mode)

Kill all background `kubectl logs` processes recorded in `{LOG_DIR}/.pids`.
Remove the PID file after cleanup.

## Error Handling

| Error | Cause | Resolution |
|-------|-------|------------|
| `/.pids: Permission denied` | `LOG_DIR` variable empty due to `&&` chain — path resolves to `/.pids` | Use `export LOG_DIR=...` with multi-line script, NOT `&&` chains. See Step 4 notes. |
| `kubectl: command not found` | kubectl not installed | Install kubectl and configure kubeconfig |
| `error: You must be logged in` | kubeconfig not configured or expired | The skill auto-generates per-cluster kubeconfig via `aws eks update-kubeconfig --kubeconfig {path}`. Check IAM permissions for EKS. |
| `No resources found` | Deployment/pod doesn't exist | Verify deployment name and namespace |
| `Unable to retrieve logs` | Pod not running or restarted | Check pod status, may need to fetch from CloudWatch Logs |
| Template ID not found | README format changed | Manually provide template ID |
| `AccessDeniedException` on `eks:DescribeCluster` | IAM does not allow EKS access | Ensure caller has `eks:ListClusters` and `eks:DescribeCluster` permissions |
| Cluster not accessible via kubectl | VPC private endpoint or security group restriction | Skip the cluster and note in output; user may need VPN or bastion access |

## Output Files

- `{EXPERIMENT_DIR}/{timestamp}-app-log-analysis.md` — Analysis report
- `/tmp/{timestamp}-fis-app-logs/` — Raw logs organized by service subdirectories
  (app logs + managed service logs). See `references/report-template.md` appendix for
  full directory layout.

## Usage Examples

```
# Real-time monitoring (during experiment)
"Analyze app logs for ./2026-03-31-14-30-22-az-power-interruption-my-cluster/"
"Monitor application behavior in the experiment directory"
"实时监控应用日志"

# Post-hoc analysis (after experiment)
"Analyze app logs using ./2026-03-31-14-35-00-az-power-interruption-my-cluster-experiment-results.md"
"分析实验报告中的应用表现"
"Check what happened to applications during the experiment"
```

## Integration with Other Skills

- **aws-fis-experiment-prepare** — Reads `README.md` and `expected-behavior.md` for context
- **aws-fis-experiment-execute** — Reads `*-experiment-results.md` for time range and service list
- Does NOT modify any files from other skills
