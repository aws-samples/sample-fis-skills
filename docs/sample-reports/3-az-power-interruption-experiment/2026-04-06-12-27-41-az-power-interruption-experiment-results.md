## FIS Experiment Results

**Experiment ID:** EXPtmxxxPai
**Template ID:**   EXTMExxxfVa
**Stack:**         fis-az-power-interruption-demo-cluster-xxx
**Status:**        completed
**Start Time:**    2026-04-06T12:27:41+00:00
**End Time:**      2026-04-06T12:42:57+00:00
**Duration:**      15 分 16 秒

### Action Results

| Action | Action ID | Status | Start (UTC) | End (UTC) | Duration |
|---|---|---|---|---|---|
| Stop-EKS-Instances | aws:ec2:stop-instances | completed | 12:27:54 | 12:42:57 | 15m 3s |
| Pause-ASG-Scaling | aws:ec2:asg-insufficient-instance-capacity-error | completed | 12:27:54 | 12:42:54 | 15m 0s |
| Pause-ElastiCache | aws:elasticache:replicationgroup-interrupt-az-power | completed | 12:27:54 | 12:42:54 | 15m 0s |
| Pause-Instance-Launches | aws:ec2:api-insufficient-instance-capacity-error | completed | 12:27:55 | 12:42:55 | 15m 0s |
| Pause-Network-Connectivity | aws:network:disrupt-connectivity | completed | 12:27:54 | 12:42:55 | 15m 1s |
| Reboot-RDS-Failover | aws:rds:reboot-db-instances | completed | 12:27:55 | 12:27:56 | 1s |

### Stop Condition Alarms

| Alarm | Final Status |
|---|---|
| (none configured) | N/A |

### Per-Service Impact Analysis

#### EC2 / EKS 节点 (i-0836xxx3ab, i-0f72xxx676)

| Time (UTC) | Event | Observation |
|---|---|---|
| 12:27:54 | Stop-EKS-Instances 开始 | FIS 开始停止 us-west-2a 中的两个 EKS 节点 |
| 12:28:19 | 节点 NotReady | ip-10-x-x-11 和 ip-10-x-x-216 变为 NotReady/SchedulingDisabled |
| 12:28:38 | Pod 开始 Terminating | 运行在 us-west-2a 节点上的 Pod 进入 Terminating 状态 |
| 12:28:38 | 新 Pod Pending | Kubernetes 尝试调度替代 Pod，但因 ASG 扩容被阻止，无法在 us-west-2a 启动新节点 |
| 12:42:57 | Stop-EKS-Instances 完成 | 实例被标记为 terminated（非 stopped），ASG 启动新实例替代 |
| 12:43:47 | 新节点加入 | 多个新节点 (ip-10-x-x-179, ip-10-x-x-194, ip-10-x-x-191 等) 加入集群 |

**Key Findings:**
- 两个 us-west-2a 节点被 **terminated**（而非 stopped），ASG 自动启动了新实例替代
- Kubernetes 快速检测到节点不可用，将 Pod 标记为 Terminating 并尝试在其他节点重新调度
- 由于 Pause-ASG-Scaling 阻止了 us-west-2a 的扩容，部分 Pod 长时间处于 Pending 状态（约15分钟）
- 实验结束后，ASG 在多个 AZ 启动了新节点，Pod 开始恢复调度

#### RDS MySQL (demo-mysql)

| Time (UTC) | Event | Observation |
|---|---|---|
| 12:27:55 | Reboot-RDS-Failover 开始 | 触发 Multi-AZ 强制故障转移 |
| 12:27:56 | Reboot-RDS-Failover 完成 | FIS action 在 1 秒内完成（实际故障转移仍在进行） |
| 12:28:00 | catalog 连接超时开始 | `read tcp 10.x.x.215:50222->10.x.x.76:3306: i/o timeout` — 旧主节点 IP 不可达 |
| 12:28:10 | catalog 持续超时 | `dial tcp 10.x.x.76:3306: i/o timeout` — 新连接也无法建立 |
| ~12:29:30 | catalog 恢复正常 | catalog 开始返回 200 响应，响应时间 6-23ms，DNS 已解析到新主节点 |
| 12:43:45 | DNS 解析超时 | 新 pod 出现 `lookup demo-mysql.xxx.rds.amazonaws.com: i/o timeout`（网络恢复过渡期） |

**Key Findings:**
- RDS Multi-AZ 故障转移成功完成，主节点从 us-west-2a 切换到 us-west-2c
- catalog 服务经历了约 **90 秒** 的数据库不可用期，期间所有查询超时
- 恢复后 catalog 服务正常响应，说明应用成功重连到新主节点
- catalog 应用未实现连接池健康检查或快速失败机制，导致超时持续时间较长

#### ElastiCache Redis (demo-redis)

| Time (UTC) | Event | Observation |
|---|---|---|
| 12:27:54 | Pause-ElastiCache 开始 | 中断 us-west-2a 节点的电力 |
| 12:28:19 | 状态变为 modifying | ElastiCache 检测到节点丢失，开始内部故障转移 |
| 12:42:54 | Pause-ElastiCache 完成 | FIS action 完成 |
| 12:43:47 | 仍在 modifying | ElastiCache 仍在恢复中 |

