# Scenario Templates Reference

JSON skeleton templates for FIS Scenario Library scenarios. Replace placeholders
(in `{CURLY_BRACES}`) with actual values from user input.

## AZ Availability: Power Interruption

This is the most complex Scenario Library scenario. It orchestrates multiple sub-actions
to simulate a complete AZ power failure.

**Default duration:** 30 min interruption + 30 min recovery

**Sub-actions and their default tags:**

| Sub-Action | FIS Action ID | Default Tag Key | Default Tag Value | Target Type |
|---|---|---|---|---|
| Stop-Instances | `aws:ec2:stop-instances` | `AzImpairmentPower` | `StopInstances` | `aws:ec2:instance` |
| Stop-ASG-Instances | `aws:ec2:stop-instances` | `AzImpairmentPower` | `IceAsg` | `aws:ec2:instance` |
| Pause Instance Launches | `aws:ec2:api-insufficient-instance-capacity-error` | — | — | `aws:iam:role` (ARN) |
| Pause ASG Scaling | `aws:ec2:asg-insufficient-instance-capacity-error` | `AzImpairmentPower` | `IceAsg` | `aws:ec2:autoscaling-group` |
| Pause Network Connectivity | `aws:network:disrupt-connectivity` | `AzImpairmentPower` | `DisruptSubnet` | `aws:ec2:subnet` |
| Failover RDS | `aws:rds:failover-db-cluster` | `AzImpairmentPower` | `DisruptRds` | `aws:rds:cluster` |
| Pause ElastiCache | `aws:elasticache:replicationgroup-interrupt-az-power` | `AzImpairmentPower` | `ElasticacheImpact` | `aws:elasticache:replicationgroup` |
| Start ARC Zonal Autoshift | `aws:arc:start-zonal-autoshift` | `AzImpairmentPower` | `RecoverAutoshiftResources` | ARC resources |
| Pause EBS I/O | `aws:ebs:pause-io` | `AzImpairmentPower` | `APIPauseVolume` | `aws:ebs:volume` |
| Disrupt S3 Express | `aws:s3:disrupt-connectivity-express-one-zone` | `AzImpairmentPower` | `DisruptS3Express` | `aws:s3:bucket` |

**Template skeleton (experiment-template.json):**

