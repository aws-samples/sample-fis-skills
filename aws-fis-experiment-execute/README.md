[English](README.md) | [中文](README_CN.md)

# AWS FIS Experiment Execute

An agent skill that verifies a CloudFormation stack is already deployed, runs an AWS FIS experiment, monitors its progress, and generates a results report. Reads configuration from a prepared experiment directory.

## Problem Statement

Running an AWS FIS experiment after preparation still involves manual verification steps:

- **Stack deployment verification** — Before running an experiment, you need to confirm that the CloudFormation stack deployed successfully and is in `CREATE_COMPLETE` status.
- **Template ID extraction** — The FIS experiment template ID must be extracted from stack outputs before the experiment can be started.
- **Safety is critical** — FIS experiments affect **real production resources**. Starting without proper confirmation, impact review, or stop conditions risks unintended damage.
- **Monitoring during the experiment is manual** — polling experiment status, watching CloudWatch dashboards, and comparing actual behavior against expected behavior must happen simultaneously.
- **Results collection is scattered** — experiment status, action outcomes, timing, and recovery verification are queried from separate CLI commands and must be consolidated manually.

## What This Skill Does

1. **Loads and validates** the prepared experiment directory (from [aws-fis-experiment-prepare](../aws-fis-experiment-prepare/) or manually created).
2. **Reads README.md** to extract the CFN stack name and experiment metadata.
3. **Verifies stack deployment** — checks that the CloudFormation stack is in `CREATE_COMPLETE` or `UPDATE_COMPLETE` status.
4. **Extracts template ID** from stack outputs.
5. **Enforces safety** — presents a clear impact warning with affected resources, requires explicit user confirmation before starting.
6. **Starts the experiment** only after explicit user confirmation.
7. **Monitors progress** — polls experiment status every 30-60 seconds, records timestamps for each status change and per-service events, reminds user to check the dashboard.
8. **Saves results report** — writes the experiment results to a local markdown file (`YYYY-mm-dd-HH-MM-SS-{scenario}-experiment-results.md`) with **per-service impact analysis** where each service has its own timeline, observations, and key findings — so readers can see the full picture for each service without jumping between sections. Prints a brief summary to the terminal.

**Note:** This skill does **NOT** deploy infrastructure. It only verifies that the stack is already deployed and proceeds with experiment execution.

## Workflow Overview

```
Step 1: Load experiment directory + validate required files
         ↓
Step 2: Read README.md → extract CFN stack name + metadata
         ↓
Step 3: Check CloudFormation stack status
         ├── CREATE_COMPLETE or UPDATE_COMPLETE → proceed
         └── Not ready / failed / not found → abort with guidance
         ↓
Step 4: Extract experiment template ID from stack outputs
         ↓
Step 5: Start experiment [CRITICAL — requires explicit user confirmation]
         ├── Display impact warning (resources, duration, stop conditions)
         ├── User confirms → start experiment
         └── User declines → skip to results report
         ↓
Step 6: Monitor experiment
         ├── Poll status every 30s (first 5 min) then 60s
         ├── Show current status after each poll
         ├── Record timestamps for each status change and action transition
         └── Remind user: check dashboard
         ↓
Step 7: Save results report to local file (YYYY-mm-dd-HH-MM-SS-{scenario}-experiment-results.md)
```

## Safety Rules

| Rule | Description |
|---|---|
| **Never auto-start** | Always require explicit user confirmation before starting the experiment |
| **Show every command** | Display each CLI command before executing it |
| **Impact warning** | Show affected resources, region, AZ, duration before start |
| **Abort at every step** | Provide abort instructions throughout the process |
| **No silent deletes** | Never delete resources without user confirmation |
| **Never deploy** | This skill only checks existing deployments, never deploys infrastructure |
| **Recommend dry-run** | Suggest reviewing all files before starting |

## Stack Status Handling

