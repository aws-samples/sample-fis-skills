# SSM Automation Generic API Fault Injection Guide

## Scope

This guide applies when the user wants to inject a fault into an **AWS service that
has no native FIS action** (e.g., MSK, OpenSearch, MemoryDB reboot, MQ, Redshift,
Neptune, AppStream, etc.). The approach uses the FIS action
`aws:ssm:start-automation-execution` to trigger an SSM Automation runbook that calls
the target service's API directly.

**Use this guide when:**
- The user requests fault injection for a service, and `aws fis list-actions` returns
  no native action for that service
- The user explicitly asks for SSM Automation-based fault injection
- The target fault scenario maps to an existing AWS API operation (e.g.,
  `kafka:RebootBroker`, `opensearch:RestartDomain`, `mq:RebootBroker`)

**Do NOT use this guide when:**
- A native FIS action exists for the service (e.g., `aws:rds:failover-db-cluster`)
- The fault requires OS-level injection on an EC2 instance — use
  `aws:ssm:send-command` with an SSM Command document instead

## Architecture Overview

```
FIS Experiment Template
  └── Action: aws:ssm:start-automation-execution
        ├── documentArn: arn:aws:ssm:{REGION}:{ACCOUNT_ID}:document/{DOC_NAME}
        ├── documentParameters: {"param1": "value1", ...}
        └── maxDuration: PT15M
              │
              ▼
SSM Automation Runbook (schema 0.3)
  ├── Step 1: ExecuteFaultApi  (aws:executeAwsApi)
  │     └── Calls target service API (e.g., kafka:RebootBroker)
  ├── Step 2: WaitForRecovery  (aws:waitForAwsResourceProperty)  [optional]
  │     └── Polls until resource returns to healthy state
  └── Step 3: VerifyState  (aws:assertAwsResourceProperty)  [optional]
        └── Asserts resource is back to expected state
```

## SSM Automation Runbook Template

The runbook uses **schema version 0.3** (required for Automation documents). It
leverages `aws:executeAwsApi` to call any AWS API via Boto3 service namespaces.

### Minimal Template (Single API Call)

```yaml
description: >-
  FIS fault injection: {FAULT_DESCRIPTION}.
  Invoked via aws:ssm:start-automation-execution from FIS.
schemaVersion: '0.3'
assumeRole: '{{ AutomationAssumeRoleArn }}'
parameters:
  AutomationAssumeRoleArn:
    type: String
    description: IAM Role ARN for SSM Automation to assume
  # Add service-specific parameters below:
  ResourceIdentifier:
    type: String
    description: Target resource identifier (e.g., cluster ARN, broker ID)
mainSteps:
  - name: ExecuteFaultApi
    action: aws:executeAwsApi
    inputs:
      Service: '{SERVICE_NAMESPACE}'  # e.g., kafka, opensearchservice, mq
      Api: '{API_OPERATION}'          # e.g., RebootBroker, RestartDomain
      # API-specific parameters:
      '{ParamName1}': '{{ ResourceIdentifier }}'
    outputs:
      - Name: ApiResponse
        Selector: '$'
        Type: StringMap
```

### Extended Template (Fault + Wait + Verify)

```yaml
description: >-
  FIS fault injection: {FAULT_DESCRIPTION} with recovery verification.
  Invoked via aws:ssm:start-automation-execution from FIS.
schemaVersion: '0.3'
assumeRole: '{{ AutomationAssumeRoleArn }}'
parameters:
  AutomationAssumeRoleArn:
    type: String
    description: IAM Role ARN for SSM Automation to assume
  ResourceIdentifier:
    type: String
    description: Target resource identifier
mainSteps:
  - name: ExecuteFaultApi
    action: aws:executeAwsApi
    inputs:
      Service: '{SERVICE_NAMESPACE}'
      Api: '{API_OPERATION}'
      '{ParamName1}': '{{ ResourceIdentifier }}'
    outputs:
      - Name: ApiResponse
        Selector: '$'
        Type: StringMap

  - name: WaitForRecovery
    action: aws:waitForAwsResourceProperty
    timeoutSeconds: 600
    inputs:
      Service: '{SERVICE_NAMESPACE}'
      Api: '{DESCRIBE_API}'  # e.g., DescribeCluster, DescribeBroker
      '{DescribeParam}': '{{ ResourceIdentifier }}'
      PropertySelector: '{JSON_PATH_TO_STATUS}'  # e.g., $.ClusterInfo.State
      DesiredValues:
        - '{HEALTHY_STATE}'  # e.g., ACTIVE, RUNNING

  - name: VerifyState
    action: aws:assertAwsResourceProperty
    inputs:
      Service: '{SERVICE_NAMESPACE}'
      Api: '{DESCRIBE_API}'
      '{DescribeParam}': '{{ ResourceIdentifier }}'
      PropertySelector: '{JSON_PATH_TO_STATUS}'
      DesiredValues:
        - '{HEALTHY_STATE}'
```

