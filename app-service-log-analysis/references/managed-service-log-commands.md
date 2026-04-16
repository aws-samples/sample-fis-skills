# Managed Service Log Commands Reference

CLI commands for detecting logging status, querying CloudWatch Logs, and collecting
ASG activity history during FIS experiments.

---

## Check Logging Status

| Service | Command | Log Group Format |
|---|---|---|
| EKS Control Plane | `aws eks describe-cluster --name {CLUSTER} --query 'cluster.logging.clusterLogging'` — check for enabled log types (`api`, `audit`, `scheduler`, `controllerManager`, `authenticator`) | `/aws/eks/{cluster-name}/cluster` |
| RDS/Aurora | `aws rds describe-db-clusters --db-cluster-identifier {CLUSTER_ID} --query 'DBClusters[0].EnabledCloudwatchLogsExports'` — returns list like `["error","slowquery"]` | `/aws/rds/cluster/{cluster-id}/{log-type}` |
| ElastiCache | `aws elasticache describe-replication-groups --replication-group-id {RG_ID} --query 'ReplicationGroups[0].LogDeliveryConfigurations'` — check for `cloudwatch-logs` destination type | Log group from `LogDeliveryConfigurations[].DestinationDetails.CloudWatchLogsDetails.LogGroup` |
| MSK (Kafka) | `aws kafka describe-cluster --cluster-arn {ARN} --query 'ClusterInfo.LoggingInfo.BrokerLogs'` — check `CloudWatchLogs.Enabled` | Log group from `CloudWatchLogs.LogGroup` |
| OpenSearch | `aws opensearch describe-domain --domain-name {DOMAIN} --query 'DomainStatus.LogPublishingOptions'` — check for keys like `INDEX_SLOW_LOGS`, `SEARCH_SLOW_LOGS`, `ES_APPLICATION_LOGS` | Log group from each option's `CloudWatchLogsLogGroupArn` |

## Key Events to Look For

| Service | Key Events |
|---|---|
| EKS Control Plane | `NodeNotReady`, pod eviction, rescheduling decisions, API server errors |
| RDS/Aurora | Failover start/complete timestamps, connection errors, slow queries during transition |
| ElastiCache | Node failover, cluster rebalancing, connection drops |
| MSK | Broker offline/online, partition reassignment, under-replicated partitions |
| OpenSearch | Shard relocation, node departure/join, cluster yellow/red status |

---

## CloudWatch Logs Insights Query

Use for querying managed service logs in Step 7a. `EPOCH_END` should be
`EXPERIMENT_END_TIME + 3 minutes` (180 seconds) to cover the post-experiment
baseline window.

**CRITICAL: Always verify the log group exists before querying.** The service API
(e.g., `describe-db-instances`) may report a log type as "enabled", but the
corresponding CloudWatch log group may not actually exist (e.g., if the instance
was recently created or the log type has never produced output). Querying a
non-existent log group causes `start-query` to fail with an empty `queryId`,
which then causes the polling loop to run infinitely.

```bash
# 0. Verify log group exists (MANDATORY — do NOT skip this step)
LOG_GROUP="{LOG_GROUP}"
if ! aws logs describe-log-groups \
  --log-group-name-prefix "${LOG_GROUP}" \
  --query "logGroups[?logGroupName==\`${LOG_GROUP}\`].logGroupName" \
  --output text | grep -q "${LOG_GROUP}"; then
  echo "SKIP: Log group ${LOG_GROUP} does not exist (service reports logging enabled but log group not found)"
  # Record in MANAGED_LOG_RECOMMENDATIONS for the report
  continue  # skip to next log group
fi

# 1. Start the query
QUERY_ID=$(aws logs start-query \
  --log-group-name "${LOG_GROUP}" \
  --start-time {EPOCH_START} \
  --end-time {EPOCH_END} \
  --query-string 'fields @timestamp, @message | sort @timestamp asc | limit 500' \
  --query 'queryId' --output text 2>/dev/null)

# 2. Guard against empty queryId (start-query failed)
if [ -z "$QUERY_ID" ] || [ "$QUERY_ID" = "None" ]; then
  echo "SKIP: Failed to start query for ${LOG_GROUP} (queryId is empty)"
  continue  # skip to next log group
fi

# 3. Poll until complete (with max retry to prevent infinite loop)
MAX_POLL=30
POLL_COUNT=0
while [ $POLL_COUNT -lt $MAX_POLL ]; do
  STATUS=$(aws logs get-query-results --query-id "$QUERY_ID" \
    --query 'status' --output text 2>/dev/null)
  if [ "$STATUS" = "Complete" ] || [ "$STATUS" = "Failed" ] || [ "$STATUS" = "Cancelled" ]; then
    break
  fi
  POLL_COUNT=$((POLL_COUNT + 1))
  sleep 2
done

if [ "$STATUS" != "Complete" ]; then
  echo "WARN: Query for ${LOG_GROUP} did not complete (status: ${STATUS}, polls: ${POLL_COUNT})"
  continue  # skip to next log group
fi

# 4. Save results to local file
mkdir -p "{LOG_DIR}/{service-name}"
aws logs get-query-results --query-id "$QUERY_ID" \
  --query 'results[].[*].join(`\t`, [value])' --output text \
  > "{LOG_DIR}/{service-name}/managed-service-logs.log"
```

Repeat for each log group in `MANAGED_LOG_GROUPS`.

---

## ASG Activity History

Always available (no logging enablement needed). Collect when the experiment
involves Auto Scaling Groups (e.g., AZ Power Interruption).

```bash
mkdir -p "{LOG_DIR}/asg-{asg-name}"
aws autoscaling describe-scaling-activities \
  --auto-scaling-group-name "{ASG_NAME}" \
  --start-time {ISO_START} \
  --max-items 100 \
  --query 'Activities[?StatusCode!=`Cancelled`]' \
  > "{LOG_DIR}/asg-{asg-name}/scaling-activities.log"
```

Key events: instance launch/terminate, `InsufficientInstanceCapacity` errors,
health check failures, capacity rebalancing across AZs.