| Status | Action |
|---|---|
| `CREATE_COMPLETE` | Stack is ready. Proceed with experiment. |
| `UPDATE_COMPLETE` | Stack is ready (was updated). Proceed with experiment. |
| `CREATE_IN_PROGRESS` | Stack is still deploying. Wait and re-check. |
| `CREATE_FAILED` | Stack deployment failed. Show failure reason and abort. |
| `ROLLBACK_COMPLETE` | Stack creation failed and rolled back. Show reason and abort. |
| `DELETE_COMPLETE` or not found | Stack does not exist. Inform user to deploy first. |

## Experiment Status Values

| Status | Meaning |
|---|---|
| `initiating` | Experiment is starting |
| `running` | Experiment is in progress |
| `completed` | Experiment finished successfully |
| `stopping` | Being stopped (by user or stop condition) |
| `stopped` | Stopped before completion |
| `failed` | Experiment failed |

## Results Report

After the experiment reaches a terminal state, a results report is saved to a local
markdown file in the experiment directory. The file name uses the format:

```
YYYY-mm-dd-HH-MM-SS-{scenario}-experiment-results.md
```

The report includes:
- Experiment ID, template ID, stack name, final status
- Start time, end time, actual duration (all timestamps in ISO 8601 with timezone)
- Per-action results table (with action ID, status, and duration per action)
- Stop condition alarm status table
- **Per-service impact analysis** — for each affected service, a dedicated sub-section containing:
  - **Key timeline** — only the events relevant to that specific service (timestamps in UTC time-only format), so readers can correlate with CloudWatch Dashboard metrics without leaving the section
  - **Observations** — observed behavior during and after the experiment
  - **Key findings** — what happened, why, and recovery behavior
- Recovery status summary table
- Issues requiring attention (with remediation commands)
- Cleanup instructions

A brief summary is printed to the terminal, including per-service recovery status.

## Required Files

The experiment directory must contain:

| File | Required | Purpose |
|---|---|---|
| `experiment-template.json` | Yes | FIS experiment template |
| `iam-policy.json` | Yes | IAM permissions for FIS role |
| `cfn-template.yaml` | Yes | CloudFormation template (reference) |
| `README.md` | Yes | Experiment overview with CFN stack name |
| `alarms/stop-condition-alarms.json` | Optional | CloudWatch alarm definitions |
| `alarms/dashboard.json` | Optional | CloudWatch dashboard |

**Critical:** The `README.md` must contain the `**CFN Stack:** {STACK_NAME}` field populated with the actual deployed stack name. This is set by `aws-fis-experiment-prepare` after successful deployment.

## Prerequisites

- **AWS CLI** (`aws`) — FIS, CloudWatch, CloudFormation operations. Must have permissions for all services.
- **Prepared experiment directory** — Configuration source, from aws-fis-experiment-prepare or manually created.

## Key CLI Commands

### Check Stack Status
```bash
aws cloudformation describe-stacks \
  --stack-name "{STACK_NAME}" \
  --region {REGION} \
  --query 'Stacks[0].{StackStatus: StackStatus, Outputs: Outputs}'
```

### Extract Template ID from Stack Outputs
```bash
TEMPLATE_ID=$(aws cloudformation describe-stacks \
  --stack-name "{STACK_NAME}" \
  --query 'Stacks[0].Outputs[?OutputKey==`ExperimentTemplateId`].OutputValue' \
  --output text --region {REGION})
```

### Start Experiment
```bash
aws fis start-experiment \
  --experiment-template-id "{TEMPLATE_ID}" \
  --region {REGION}
```

### Get Experiment Status
```bash
aws fis get-experiment \
  --id "{EXPERIMENT_ID}" \
  --region {REGION} \
  --query 'experiment.state.status'
```

### Stop Experiment (Emergency)
```bash
aws fis stop-experiment --id "{EXPERIMENT_ID}" --region {REGION}
```

## Cleanup Guide

