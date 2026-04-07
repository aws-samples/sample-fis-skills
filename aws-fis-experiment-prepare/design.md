# FIS Experiment Execution Skills Design

**Date:** 2026-03-25
**Status:** Approved

## Overview

Two independent skills for creating and executing AWS FIS (Fault Injection Service)
experiments. Skill 1 generates all configuration files and deploys them via
CloudFormation with self-healing iteration; Skill 2 starts and monitors the experiment.

## Skills

| Skill | Purpose | Input | Output |
|---|---|---|---|
| **aws-fis-experiment-prepare** | Discover resources, generate configs, deploy via CFN | User describes scenario | Local directory + deployed CFN stack |
| **aws-fis-experiment-execute** | Start experiment, monitor, report results | Prepare output directory (or deployed stack) | Experiment results report |

## Skill 1: aws-fis-experiment-prepare

### Triggers
- "prepare FIS experiment", "create FIS experiment for..."
- "generate AZ Power Interruption config"
- "准备 FIS 实验", "生成混沌实验配置"

### Workflow
1. Identify scenario (Scenario Library or custom FIS action)
2. Discover target resources (CLI: list-actions, user provides tags/ARNs)
3. **Validate resource-action compatibility** — inspect actual resources via CLI
   (describe-db-instances, describe-db-clusters, etc.), cross-check against FIS
   action's required `resourceType`, suggest alternatives if incompatible
4. Determine monitoring metrics per affected service
5. Generate all config files to local directory
6. **Deploy CFN template with self-healing loop:**
   - Validate template syntax (`validate-template`)
   - Deploy stack (`cloudformation deploy`)
   - On failure: read stack events -> diagnose error -> fix `cfn-template.yaml` -> delete failed stack -> retry
   - Max 5 retries; on success extract outputs and update local files with real ARNs
7. Present summary (stack name, experiment template ID, dashboard URL, next steps)

### Output Directory
```
./az-power-interruption-yyyy-mm-dd-HH-MM-SS/
├── README.md
├── experiment-template.json
├── iam-policy.json
├── cfn-template.yaml        # Full: IAM role + alarms + experiment template
├── alarms/
│   ├── stop-condition-alarms.json
│   └── dashboard.json
└── expected-behavior.md
```

### Scenario Sources
- **Scenario Library:** AZ Power Interruption, AZ Application Slowdown, Cross-AZ Traffic Slowdown, Cross-Region Connectivity, EC2/EKS/EBS stress series
- **Custom FIS action:** User specifies action ID and parameters

## Skill 2: aws-fis-experiment-execute

### Triggers
- "execute FIS experiment", "run FIS experiment"
- "启动 FIS 实验", "运行混沌实验"

### Workflow
1. Load and validate experiment directory (or use already-deployed stack from Prepare)
2. User chooses deployment method: CLI or CFN (skip if already deployed by Prepare)
3. Deploy infrastructure if needed (user confirms)
4. Start experiment (user confirms with impact warning)
5. Monitor experiment status (poll + dashboard URL)
6. Generate results report

### Safety Gates
- Step 3: User confirms resource creation
- Step 4: User explicitly confirms experiment start with impact summary
- All CLI commands shown before execution
- Dry-run mode supported

## Design Decisions
- **Independent skills:** No dependency on aws-service-chaos-research
- **File as interface:** Prepare outputs files, Execute reads them
- **Both CLI + CFN:** Single CFN template contains all resources
- **Prepare deploys CFN:** Prepare skill auto-deploys and self-heals on failure (up to 5 retries), so user receives a validated, working stack
- **Strict confirmation:** Experiment start requires explicit user approval (Prepare deploys infra automatically, but never starts the experiment)
- **Resource-action compatibility gate:** Before generating any files, inspect the actual resource type via CLI and verify it matches the FIS action's required resourceType. Prevents wasted effort (e.g., `aws:rds:failover-db-cluster` targeting standalone RDS instead of Aurora)
- **Language follows user:** Output matches conversation language
