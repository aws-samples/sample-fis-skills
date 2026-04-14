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
2. **Smart dependency discovery** — Reads experiment context and guides user to specify app-to-service dependencies
3. **Parallel log collection** — Background `kubectl logs -f` processes for multiple applications simultaneously, collecting only regular containers (excluding FIS-injected ephemeral containers)
4. **Live insight display** — Every 30 seconds: actual error logs (5 lines) + analysis insights per service group
5. **Comprehensive analysis report** — Error timelines, patterns, cross-service correlation, recovery analysis

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
├── Read service list from expected-behavior.md or report
├── Ask user for dependent applications per service
├── Collect logs (background streaming or batch)
├── Display insights (real-time) or analyze (post-hoc)
└── Generate analysis report
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

- Experiment metadata (ID, time range, duration)
- Summary table: errors per application, peak error rate, recovery time
- Per-service application analysis:
  - Error timeline table
  - Key error patterns with counts
  - Actual log samples (5-10 lines)
  - Insights and correlation with fault injection events
- Cross-service correlation timeline
- Recommendations for improvement

## Prerequisites

- **kubectl** — Log collection. Must have access to target EKS cluster.
- **AWS CLI** — Query FIS experiment status (for real-time mode).
- **Experiment directory** — Context source, from aws-fis-experiment-prepare.
- **Experiment report** — Time range source, from aws-fis-experiment-execute.
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
eks-app-log-analysis/
├── SKILL.md          # Main skill definition
├── README.md         # This file (English)
└── README_CN.md      # Chinese documentation
```

## Known Limitations

- Requires kubectl access to EKS cluster; logs not in cluster are not captured
- Pod restarts during experiment may cause log gaps (kubectl logs only shows current pod)
- FIS pod-level fault injection uses ephemeral containers — the skill explicitly excludes these to avoid noise in application logs
- For long-running experiments, consider using CloudWatch Logs Insights instead
- Real-time mode requires experiment to be running; if experiment completed, use post-hoc mode
