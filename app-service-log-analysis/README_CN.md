[English](README.md) | [中文](README_CN.md)

# EKS 应用日志分析

一个 Agent Skill，用于在 AWS FIS 故障注入实验期间分析 EKS 应用日志，了解应用对基础设施故障的实际响应。

## 问题背景

在使用 AWS FIS 进行 BCP（业务连续性计划）演练时：

- **服务层报告缺少应用视角** — FIS 实验报告展示 AWS 服务行为（RDS 故障转移时间、ElastiCache 状态），但不显示应用的实际响应
- **手动收集日志繁琐** — 将 kubectl 日志与实验时间线关联需要大量手动工作
- **依赖关系是隐式的** — 中断 RDS 时，需要知道哪些应用依赖它才能收集相关日志
- **实时可见性有限** — 实验期间，运维人员希望实时看到应用行为，而不仅仅是事后查看

## 核心功能

1. **双模式运行** — 实验期间实时监控 或 实验后事后分析
2. **多集群深度依赖发现** — 自动发现目标 Region 中所有 EKS 集群，为每个集群生成独立的 kubeconfig 文件（绝不覆盖 `~/.kube/config`），并行扫描所有可访问的集群。深度扫描覆盖 Pod 环境变量、ConfigMap、Secret 键名（仅元数据）、EnvFrom 引用、Service ExternalName、Volume 挂载（projected/CSI）。
3. **托管服务日志收集** — 自动检测 EKS 控制平面、RDS/Aurora、ElastiCache、MSK、OpenSearch 的 CloudWatch 日志是否开启；已开启的自动查询实验时间窗口内的日志（查询结束时间延长至实验结束后 3 分钟，以捕获恢复基线），与应用层影响交叉关联
4. **跨集群并行日志收集** — 后台 `kubectl logs -f` 进程同时收集多个集群上多个应用的日志，仅收集常规容器日志（排除 FIS 注入的临时容器）
5. **实时洞察展示** — 每 30 秒：实际错误日志（5 行）+ 按服务分组的分析洞察
6. **全面分析报告** — 错误时间线、模式识别、跨服务关联、托管服务日志洞察、恢复分析

## 工作流程概览

```
模式检测
├── 目录输入 → 实时模式
│   ├── 从 README.md 读取 template ID
│   ├── 检查实验是否正在运行
│   └── 启动后台日志收集
└── 报告文件输入 → 事后模式
    ├── 从报告解析开始/结束时间
    └── 批量获取历史日志

公共步骤
├── 发现 Region 内所有 EKS 集群 + 生成独立 kubeconfig
├── 从 expected-behavior.md 或报告读取服务列表
├── 深度扫描所有集群的应用依赖（env vars、ConfigMap、Secrets、ExternalName 等）
├── 检测并收集托管服务日志（EKS/RDS/ElastiCache/MSK/OpenSearch）
├── 跨集群收集应用日志（后台流式或批量）
├── 展示洞察（实时）或分析（事后）
└── 生成分析报告（包含托管服务日志关联分析）
```

## 实时展示格式

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
[10:05:32] RDS cluster-xxx 影响分析
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

▶ app-backend（最近 30s：12 个错误，3 个警告）
┌─────────────────────────────────────────────────────────────┐
│ 2026-04-01T10:05:01Z ERROR 无法连接数据库                    │
│ 2026-04-01T10:05:03Z ERROR Connection refused (10.0.1.50)   │
│ 2026-04-01T10:05:05Z WARN  重试第 3/5 次                     │
│ 2026-04-01T10:05:12Z ERROR 连接超时 10 秒                    │
│ 2026-04-01T10:05:18Z INFO  数据库连接已恢复                   │
└─────────────────────────────────────────────────────────────┘
💡 洞察：10:05:01 - 10:05:15 期间发生 12 个错误
✅ 10:05:18 检测到恢复信号
```

## 分析报告

生成的报告包含：

- 实验元数据（ID、时间范围含 3 分钟实验后基线窗口、持续时间）
- 汇总表：每个应用的错误数、峰值错误率、恢复时间
- 按服务分组的应用分析：
  - 错误时间线表
  - 关键错误模式及数量
  - 实际日志样本（5-10 行）
  - 洞察及与故障注入事件的关联
- 跨服务关联时间线（包含实验后基线窗口标注，显示实验结束和基线采集时段）
- 改进建议

## 前置条件

- **kubectl** — 日志收集。需要本地安装，每个集群的 kubeconfig 自动生成。
- **AWS CLI** — 查询 FIS 实验状态、EKS 集群发现、托管服务日志查询。
- **实验目录** — 上下文来源，来自 aws-fis-experiment-prepare。
- **实验报告** — 时间范围来源，来自 aws-fis-experiment-execute。
- **IAM 权限** — 多集群发现需要 `eks:ListClusters`、`eks:DescribeCluster` 权限。
- （可选）**CloudWatch Container Insights** — 启用后可提供更丰富的 Pod/Node 级别指标用于关联分析。参见 [启用 Container Insights](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Container-Insights-setup-EKS-quickstart.html)。
- （可选）**EKS 控制平面日志** — 启用 API Server、Audit、Scheduler 等日志，用于更深入的分析。参见 [启用控制平面日志](https://docs.aws.amazon.com/eks/latest/userguide/control-plane-logs.html)。

## 输出文件

```
{实验目录}/
├── {时间戳}-app-log-analysis.md       # 分析报告（保存到实验目录）
│
/tmp/{时间戳}-fis-app-logs/            # 原始日志临时目录
├── {服务-1}/
│   ├── {应用-1}.log
│   └── {应用-2}.log
└── {服务-2}/
    └── {应用-3}.log
```

## 使用示例

```
# 实时监控（实验期间）
"分析 ./2026-03-31-az-power-interruption/ 目录的应用日志"
"监控实验期间的应用表现"
"Analyze app logs for the experiment"

# 事后分析（实验后）
"使用 ./2026-03-31-experiment-results.md 分析应用日志"
"分析实验报告中的应用表现"
```

## 相关 Skill

- [aws-fis-experiment-prepare](../aws-fis-experiment-prepare/) — 生成实验配置和 expected-behavior.md
- [aws-fis-experiment-execute](../aws-fis-experiment-execute/) — 运行实验并生成结果报告
- [aws-service-chaos-research](../aws-service-chaos-research/) — 研究混沌测试场景

## 目录结构

```
app-service-log-analysis/
├── SKILL.md          # Skill 主定义文件
├── README.md         # 英文文档
├── README_CN.md      # 本文件（中文）
└── references/
    ├── managed-service-log-commands.md   # 托管服务日志收集命令
    └── report-template.md               # 日志分析报告模板
```

## 已知限制

- 需要本地安装 kubectl；每个集群的 kubeconfig 自动生成到日志目录
- 多集群扫描需要 `eks:ListClusters` 和 `eks:DescribeCluster` 权限
- 私有 EKS 集群可能无法通过 VPN/堡垒机以外的方式访问；不可访问的集群会被跳过
- 实验期间 Pod 重启可能导致日志间隙（kubectl logs 只显示当前 Pod 日志）
- FIS pod 级故障注入使用临时容器（ephemeral containers）— 本 skill 明确排除这些容器以避免应用日志中的噪音
- 对于长时间运行的实验，建议改用 CloudWatch Logs Insights
- 实时模式需要实验正在运行；如果实验已结束，请使用事后模式
