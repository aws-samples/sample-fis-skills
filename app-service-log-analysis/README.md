[English](README.md) | [中文](README_CN.md)

# EKS App Log Analysis

An Agent Skill for analyzing EKS application logs during AWS FIS fault injection experiments to understand how applications respond to infrastructure failures.

## Problem Background

During BCP (Business Continuity Plan) drills with AWS FIS:

- **Service-level reports lack application perspective** — FIS experiment reports show AWS service behavior (RDS failover time, ElastiCache status) but not how applications actually responded
- **Manual log collection is tedious** — Correlating kubectl logs with experiment timelines requires manual effort
- **Dependency mapping is implicit** — When you disrupt RDS, you need to know which applications depend on it to collect relevant logs
- **Real-time visibility is limited** — During experiments, operators want to see application behavior as it happens, not just after

## Core Features

1. **Dual-mode operation** — Real-time monitoring during experiments OR post-hoc analysis after experiments
2. **Multi-cluster deep dependency discovery** — Automatically discovers ALL EKS clusters in the target region, generates isolated kubeconfig files (never overwrites `~/.kube/config`), and scans all accessible clusters in parallel. Deep scan covers pod env vars, ConfigMaps, Secret key names (metadata only), EnvFrom references, Service ExternalName, and volume mounts (projected/CSI).
3. **Managed service log collection** — Automatically detects CloudWatch logging for EKS control plane, RDS/Aurora, ElastiCache, MSK, and OpenSearch; queries logs for the experiment time window (extended by 3 minutes past experiment end to capture recovery baseline) to correlate with application-level impact
4. **Parallel log collection** — Background `kubectl logs -f` processes for multiple applications across multiple clusters simultaneously, collecting only regular containers (excluding FIS-injected ephemeral containers)
5. **Live insight display** — Every 30 seconds: actual error logs (5 lines) + analysis insights per service group
6. **Comprehensive analysis report** — Error timelines, patterns, cross-service correlation, managed service log insights, recovery analysis

## Workflow Overview

```
Mode Detection
├── Directory input → Real-time mode
│   ├── Read README.md for template ID
│   ├── Check if experiment is running
│   └── Start background log collection
└── Report file input → Post-hoc mode
    ├── Parse Start/End time from report
    └── Batch fetch historical logs

Common Steps
├── Discover all EKS clusters in region + generate isolated kubeconfigs
├── Read service list from expected-behavior.md or report
├── Deep-scan all clusters for app dependencies + validate matches against target instances
├── Detect and collect managed service logs (EKS/RDS/ElastiCache/MSK/OpenSearch)
├── Collect application logs across clusters (background streaming or batch)
├── Display insights (real-time) or analyze (post-hoc)
└── Generate analysis report (includes managed service log correlation)
```

## Real-time Display Format

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
[10:05:32] RDS cluster-xxx Impact Analysis
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

▶ app-backend (last 30s: 12 errors, 3 warnings)
┌─────────────────────────────────────────────────────────────┐
│ 2026-04-01T10:05:01Z ERROR Failed to connect to database    │
│ 2026-04-01T10:05:03Z ERROR Connection refused (10.0.1.50)   │
│ 2026-04-01T10:05:05Z WARN  Retry attempt 3/5                │
│ 2026-04-01T10:05:12Z ERROR Connection timeout after 10s     │
│ 2026-04-01T10:05:18Z INFO  Database connection restored     │
└─────────────────────────────────────────────────────────────┘
💡 Insight: 12 errors between 10:05:01 - 10:05:15
✅ Recovery signal detected at 10:05:18
```

## Analysis Report

The generated report includes:

- Experiment metadata (ID, time range with 3-min post-experiment baseline, duration)
- Summary table: errors per application, peak error rate, recovery time
- Per-service application analysis:
  - Error timeline table
  - Key error patterns with counts
  - Actual log samples (5-10 lines)
  - Insights and correlation with fault injection events
- Cross-service correlation timeline (includes post-baseline window annotations showing experiment end and baseline collection period)
- Recommendations for improvement

## Prerequisites

- **kubectl** — Log collection. Must be installed locally (kubeconfig per cluster is auto-generated).
- **AWS CLI** — Query FIS experiment status, EKS cluster discovery, managed service log queries.
- **Experiment directory** — Context source, from aws-fis-experiment-prepare.
- **Experiment report** — Time range source, from aws-fis-experiment-execute.
- **IAM permissions** — `eks:ListClusters`, `eks:DescribeCluster` for multi-cluster discovery.
- (Optional) **CloudWatch Container Insights** — Enables richer pod/node-level metrics for correlation. See [Enabling Container Insights](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Container-Insights-setup-EKS-quickstart.html).
- (Optional) **EKS Control Plane Logging** — Enables API server, audit, and scheduler logs for deeper analysis. See [Enabling Control Plane Logs](https://docs.aws.amazon.com/eks/latest/userguide/control-plane-logs.html).

## Output Files

```
{experiment-dir}/
├── {timestamp}-app-log-analysis.md    # Analysis report (saved to experiment directory)
│
/tmp/{timestamp}-fis-app-logs/         # Temp directory for raw logs
├── {service-1}/
│   ├── {app-1}.log
│   └── {app-2}.log
└── {service-2}/
    └── {app-3}.log
```

## Usage Examples

```
# Real-time monitoring (during experiment)
"Analyze app logs for ./2026-03-31-az-power-interruption/"
"Monitor application behavior during the experiment"
"实时监控应用日志"

# Post-hoc analysis (after experiment)
"Analyze logs using ./2026-03-31-experiment-results.md"
"分析实验后的应用日志"
```

## Related Skills

- [aws-fis-experiment-prepare](../aws-fis-experiment-prepare/) — Generates experiment config and expected-behavior.md
- [aws-fis-experiment-execute](../aws-fis-experiment-execute/) — Runs experiment and generates results report
- [aws-service-chaos-research](../aws-service-chaos-research/) — Research chaos testing scenarios

## Directory Structure

```
app-service-log-analysis/
├── SKILL.md          # Main skill definition
├── README.md         # This file (English)
├── README_CN.md      # Chinese documentation
└── references/
    ├── managed-service-log-commands.md   # Managed service log collection commands
    └── report-template.md               # Log analysis report template
```

## Known Limitations

- Requires kubectl installed locally; kubeconfig per cluster is auto-generated in the log directory
- Multi-cluster scanning requires `eks:ListClusters` and `eks:DescribeCluster` permissions
- Private EKS clusters may not be accessible without VPN/bastion; inaccessible clusters are skipped
- Pod restarts during experiment may cause log gaps (kubectl logs only shows current pod)
- FIS pod-level fault injection uses ephemeral containers — the skill explicitly excludes these to avoid noise in application logs
- For long-running experiments, consider using CloudWatch Logs Insights instead
- Real-time mode requires experiment to be running; if experiment completed, use post-hoc mode