```json
{
    "tags": {
        "Name": "AZ-Power-Interruption-{AZ_ID}",
        "Scenario": "AZ-Availability-Power-Interruption",
        "CreatedBy": "fis-experiment-prepare-skill"
    },
    "description": "Simulate complete AZ power interruption in {AZ_ID}",
    "targets": {
        "instances-target": {
            "resourceType": "aws:ec2:instance",
            "resourceArns": ["{EC2_INSTANCE_ARN_1}", "{EC2_INSTANCE_ARN_2}"],
            "filters": [
                {
                    "path": "State.Name",
                    "values": ["running"]
                }
            ],
            "selectionMode": "ALL"
        },
        "asg-instances-target": {
            "resourceType": "aws:ec2:instance",
            "resourceArns": ["{ASG_INSTANCE_ARN_1}", "{ASG_INSTANCE_ARN_2}"],
            "filters": [
                {
                    "path": "State.Name",
                    "values": ["running"]
                }
            ],
            "selectionMode": "ALL"
        },
        "iam-role-target": {
            "resourceType": "aws:iam:role",
            "resourceArns": ["{IAM_ROLE_ARN_FOR_INSTANCE_LAUNCHES}"],
            "selectionMode": "ALL"
        },
        "asg-target": {
            "resourceType": "aws:ec2:autoscaling-group",
            "resourceTags": {
                "{TAG_KEY}": "{ASG_TAG_VALUE}"
            },
            "selectionMode": "ALL"
        },
        "subnet-target": {
            "resourceType": "aws:ec2:subnet",
            "resourceArns": ["{SUBNET_ARN_1}", "{SUBNET_ARN_2}"],
            "selectionMode": "ALL"
        },
        "rds-cluster-target": {
            "resourceType": "aws:rds:cluster",
            "resourceArns": ["{RDS_CLUSTER_ARN}"],
            "selectionMode": "ALL"
        },
        "elasticache-target": {
            "resourceType": "aws:elasticache:replicationgroup",
            "resourceTags": {
                "{TAG_KEY}": "{ELASTICACHE_TAG_VALUE}"
            },
            "selectionMode": "ALL"
        }
    },
    "actions": {
        "StopInstances": {
            "actionId": "aws:ec2:stop-instances",
            "description": "Stop EC2 instances in target AZ",
            "parameters": {
                "startInstancesAfterDuration": "PT30M"
            },
            "targets": {
                "Instances": "instances-target"
            }
        },
        "StopASGInstances": {
            "actionId": "aws:ec2:stop-instances",
            "description": "Stop ASG-managed instances in target AZ",
            "parameters": {
                "startInstancesAfterDuration": "PT30M"
            },
            "targets": {
                "Instances": "asg-instances-target"
            }
        },
        "PauseInstanceLaunches": {
            "actionId": "aws:ec2:api-insufficient-instance-capacity-error",
            "description": "Block new instance launches in target AZ",
            "parameters": {
                "availabilityZoneIdentifiers": "{AZ_ID}",
                "duration": "PT30M",
                "percentage": "100"
            },
            "targets": {
                "Roles": "iam-role-target"
            }
        },
        "PauseASGScaling": {
            "actionId": "aws:ec2:asg-insufficient-instance-capacity-error",
            "description": "Block ASG scaling in target AZ",
            "parameters": {
                "availabilityZoneIdentifiers": "{AZ_ID}",
                "duration": "PT30M",
                "percentage": "100"
            },
            "targets": {
                "AutoScalingGroups": "asg-target"
            }
        },
        "PauseNetworkConnectivity": {
            "actionId": "aws:network:disrupt-connectivity",
            "description": "Block all network connectivity for subnets in target AZ",
            "parameters": {
                "duration": "PT2M",
                "scope": "all"
            },
            "targets": {
                "Subnets": "subnet-target"
            }
        },
        "FailoverRDS": {
            "actionId": "aws:rds:failover-db-cluster",
            "description": "Failover RDS cluster if writer is in target AZ",
            "targets": {
                "Clusters": "rds-cluster-target"
            }
        },
        "PauseElastiCache": {
            "actionId": "aws:elasticache:replicationgroup-interrupt-az-power",
            "description": "Interrupt ElastiCache nodes in target AZ",
            "parameters": {
                "availabilityZoneIdentifier": "{AZ_ID}",
                "duration": "PT30M"
            },
            "targets": {
                "ReplicationGroups": "elasticache-target"
            }
        }
    },
    "stopConditions": [
        {
            "source": "none"
        }
    ],
    "roleArn": "{FIS_EXECUTION_ROLE_ARN}"
}
```

**Notes:**
- Remove target/action blocks for services the user doesn't have
- Sub-actions with no matching targets are automatically skipped by FIS
- The `startAfter` field can be used to sequence actions (e.g., network disruption
  before RDS failover)
- If the user provides a stop condition alarm, replace `"source": "none"` with
  `"source": "aws:cloudwatch:alarm"` and add `"value": "{ALARM_ARN}"`

---

## AZ: Application Slowdown

**Default duration:** 30 min

Adds latency between resources within a single AZ.

**Key sub-actions:**

| Sub-Action | FIS Action ID | Target Type |
|---|---|---|
| EC2 Network Latency | `aws:ssm:send-command` (AWSFIS-Run-Network-Latency) | `aws:ec2:instance` |
| EKS Pod Network Latency | `aws:eks:pod-network-latency` | `aws:eks:pod` |

