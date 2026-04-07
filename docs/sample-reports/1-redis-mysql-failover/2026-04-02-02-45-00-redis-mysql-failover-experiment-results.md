## FIS 实验结果

**Experiment ID:** EXPNvxxxq3x
**Template ID:**   EXT44xxxqeg
**Stack:**         fis-redis-mysql-failover-xxx
**Status:**        completed
**Start Time:**    2026-04-02T02:33:55+00:00
**End Time:**      2026-04-02T02:44:08+00:00
**Duration:**      10 分 13 秒

### Action Results

| Action | Action ID | Status | Start (UTC) | End (UTC) | Duration |
|---|---|---|---|---|---|
| InterruptElastiCacheAZPower | aws:elasticache:replicationgroup-interrupt-az-power | completed | 02:34:07 | 02:44:08 | 10 分 01 秒 |
| RebootRDSWithFailover | aws:rds:reboot-db-instances | completed | 02:34:07 | 02:34:08 | ~1 秒 (触发) |

### Stop Condition Alarms

| Alarm | Final Status |
|---|---|
| (无停止条件) | N/A |

### Per-Service Impact Analysis

#### ElastiCache Redis (demo-redis)

| Time (UTC) | Event | Observation |
|---|---|---|
| 02:34:07 | FIS 触发 AZ 电源中断 | InterruptElastiCacheAZPower action 启动 |
| 02:34:07 | Redis 复制组进入 modifying 状态 | us-west-2a 节点断电 |
| 02:34:07 - 02:44:08 | 电源中断持续 10 分钟 | 复制组持续处于 modifying 状态 |
| 02:44:08 | FIS action 完成 | 电源恢复，节点开始恢复 |
| 02:44:56 | 复制组仍在 modifying | 节点恢复中，2 个分片均 available |

**Key Findings:**
- checkout 应用在整个 Redis AZ 电源中断期间 **零错误**，证明 Redis 集群模式的跨 AZ 副本切换对应用完全透明
- Redis 集群配置 (2 分片, 每分片 3 节点) 在单 AZ 故障下具备良好的高可用性
- 应用使用集群模式 (`RETAIL_CHECKOUT_PERSISTENCE_REDIS_CLUSTER=true`)，客户端能自动重定向到健康节点

#### RDS MySQL (demo-mysql)

| Time (UTC) | Event | Observation |
|---|---|---|
| 02:34:07 | FIS 触发 RDS reboot with failover | RebootRDSWithFailover action 启动 |
| 02:34:13 | Multi-AZ failover 开始 | RDS 事件: "Multi-AZ instance failover started" |
| 02:34:14 | catalog 第一个 SQL 查询开始超时 | SELECT 查询开始挂起 (10s timeout) |
| 02:34:23 | 2 个 SQL 查询 10s 超时 | `SELECT * FROM products` 和 `SELECT * FROM tags` 返回 10s 超时 |
| 02:34:23 | UI 首次 500 错误 | SocketTimeoutException 级联到前端 |
| 02:34:26 | RDS 实例重启 (新 primary) | RDS 事件: "DB instance restarted" |
| 02:34:28-29 | catalog 连接错误爆发 | 4 次 `dial tcp 10.x.x.178:3306: i/o timeout` (5s 超时) |
| 02:34:31 | 旧 primary 重启为 standby | RDS 事件: "DB instance restarted" |
| 02:34:33 | 最后一个连接错误 | `dial tcp: i/o timeout` |
| 02:34:34 | catalog 恢复正常 | 首个成功 200 响应 (1.2ms 延迟) |
| 02:35:03 | RDS failover 完成 | RDS 事件: "Multi-AZ instance failover completed" |

**Key Findings:**
- RDS Multi-AZ failover 从触发到应用恢复总计约 **27 秒** (02:34:07 → 02:34:34)
- 应用层实际中断时间约 **10 秒** (02:34:23 首次超时 → 02:34:34 恢复)
- catalog 应用的 Go MySQL 客户端连接超时设置为 5-10 秒，导致排队请求在超时后快速失败
- DNS 切换后应用自动重连到新 primary，无需人工干预
- RDS 已从 us-west-2a 切换到 us-west-2b，后续又切换回 us-west-2a

#### UI 前端 (间接受影响)

| Time (UTC) | Event | Observation |
|---|---|---|
| 02:34:23 | 首次 500 错误 | `GET /catalog` → SocketTimeoutException (catalog 后端超时) |
| 02:34:28-33 | 500 错误集中爆发 | 7 个 500 错误：包括 `/catalog`, `/`, `/cart` 端点 |
| 02:34:34 | 恢复正常 | catalog 恢复后 UI 请求正常 |

**Key Findings:**
- UI 的 500 错误完全由 catalog 后端的 MySQL 连接超时级联导致
- 受影响端点: `/catalog` (商品列表)、`/catalog/{id}` (商品详情)、`/` (首页)、`/cart` (购物车)
- 实验期间共 7 个 500 错误，影响范围有限

### Recovery Status Summary

| Resource | Recovery Status | Notes |
|---|---|---|
| ElastiCache Redis (demo-redis) | Recovering | 复制组仍在 modifying，分片节点 available，预计数分钟内完全恢复 |
| RDS MySQL (demo-mysql) | Recovered | available 状态，已切换回 us-west-2a，Multi-AZ 保持启用 |
| checkout (应用) | Recovered | 整个实验期间零错误 |
| catalog (应用) | Recovered | 02:34:34 恢复正常服务，延迟恢复到 ms 级 |
| ui (应用) | Recovered | 随 catalog 恢复而恢复 |

### Issues Requiring Attention

#### 1. catalog 应用缺乏数据库连接池重连机制
- **Problem:** catalog 在 RDS failover 期间产生了 5 次 TCP 连接超时错误，说明应用持有到旧 primary 的 stale 连接
- **Recommendation:** 配置 Go SQL 驱动的 `MaxIdleConns`、`MaxOpenConns` 和 `ConnMaxLifetime` 参数，确保连接池能快速检测和丢弃失效连接

#### 2. UI 未实现 catalog 服务的降级策略
- **Problem:** catalog 后端超时直接导致 UI 返回 500 错误，用户体验完全中断
- **Recommendation:** 在 UI 层实现 circuit breaker 模式或缓存降级：当 catalog 不可用时显示缓存的商品数据或友好的错误页面

#### 3. ElastiCache 恢复时间较长
- **Problem:** FIS action 在 02:44:08 完成，但复制组在实验结束后仍处于 modifying 状态
- **Recommendation:** 这是 ElastiCache 的正常恢复行为，但建议监控恢复完成时间并设置 CloudWatch 告警