## Service Example Catalog

### Amazon MSK — Reboot Broker

**API:** `kafka:RebootBroker`
**Use case:** Test Kafka consumer/producer resilience to broker restarts

```yaml
description: 'FIS: Reboot MSK broker to test Kafka client resilience'
schemaVersion: '0.3'
assumeRole: '{{ AutomationAssumeRoleArn }}'
parameters:
  AutomationAssumeRoleArn:
    type: String
    description: IAM Role ARN for SSM Automation to assume
  ClusterArn:
    type: String
    description: MSK cluster ARN
  BrokerId:
    type: String
    description: 'Broker ID to reboot (e.g., 1, 2, 3)'
mainSteps:
  - name: RebootMskBroker
    action: aws:executeAwsApi
    inputs:
      Service: kafka
      Api: RebootBroker
      ClusterArn: '{{ ClusterArn }}'
      BrokerIds:
        - '{{ BrokerId }}'
    outputs:
      - Name: ClusterArn
        Selector: $.ClusterArn
        Type: String
      - Name: ClusterOperationArn
        Selector: $.ClusterOperationArn
        Type: String

  - name: WaitForClusterActive
    action: aws:waitForAwsResourceProperty
    timeoutSeconds: 600
    inputs:
      Service: kafka
      Api: DescribeCluster
      ClusterArn: '{{ ClusterArn }}'
      PropertySelector: $.ClusterInfo.State
      DesiredValues:
        - ACTIVE
```

**IAM permissions for Automation Role:**
```json
{
  "Effect": "Allow",
  "Action": [
    "kafka:RebootBroker",
    "kafka:DescribeCluster"
  ],
  "Resource": "{MSK_CLUSTER_ARN}"
}
```

**Discover broker IDs:**
```bash
aws kafka list-nodes --cluster-arn {CLUSTER_ARN} \
  --query 'NodeInfoList[].BrokerNodeInfo.BrokerId' --output json
```

### Amazon MQ — Reboot Broker

**API:** `mq:RebootBroker`
**Use case:** Test application resilience to ActiveMQ/RabbitMQ broker restarts

```yaml
mainSteps:
  - name: RebootMqBroker
    action: aws:executeAwsApi
    inputs:
      Service: mq
      Api: RebootBroker
      BrokerId: '{{ BrokerId }}'
```

**IAM permissions:** `mq:RebootBroker`, `mq:DescribeBroker`

### Amazon OpenSearch — Restart Domain (rolling)

**API:** No direct restart API — use `opensearch:UpdateDomainConfig` with a
benign config change to trigger a rolling blue/green deployment, or use
`opensearch:UpgradeDomain` for a version upgrade.

**Alternative fault approach:** Use `aws:ssm:send-command` on the underlying
EC2 instances (if using managed-node access), or use `aws:network:disrupt-connectivity`
to disrupt network to the OpenSearch domain endpoint.

### Amazon Redshift — Reboot Cluster

**API:** `redshift:RebootCluster`

```yaml
mainSteps:
  - name: RebootRedshiftCluster
    action: aws:executeAwsApi
    inputs:
      Service: redshift
      Api: RebootCluster
      ClusterIdentifier: '{{ ClusterIdentifier }}'

  - name: WaitForAvailable
    action: aws:waitForAwsResourceProperty
    timeoutSeconds: 900
    inputs:
      Service: redshift
      Api: DescribeClusters
      ClusterIdentifier: '{{ ClusterIdentifier }}'
      PropertySelector: $.Clusters[0].ClusterStatus
      DesiredValues:
        - available
```

**IAM permissions:** `redshift:RebootCluster`, `redshift:DescribeClusters`

### Amazon Neptune — Failover DB Cluster