**Key Findings:**
- ElastiCache Redis 的 Multi-AZ 自动故障转移 **对应用透明** — checkout 服务 **未记录任何错误**
- Redis 集群模式 (2 分片 x 3 节点) 提供了良好的高可用性保护
- 实验结束后 ElastiCache 仍处于 modifying 状态，完全恢复需要额外时间
- checkout 应用使用集群模式连接 (clustercfg endpoint)，节点故障转移对客户端完全透明

#### 子网网络连通性 (subnet-0151xxx707, subnet-0e0dxxxb70)

| Time (UTC) | Event | Observation |
|---|---|---|
| 12:27:54 | Pause-Network-Connectivity 开始 | us-west-2a private-a 和 public-a 子网全部网络中断 |
| 12:28:00 | 连接超时开始 | 所有经过 us-west-2a 子网的流量中断 |
| 12:42:55 | Pause-Network-Connectivity 完成 | 网络连通性恢复 |

**Key Findings:**
- 网络中断放大了 RDS 故障转移的影响，导致 catalog 在 DNS 切换前无法通过旧 IP 连接
- CloudWatch Agent（部署在 amazon-cloudwatch 命名空间）也受到网络中断影响，导致 OTel exporter 持续超时

#### ASG (eks-demo-nodegroup-xxx)

| Time (UTC) | Event | Observation |
|---|---|---|
| 12:27:54 | Pause-ASG-Scaling 开始 | 阻止 ASG 在 us-west-2a 扩容 |
| 12:28:38 | Pod Pending | 新 Pod 无法调度，因为 us-west-2a 无法扩容节点 |
| 12:42:54 | Pause-ASG-Scaling 完成 | ASG 恢复正常扩容能力 |
| 12:43:47 | 新节点启动 | ASG 在 us-west-2a 和 us-west-2b 启动新节点 |

**Key Findings:**
- ASG 扩容限制成功模拟了 AZ 完全不可用场景
- 实验期间 7 个 Pod 持续 Pending，无法调度到新节点
- us-west-2b 的现有节点 (ip-10-x-x-28, ip-10-x-x-81) 接收了部分重新调度的 Pod

#### CloudWatch Agent (间接影响)

| Time (UTC) | Event | Observation |
|---|---|---|
| 12:28:10 | OTel exporter 超时开始 | orders, catalog 的 OTel agent 无法将 metrics/traces 发送到 CloudWatch Agent |
| 12:28:29 | catalog traces 导出超时 | `Post "http://cloudwatch-agent.amazon-cloudwatch:4316/v1/traces": context deadline exceeded` |
| 12:43:29 | 仍在超时 | 网络恢复后仍有部分超时（恢复过渡期） |

**Key Findings:**
- CloudWatch Agent pod 可能运行在 us-west-2a 节点上，导致 OTel 收集中断
- orders 服务的大部分 "errors" 实际是 OTel exporter 超时，**非业务逻辑错误**
- 建议确保 CloudWatch Agent 跨 AZ 部署以提高可观测性的可用性

### Recovery Status Summary

| Resource | Recovery Status | Notes |
|---|---|---|
| EKS 节点 (us-west-2a) | Recovering | 原实例 terminated，新节点正在启动并加入集群 |
| RDS MySQL (demo-mysql) | Recovered | 已故障转移至 us-west-2c，主节点可用，Multi-AZ 已重建 |
| ElastiCache Redis (demo-redis) | Recovering | 仍处于 modifying 状态，应用层已恢复正常 |
| 子网网络 | Recovered | 网络连通性已恢复 |
| ASG 扩容 | Recovered | 已恢复正常扩容，新节点正在启动 |
| catalog 应用 | Recovered | MySQL 故障转移后约 90 秒恢复 |
| checkout 应用 | Not Impacted | Redis 故障转移对应用透明 |
| orders 应用 | Recovering | OTel exporter 仍有超时，业务功能正常 |

### Issues Requiring Attention

#### 1. catalog 服务 RDS 故障转移恢复时间过长 (~90秒)
- **Problem:** catalog 使用直接 TCP 连接到 RDS，未实现快速失败或连接池健康检查。RDS 故障转移期间，旧连接超时需等待较长时间。
- **Recommendation:** 配置 MySQL 连接池的 `connectTimeout` 和 `socketTimeout` 为较短值（如 5 秒），并启用连接验证。考虑使用 RDS Proxy 来处理故障转移。

#### 2. EKS 实例被 terminated 而非 stopped
- **Problem:** 实验中 EC2 实例被 terminated，导致 ASG 需要启动全新实例。实验模板配置了 `startInstancesAfterDuration: PT15M`，但实例在恢复前被终止。
- **Recommendation:** 检查 ASG 终止保护设置。实际 AZ 故障场景中实例 terminate 是预期行为，确保应用能容忍节点替换。

#### 3. CloudWatch Agent 可观测性中断
- **Problem:** CloudWatch Agent 可能集中部署在 us-west-2a，导致实验期间 OTel metrics/traces 收集中断。
- **Recommendation:** 确保 CloudWatch Agent DaemonSet 的 Pod 分布在所有 AZ，并配置 Pod Topology Spread Constraints。

#### 4. 多个 Pod 长时间 Pending (15分钟)
- **Problem:** 由于 ASG 扩容被阻止，7 个 Pod 在实验全程处于 Pending 状态。
- **Recommendation:** 考虑配置 Pod Disruption Budgets (PDB) 确保关键服务的最小副本数，并将副本分布在多个 AZ 使用 topologySpreadConstraints。