```json
{
    "tags": {
        "Name": "AZ-Application-Slowdown-{AZ_ID}",
        "Scenario": "AZ-Application-Slowdown"
    },
    "description": "Add latency between resources within {AZ_ID}",
    "targets": {
        "ec2-instances": {
            "resourceType": "aws:ec2:instance",
            "resourceArns": ["{EC2_INSTANCE_ARN_1}", "{EC2_INSTANCE_ARN_2}"],
            "selectionMode": "ALL"
        }
    },
    "actions": {
        "InjectLatency": {
            "actionId": "aws:ssm:send-command",
            "parameters": {
                "duration": "PT30M",
                "documentArn": "arn:aws:ssm:{REGION}::document/AWSFIS-Run-Network-Latency",
                "documentParameters": "{\"DelayMilliseconds\":\"200\",\"Interface\":\"eth0\",\"DurationSeconds\":\"1800\",\"InstallDependencies\":\"True\"}"
            },
            "targets": {
                "Instances": "ec2-instances"
            }
        }
    },
    "stopConditions": [
        {
            "source": "none"
        }
    ],
    "roleArn": "{FIS_EXECUTION_ROLE_ARN}"
}
```

---

## Cross-AZ: Traffic Slowdown

**Default duration:** 30 min

Injects packet loss to disrupt cross-AZ traffic.

**Key sub-actions:**

| Sub-Action | FIS Action ID | Target Type |
|---|---|---|
| Network Packet Loss | `aws:network:transit-gateway-disrupt-cross-region-connectivity` | Transit Gateway |
| EKS Pod Packet Loss | `aws:eks:pod-network-packet-loss` | `aws:eks:pod` |

---

## Custom Single-Action Template

For custom FIS actions, use this generic template:

```json
{
    "tags": {
        "Name": "{EXPERIMENT_NAME}",
        "CreatedBy": "fis-experiment-prepare-skill"
    },
    "description": "{EXPERIMENT_DESCRIPTION}",
    "targets": {
        "{TARGET_NAME}": {
            "resourceType": "{RESOURCE_TYPE}",
            "resourceArns": ["{RESOURCE_ARN}"],
            "selectionMode": "{SELECTION_MODE}"
        }
    },
    "actions": {
        "{ACTION_NAME}": {
            "actionId": "{FIS_ACTION_ID}",
            "description": "{ACTION_DESCRIPTION}",
            "parameters": {
                "{PARAM_KEY}": "{PARAM_VALUE}"
            },
            "targets": {
                "{TARGET_KEY}": "{TARGET_NAME}"
            }
        }
    },
    "stopConditions": [
        {
            "source": "none"
        }
    ],
    "roleArn": "{FIS_EXECUTION_ROLE_ARN}"
}
```

**To discover action parameters:**
```bash
aws fis get-action --id "{ACTION_ID}" --region {REGION} --output json
```

This returns the action definition including required targets and parameters.

---

## EC2 Stress Scenarios

Simple single-action scenarios using SSM documents.

### CPU Stress
```json
{
    "tags": {"Name": "EC2-CPU-Stress"},
    "description": "Stress test CPU on target EC2 instances",
    "targets": {
        "instances": {
            "resourceType": "aws:ec2:instance",
            "resourceArns": ["{EC2_INSTANCE_ARN}"],
            "selectionMode": "ALL"
        }
    },
    "actions": {
        "CPUStress": {
            "actionId": "aws:ssm:send-command",
            "parameters": {
                "duration": "PT10M",
                "documentArn": "arn:aws:ssm:{REGION}::document/AWSFIS-Run-CPU-Stress",
                "documentParameters": "{\"DurationSeconds\":\"300\",\"InstallDependencies\":\"True\",\"CPU\":\"0\"}"
            },
            "targets": {"Instances": "instances"}
        }
    },
    "stopConditions": [{"source": "none"}],
    "roleArn": "{ROLE_ARN}"
}
```

### Memory Stress
Same structure, replace document with `AWSFIS-Run-Memory-Stress` and parameters with
`{"DurationSeconds":"300","InstallDependencies":"True","Percent":"80"}`.

### Disk Stress
Same structure, replace document with `AWSFIS-Run-Disk-Fill` and parameters with
`{"DurationSeconds":"300","InstallDependencies":"True","Percent":"80","Path":"/"}`.

### Network Latency
Same structure, replace document with `AWSFIS-Run-Network-Latency` and parameters with
`{"DelayMilliseconds":"200","Interface":"eth0","DurationSeconds":"300","InstallDependencies":"True"}`.