**API:** `neptune:FailoverDBCluster`

```yaml
mainSteps:
  - name: FailoverNeptuneCluster
    action: aws:executeAwsApi
    inputs:
      Service: neptune
      Api: FailoverDBCluster
      DBClusterIdentifier: '{{ DBClusterIdentifier }}'

  - name: WaitForAvailable
    action: aws:waitForAwsResourceProperty
    timeoutSeconds: 600
    inputs:
      Service: neptune
      Api: DescribeDBClusters
      DBClusterIdentifier: '{{ DBClusterIdentifier }}'
      PropertySelector: $.DBClusters[0].Status
      DesiredValues:
        - available
```

**IAM permissions:** `neptune:FailoverDBCluster`, `neptune:DescribeDBClusters`

## CFN Template Integration

### SSM Document Resource

The SSM Automation runbook is deployed as an `AWS::SSM::Document` in the same
CFN template alongside the FIS experiment template.

```yaml
Resources:
  SSMAutomationDocument:
    Type: AWS::SSM::Document
    Properties:
      DocumentType: Automation
      DocumentFormat: YAML
      Name: !Sub 'fis-${ExperimentName}-runbook'
      Content:
        # Paste the full runbook YAML content here
        description: 'FIS: ...'
        schemaVersion: '0.3'
        assumeRole: '{{ AutomationAssumeRoleArn }}'
        parameters:
          AutomationAssumeRoleArn:
            type: String
            description: IAM Role ARN
          # ... service-specific parameters
        mainSteps:
          # ... steps

  SSMAutomationRole:
    Type: AWS::IAM::Role
    Properties:
      RoleName: !Sub 'fis-ssm-auto-role-${RandomSuffix}'
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: ssm.amazonaws.com
            Action: sts:AssumeRole
      Policies:
        - PolicyName: ServiceApiAccess
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Sid: TargetServiceAccess
                Effect: Allow
                Action:
                  # Service-specific permissions (e.g., kafka:RebootBroker)
                  - '{SERVICE}:{API_ACTION}'
                  - '{SERVICE}:{DESCRIBE_ACTION}'
                Resource: '{RESOURCE_ARN}'
```

### FIS Experiment Template Action

```yaml
FISExperimentTemplate:
  Type: AWS::FIS::ExperimentTemplate
  DependsOn:
    - FISExperimentRole
    - SSMAutomationDocument
  Properties:
    Description: !Sub 'FIS: ${ExperimentName}'
    RoleArn: !GetAtt FISExperimentRole.Arn
    StopConditions:
      - Source: 'none'
    Tags:
      Name: !Ref ExperimentName
    Targets: {}  # No targets needed — SSM Automation handles targeting
    Actions:
      ExecuteSSMAutomation:
        ActionId: aws:ssm:start-automation-execution
        Parameters:
          documentArn:
            - !Sub 'arn:aws:ssm:${AWS::Region}:${AWS::AccountId}:document/fis-${ExperimentName}-runbook'
          documentParameters:
            - !Sub >-
              {"AutomationAssumeRoleArn":"${SSMAutomationRole.Arn}",
               "ResourceIdentifier":"{RESOURCE_VALUE}"}
          maxDuration:
            - 'PT15M'
```

### FIS Experiment Role — Additional Permissions

The FIS Experiment Role must have `AWSFaultInjectionSimulatorSSMAccess` managed
policy attached (or equivalent inline permissions):

```yaml
FISExperimentRole:
  Type: AWS::IAM::Role
  Properties:
    RoleName: !Sub 'fis-role-${RandomSuffix}'
    AssumeRolePolicyDocument:
      Version: '2012-10-17'
      Statement:
        - Effect: Allow
          Principal:
            Service: fis.amazonaws.com
          Action: sts:AssumeRole
    ManagedPolicyArns:
      - arn:aws:iam::aws:policy/service-role/AWSFaultInjectionSimulatorSSMAccess
    Policies:
      - PolicyName: PassSSMAutomationRole
        PolicyDocument:
          Version: '2012-10-17'
          Statement:
            - Sid: PassRoleToSSM
              Effect: Allow
              Action: iam:PassRole
              Resource: !GetAtt SSMAutomationRole.Arn
              Condition:
                StringEquals:
                  iam:PassedToService: ssm.amazonaws.com
```

## IAM Design — Two-Role Pattern