### CFN Cleanup (Recommended)
```bash
aws cloudformation delete-stack --stack-name "{STACK_NAME}" --region {REGION}
aws cloudformation wait stack-delete-complete --stack-name "{STACK_NAME}" --region {REGION}
```

### Manual Resource Cleanup (if needed)
```bash
aws fis delete-experiment-template --id "{TEMPLATE_ID}" --region {REGION}
aws cloudwatch delete-alarms --alarm-names "FIS-StopCondition-{SCENARIO}-{SERVICE}" --region {REGION}
aws cloudwatch delete-dashboards --dashboard-names "FIS-{SCENARIO}" --region {REGION}
```

## Error Handling

| Error | Cause | Resolution |
|---|---|---|
| Stack name not found in README | README missing `**CFN Stack:**` field | Check if prepared with recent version of aws-fis-experiment-prepare |
| Stack not found (`ValidationError`) | Stack does not exist or was deleted | Deploy the stack first using aws-fis-experiment-prepare |
| Stack in `CREATE_FAILED` | Stack deployment failed | Check stack events for failure reason, fix and redeploy |
| `ExperimentTemplateId` not in outputs | Stack template missing output | Check cfn-template.yaml for the output definition |
| `AccessDeniedException` | Insufficient permissions | Check IAM permissions for FIS, CloudWatch, CloudFormation |
| `ResourceNotFoundException` on targets | Tagged resources not found | Verify resource tags match experiment template |
| Experiment stuck in `initiating` | IAM role propagation delay | Wait 30 seconds and check again |

## Usage Examples

```
"Execute the FIS experiment in ./2025-03-27-10-30-00-az-power-interruption-my-cluster/"
"Run the chaos experiment I just prepared"
"启动 FIS 实验"
"Check if the stack is deployed and run the experiment"
"运行混沌实验，目录在 ./2025-03-27-rds-failover-prod-db/"
```

## Key Design Decisions

1. **No deployment — only verification.** This skill assumes the CloudFormation stack has already been deployed (by `aws-fis-experiment-prepare` or manually). It verifies the stack status before proceeding.

2. **Stack name from README.** The stack name is extracted from the `**CFN Stack:**` field in the experiment directory's README.md, ensuring consistency with the prepare skill's output.

3. **Explicit confirmation is non-negotiable.** FIS experiments cause real impact. The skill never auto-starts — it always presents a warning with specific resource details and requires the user to type confirmation.

4. **Continuous monitoring with reminders.** During the experiment, the skill polls status and reminds the user to check the CloudWatch dashboard. Operators should not rely solely on terminal output during fault injection.

5. **Results saved to file.** The experiment results report is written to a timestamped local markdown file, keeping terminal output concise while preserving a full record.

6. **Cleanup is offered, not forced.** After the experiment, cleanup commands are suggested but never executed without confirmation.

## Directory Structure

```
aws-fis-experiment-execute/
├── SKILL.md                              # Main skill definition (agent instructions)
├── README.md                             # This file (English)
├── README_CN.md                          # Chinese version
└── references/
    └── cli-commands.md                   # AWS CLI command reference
```

## Limitations

- Requires AWS CLI with permissions for FIS, CloudWatch, and CloudFormation.
- **Does not deploy infrastructure** — expects the stack to be already deployed.
- Monitoring relies on CLI polling; real-time dashboard requires the user to open the CloudWatch console.
- The skill does not handle multi-step recovery verification — it reminds the user to check, but cannot verify application-level health automatically.

## Related Skills

- [aws-fis-experiment-prepare](../aws-fis-experiment-prepare/) — Generate and deploy experiment configuration (run before this skill)
- [aws-service-chaos-research](../aws-service-chaos-research/) — Research chaos testing scenarios for any AWS service
- [eks-workload-best-practice-assessment](../eks-workload-best-practice-assessment/) — Assess EKS workload configurations
