## FIS 实验结果

**实验 ID:** EXPqkxxx1ke
**模板 ID:** EXT2dxxx9fn
**Stack:** fis-eks-pod-network-latency-xxx
**状态:** completed
**开始时间:** 2026-04-02T04:48:36+00:00
**结束时间:** 2026-04-02T04:54:57+00:00
**持续时间:** 6 分 21 秒

### Action 结果

| Action | Action ID | 状态 | 开始 (UTC) | 结束 (UTC) | 持续时间 |
|---|---|---|---|---|---|
| InjectNetworkLatency | aws:eks:pod-network-latency | completed | 04:49:08 | 04:54:36 | 5 分 28 秒 |

### Stop Condition 告警

| 告警 | 最终状态 |
|---|---|
| (无) | 未配置 Stop Condition |

### 各服务影响分析

#### Catalog Pod (retail-store/catalog on demo-cluster)

| 时间 (UTC) | 事件 | 观察 |
|---|---|---|
| 04:48:36 | 实验启动 | 实验进入 initiating 状态 |
| 04:49:08 | Action 开始执行 | FIS 开始对 catalog Pod 注入 20s 网络延迟 |
| 04:49:23 | 首次错误出现 | `dial tcp 10.x.x.101:3306: i/o timeout` — 到 RDS 的连接超时 |
| 04:49:28 | context canceled 错误 | 4 次 context canceled — 请求在等待数据库响应时被上下文超时取消 |
| 04:49:28 | SQL 查询耗时 ~10s | SELECT 查询从正常 2-3ms 飙升至 9998-10001ms |
| 04:49:28 | HTTP 404 响应 | catalog 因数据库超时返回 404 而非正常 200 |
| 04:49:33 - 04:54:19 | 持续性故障 | 每 ~5 秒出现 2 个 timeout 错误，持续约 5 分钟 |
| 04:54:19 | 最后一条错误 | 最后一个 `i/o timeout` 错误 |
| 04:54:25 | 开始恢复 | 首次出现正常 200 响应，延迟恢复到 2-3ms |
| 04:54:29 | 完全恢复 | 所有请求恢复正常，响应时间 0.5-4ms |
| 04:54:36 | Action 完成 | FIS 停止故障注入 |

**关键发现:**
- 网络延迟注入后 **15 秒**内出现首次错误（04:49:08 → 04:49:23）
- 错误模式稳定：每 5 秒 2 个 timeout 错误（分别来自 2 个 catalog Pod 副本）
- Go 应用的 MySQL 连接超时设置为 **5 秒**（所有 timeout 错误均在 5000ms 左右）
- 数据库查询超时导致 **404** 响应（而非 500），说明应用将 "未找到数据" 与 "数据库错误" 混淆
- 恢复**几乎即时**：Action 停止后 ~10 秒内完全恢复正常（04:54:19 最后错误 → 04:54:25 恢复）
- 总计 **124 次错误**：120 次 i/o timeout + 4 次 context canceled

#### RDS MySQL (demo-mysql)

| 时间 (UTC) | 事件 | 观察 |
|---|---|---|
| 04:49:08 | 故障注入开始 | RDS 实例本身不受直接影响 |
| 04:49:23 - 04:54:19 | catalog 连接超时 | catalog Pod 无法在 5s 内建立到 RDS 端点 (10.x.x.101:3306) 的 TCP 连接 |
| 04:54:25 | 连接恢复 | catalog 重新建立到 RDS 的正常连接 |

**关键发现:**
- RDS MySQL 实例本身运行正常，故障完全在网络层
- 网络延迟 20s >> 应用连接超时 5s，导致所有新连接尝试必定失败
- RDS 端点通过 ClusterIP 10.x.x.101 解析，表明可能使用了 Kubernetes Service 或 DNS 转发

#### UI (retail-store/ui) — 间接影响

| 时间 (UTC) | 事件 | 观察 |
|---|---|---|
| 04:49:23 | 首次级联错误 | UI 调用 catalog 服务超时，抛出 HTTPError |
| 04:49:23 - 04:54:19 | 持续性级联错误 | 2546 次错误/异常，主要为 HTTPError (1079) 和 ApiException (476) |
| 04:54:20 | 最后一批错误 | Thymeleaf 模板渲染错误（因 catalog 数据缺失） |
| 04:54:25 | 恢复 | UI 恢复正常页面渲染 |

**关键发现:**
- UI 的错误数 (2546) 远超 catalog 错误数 (124)，因为每个页面请求会触发多个 catalog 子调用
- 错误类型链：SocketTimeoutException → RuntimeException → HTTPError → ApiException → TemplateProcessingException
- UI 缺乏对 catalog 服务降级的优雅处理（无缓存回退、无默认页面）

### 恢复状态总结

| 资源 | 恢复状态 | 备注 |
|---|---|---|
| Catalog Pod | 已恢复 | Action 结束后 ~10 秒内完全恢复，响应时间恢复至 2-3ms |
| RDS MySQL | 已恢复 | 实例未受直接影响，连接在故障注入结束后立即恢复 |
| UI | 已恢复 | 随 catalog 恢复同步恢复正常 |

### 需关注问题

#### 1. Catalog 错误处理：数据库错误返回 404 而非 500
- **问题:** 当数据库连接超时时，catalog 服务返回 HTTP 404（Not Found）而非 500（Internal Server Error）
- **建议:** 修改 `repository.go` 中的错误处理逻辑，区分 "记录不存在" 和 "数据库不可达" 场景，分别返回 404 和 503

#### 2. UI 缺乏服务降级机制
- **问题:** catalog 不可用时，UI 完全无法渲染页面，抛出大量级联异常
- **建议:** 实现 catalog 服务的断路器（Circuit Breaker）模式，在下游不可用时返回缓存数据或降级页面

#### 3. 未配置 Stop Condition 告警
- **问题:** 实验未设置任何 Stop Condition，如果故障影响超出预期无法自动终止
- **建议:** 为生产环境实验配置 CloudWatch Alarm 作为 Stop Condition（如错误率、延迟 P99）