**This approach requires TWO IAM roles:**

| Role | Trusted By | Purpose | Permissions |
|---|---|---|---|
| FIS Experiment Role | `fis.amazonaws.com` | FIS starts the SSM Automation execution | `AWSFaultInjectionSimulatorSSMAccess` + `iam:PassRole` for SSM Automation Role |
| SSM Automation Role | `ssm.amazonaws.com` | SSM Automation calls the target service API | Target service API permissions (e.g., `kafka:RebootBroker`) |

**Why two roles?** FIS invokes `ssm:StartAutomationExecution` using the FIS
Experiment Role, but the SSM Automation runbook itself needs to call the target
service API (e.g., `kafka:RebootBroker`). SSM Automation requires a separate
`assumeRole` that trusts `ssm.amazonaws.com` — the FIS role (which trusts
`fis.amazonaws.com`) cannot be used directly by SSM Automation.

## Discovery Workflow

When a user requests fault injection for a service without native FIS actions:

1. **Verify no native FIS action exists:**
   ```bash
   aws fis list-actions --query "actions[?starts_with(id, 'aws:{SERVICE}:')]" \
     --region {REGION} --output table
   ```
   If no results, proceed with SSM Automation approach.

2. **Identify the target API operation.** Look up the Boto3 service namespace
   and the disruptive API operation:
   ```bash
   # Reference: https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/{SERVICE}.html
   ```

3. **Identify the describe/status API** for recovery verification (optional
   but recommended).

4. **Determine IAM permissions** required for both the fault API and the
   describe API.

5. **Generate the SSM document + CFN template** using the templates in this guide.

## Scenario Slug

When generating the output directory and CFN stack name, use `ssm-auto` as a
prefix for the scenario slug, followed by the service and operation:

| Scenario | SCENARIO_SLUG | Example Stack Name |
|---|---|---|
| MSK broker reboot | `ssm-auto-msk-reboot` | `fis-ssm-auto-msk-reboot-my-cluster-a3x7k2` |
| MQ broker reboot | `ssm-auto-mq-reboot` | `fis-ssm-auto-mq-reboot-my-broker-b8y2m1` |
| Redshift cluster reboot | `ssm-auto-rs-reboot` | `fis-ssm-auto-rs-reboot-my-cluster-c4z9n3` |
| Neptune failover | `ssm-auto-np-failover` | `fis-ssm-auto-np-failover-my-db-d5w1p7` |

**Abbreviation rules:** `redshift` → `rs`, `neptune` → `np`, `opensearch` → `os`,
`memorydb` → `memdb`

## Key Constraints

| Constraint | Detail |
|---|---|
| `aws:executeAwsApi` max duration | 25 seconds per step execution. For APIs that return quickly (e.g., `RebootBroker`), this is sufficient. For long-running operations, use `aws:waitForAwsResourceProperty` to poll. |
| SSM document naming | Max 128 characters. Use `fis-{ExperimentName}-runbook` pattern. |
| `documentParameters` in FIS | Must be a JSON string (not a JSON object). FIS passes this as a single string parameter to `StartAutomationExecution`. |
| SSM Automation concurrency | Default max 25 concurrent automation executions per account per region. For high-frequency experiments, request a quota increase. |
| Rollback | SSM Automation runbooks support `onFailure` and `onCancel` handlers. Add rollback steps if the fault API requires explicit cleanup (most reboot/restart APIs are self-recovering). |

## Common Mistakes

| Mistake | Fix |
|---|---|
| Using FIS Experiment Role as SSM Automation `assumeRole` | FIS role trusts `fis.amazonaws.com`, not `ssm.amazonaws.com`. Create a separate SSM Automation Role. |
| Passing `documentParameters` as JSON object instead of string | In CFN, use `!Sub` with the JSON stringified. In CLI, wrap in single quotes. |
| Forgetting `iam:PassRole` on FIS Experiment Role | FIS needs to pass the SSM Automation Role to SSM. Add `iam:PassRole` with condition `iam:PassedToService: ssm.amazonaws.com`. |
| Not waiting for recovery | Without `aws:waitForAwsResourceProperty`, the FIS experiment ends immediately after the API call, before the fault has full effect. Add a wait/verify step. |
| Hardcoding region/account in document ARN | Use `!Sub` with `${AWS::Region}` and `${AWS::AccountId}` in CFN templates. |
