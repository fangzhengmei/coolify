# Sentinel Watchdog 协作机制全解

本文档追踪 Coolify 中「远程守护进程检查」「状态轮询」「自动启动」「异常恢复」四个段的协作方式，厘清 Sentinel 守护组件状态与服务器健康记录的同步逻辑，并剖析服务器掉线期间排队任务堆积的完整处理路径。

---

## 一、四大组件概览与入口调度

所有定时检查的唯一入口是 [ServerManagerJob](file:///d:/fz/0601-1/solo-dogfeeding/code/98-coolify/app/Jobs/ServerManagerJob.php#L18-L208)，它由 [Console Kernel](file:///d:/fz/0601-1/solo-dogfeeding/code/98-coolify/app/Console/Kernel.php#L59) 每分钟调度一次 (`everyMinute()->onOneServer()`)。

`ServerManagerJob::handle()` 做两件事：

1. **`dispatchConnectionChecks()`** — 连接检查（远程守护进程检查）
2. **`processScheduledTasks()`** — 状态轮询 + 自动启动 + 异常恢复

下面按这四个段逐一拆解。

#### Console Kernel dev 分支与生产分支的差异

[Console Kernel](file:///d:/fz/0601-1/solo-dogfeeding/code/98-coolify/app/Console/Kernel.php#L32-L94) 的 `schedule()` 方法用 `if (isDev())` 分了两套调度配置，差异很大：

| 调度项 | Dev 分支 | Production 分支 |
|--------|---------|----------------|
| ServerManagerJob | everyMinute | everyMinute（两边都有） |
| horizon:snapshot | everyMinute | everyFiveMinutes |
| CleanupInstanceStuffsJob | everyMinute | everyTwoMinutes |
| CheckHelperImageJob | everyTenMinutes | 跟随 update_check_frequency |
| ScheduledJobManager | everyMinute | everyMinute（两边都有） |
| uploads:clear | everyTwoMinutes | everyTwoMinutes（两边都有） |
| cleanup:unreachable-servers | ❌ 没有 | daily（每日） |
| cleanup:database --yes | ❌ 没有 | daily（每日） |
| PullTemplatesFromCDN | ❌ 没有 | 跟随 update_check_frequency |
| PullChangelog | ❌ 没有 | 跟随 update_check_frequency |
| CheckForUpdatesJob | ❌ 没有 | 跟随 update_check_frequency |
| UpdateCoolifyJob | ❌ 没有 | 跟随 is_auto_update_enabled |
| RegenerateSslCertJob | ❌ 没有 | twiceDaily |
| CheckTraefikVersionJob | ❌ 没有 | 每周日零点 |
| CleanupOrphanedPreviewContainersJob | ❌ 没有 | daily |
| cleanup:redis --clear-locks | ❌ 没有 | daily |
| sanctum:prune-expired | ❌ 没有 | hourly |
| ApiTokenExpirationWarningJob | ❌ 没有 | hourly |
| cleanup:ssh-mux | hourly（条件触发） | hourly（条件触发） |

**dev 环境缺失的关键清理任务**：
- `cleanup:unreachable-servers` — 7 天不可达服务器自动禁用
- `cleanup:database --yes` — 数据库清理
- `CleanupOrphanedPreviewContainersJob` — 孤立预览容器清理
- `cleanup:redis --clear-locks` — Redis 锁清理

**设计意图**：开发环境下不跑这些清理任务，避免开发时数据被清掉、服务器被误禁用。同时开发环境更新检查更频繁（每分钟），方便调试。

**两边都有的 ServerManagerJob 差异**：
虽然两边都是 everyMinute，但 ServerManagerJob 内部的 `checkFrequency` 会根据 `isCloud()` 变化——但 dev 环境通常不是 cloud，所以都是 1 分钟周期。真正的差异在 Kernel 层的其他调度项。

---

## 二、段 1：远程守护进程检查 — ServerConnectionCheckJob

### 调度条件

[ServerManagerJob::dispatchConnectionChecks()](file:///d:/fz/0601-1/solo-dogfeeding/code/98-coolify/app/Jobs/ServerManagerJob.php#L79-L101) 对每个服务器做如下判断：

```
如果 Sentinel 已启用 && Sentinel 心跳存活 (isSentinelLive) → 跳过 SSH 连接检查
否则 → 检查是否因退避策略应跳过 → 不跳过则 dispatch ServerConnectionCheckJob
```

关键逻辑：**Sentinel 心跳存活即视为服务器连通的证明，无需再 SSH 探测**。这是一条核心的优化——避免对已确认在线的服务器浪费 SSH 连接。

### 执行流程

[ServerConnectionCheckJob](file:///d:/fz/0601-1/solo-dogfeeding/code/98-coolify/app/Jobs/ServerConnectionCheckJob.php#L21-L247) 的完整路径：

1. 检查 `force_disabled` → 直接标记不可达
2. 检查 Hetzner 云服务器状态 → 如果 off 则直接抛异常标记不可达
3. SSH 连接检查 (`ls -la /`) → 失败则标记 `is_reachable=false, is_usable=false`
4. SSH 可达则检查 Docker 可用性 (`docker version --format json`) → 设置 `is_usable`
5. 更新健康记录并触发可达性变更事件

### 健康记录同步

连接检查的结果直接写入 `server_settings` 表的两个关键字段：

| 字段 | 含义 | 写入时机 |
|------|------|----------|
| `is_reachable` | SSH 是否可达 | 连接检查成功/失败 |
| `is_usable` | Docker 是否可用 | 连接可达后检查 Docker |

同时操作 `servers` 表的 `unreachable_count` 字段：
- 不可达时 `increment('unreachable_count')`
- 恢复可达时 `update(['unreachable_count' => 0])`

### 退避策略

[ServerManagerJob::shouldSkipDueToBackoff()](file:///d:/fz/0601-1/solo-dogfeeding/code/98-coolify/app/Jobs/ServerManagerJob.php#L193-L207) 根据 `unreachable_count` 决定检查频率：

| unreachable_count | 检查间隔（周期倍数） | 实际间隔（自建 1min/云 5min） |
|--------------------|---------------------|-------------------------------|
| 0-2 | 每周期 | 1min / 5min |
| 3-5 | 每 3 周期 | ~3min / ~15min |
| 6-11 | 每 6 周期 | ~6min / ~30min |
| 12+ | 每 12 周期 | ~12min / ~60min |

退避使用 `crc32(server_id)` 哈希分散检查，避免惊群效应。

#### 退避分粒度时序：自建 1min vs 云 5min 的两套周期

退避不是以绝对时间计算的，而是以**调度周期倍数**为单位。周期长度由环境决定：

- **自建环境**：`checkFrequency = '* * * * *'` → 每 1 分钟一个周期
- **云环境**：`checkFrequency = '*/5 * * * *'` → 每 5 分钟一个周期

[ServerManagerJob::shouldSkipDueToBackoff()](file:///d:/fz/0601-1/solo-dogfeeding/code/98-coolify/app/Jobs/ServerManagerJob.php#L193-L207) 的计算方式：

```php
$cyclePeriodMinutes = isCloud() ? 5 : 1;
$cycleIndex = intdiv($this->executionTime->minute, $cyclePeriodMinutes);
$serverHash = abs(crc32((string)$server->id));
return ($cycleIndex + $serverHash) % $interval !== 0;
```

**为什么用周期索引 + 哈希取模**：
- `cycleIndex` 是当前分钟属于第几个周期（云环境每 5 分钟 = 1 个周期，第 0 分钟周期 0，第 5 分钟周期 1 …）
- `serverHash` 是服务器 ID 的 CRC32 哈希，每个服务器分到一个固定的偏移量
- `(cycleIndex + serverHash) % interval !== 0` 表示当前周期轮不到这台服务器

**实际检查间隔对照表**：

| unreachable_count | 间隔（周期倍数） | 自建环境（1min/周期） | 云环境（5min/周期） |
|--------------------|----------------|----------------------|---------------------|
| 0-2 | 1 周期 | ~1 分钟 | ~5 分钟 |
| 3-5 | 3 周期 | ~3 分钟 | ~15 分钟 |
| 6-11 | 6 周期 | ~6 分钟 | ~30 分钟 |
| 12+ | 12 周期 | ~12 分钟 | ~60 分钟 |

**设计意图**：
- 自建环境服务器少，检查频率高（每分钟）
- 云环境服务器多，检查频率低（每 5 分钟），减少 SSH 连接风暴
- 哈希分散确保同一时刻不会所有服务器同时检查，负载平均分布
- 退避用周期倍数而非绝对时间，两套环境复用同一套退避逻辑

### 可达性变更事件

[ServerReachabilityChanged](file:///d:/fz/0601-1/solo-dogfeeding/code/98-coolify/app/Events/ServerReachabilityChanged.php#L8-L17) 的唯一功能是在构造函数中直接调用 `server->isReachableChanged()`，由 [Server::isReachableChanged()](file:///d:/fz/0601-1/solo-dogfeeding/code/98-coolify/app/Models/Server.php#L1235-L1252) 执行实际的通知逻辑：

```php
// isReachableChanged() 内部逻辑：
if ($isReachable === true && $unreachableNotificationSent) {
    $this->sendReachableNotification();  // 发恢复通知 + unreachable_notification_sent = false + save();
}
if (!$isReachable && $this->unreachable_count >= 2 && !$unreachableNotificationSent) {
    $this->sendUnreachableNotification(); // 发不可达通知 + unreachable_notification_sent = true + save();
}
```

阈值为 2：单次抖动不触发通知，连续 2 次不可达才发。

#### 构造即副作用：反 Laravel 惯例的设计

**Laravel 对 Event 类的惯例是：** Event 应该是纯数据载体（DTO），只承载事件相关的属性，不该有任何副作用。副作用（写数据库、发通知等）应该放在 Listener 中执行。Event 本身只是一个「信号」。

**这里的实现完全打破了这一惯例：** 构造函数里直接调用 `isReachableChanged()`，把「判定是否需要通知 + 更新通知标记 + 写数据库 + 发送通知」全部做了。这个 Event 没有任何 Listener，构造即执行。

**为什么这么设计（可能的权衡）：**
- 简单直接，不需要注册 Listener，少一个文件
- 判断逻辑封装在模型方法 `isReachableChanged()` 里，Event 只是一个触发壳
- 避免了 Listener 注册和发现的样板代码

**代价（这个设计的真实问题）：**

**代价 1：4 个 dispatch 点的副作用全部集中在构造函数中**

以下 4 个位置调用 `ServerReachabilityChanged::dispatch($server)` 后，构造函数立即执行 `isReachableChanged()`，副作用在 dispatch 的同一瞬间就完成了：

1. [Server::validateConnection()](file:///d:/fz/0601-1/solo-dogfeeding/code/98-coolify/app/Models/Server.php#L1282-L1293) — SSH 可达性检测结果变化时
2. [Team::forceDisableServer()](file:///d:/fz/0601-1/solo-dogfeeding/code/98-coolify/app/Models/Team.php#L238) — 手动强制禁用服务器
3. [Livewire Server\Show](file:///d:/fz/0601-1/solo-dogfeeding/code/98-coolify/app/Livewire/Server/Show.php#L309) — 前端手动验证服务器连通性
4. [ValidateAndInstallServerJob](file:///d:/fz/0601-1/solo-dogfeeding/code/98-coolify/app/Jobs/ValidateAndInstallServerJob.php#L191) — 服务器安装验证

无论从哪个入口 dispatch，副作用都在构造函数里完成，后续没有任何 Listener 能改变这一行为。

**代价 2：添加 Listener 无效**

假设开发者想在可达性变化时做额外的事（比如写审计日志、触发 Webhook、同步到监控系统），按 Laravel 惯例应该加一个 Listener。但加了也没用——副作用在 dispatch 时构造函数里已经执行完了，Listener 拿到的是一个空壳 Event。即便 Listener 再调一次 `isReachableChanged()`，也只会再执行一遍副作用。

更关键的是：Laravel 支持 `ShouldQueue` 异步 Listener，但这个 Event 的副作用在构造函数同步执行，异步 Listener 的延迟语义完全失效。

**代价 3：可扩展性差**

如果将来想把通知拆成多个独立阶段（比如可达性变化 → 触发审计日志 → 触发监控告警 → 触发业务通知），完全做不到。Event 的执行路径被锁死在「构造函数 → isReachableChanged()」这一条单链上。

**代价 4：单测隔离困难**

正常 Laravel Event 可以用 `Event::fake()` 来阻止真实 Listener 执行，只断言 Event 是否被 dispatch。但这个 Event 根本没有 Listener，`Event::fake()` 没用——构造函数里的副作用照样执行，照样写数据库、照样发通知。想测「dispatch 了这个 Event」就必须接受所有副作用。

想单独测试「某个 dispatch 点是否正确触发了事件」也做不到——因为测试里必须 stub 整个通知系统：必须用 `Notification::fake()` 才能阻止真实通知，必须 mock Server 模型才能阻止数据库写入，测试 setup 极其笨重。

**总结：**

类名 `ServerReachabilityChanged` 叫 Event 名不副实。它更像一个「动作类」（Action）而不是「事件」（Event）。dispatch 它等于直接执行一个匿名函数，等于调用一个方法，而不是广播一个信号。

---

## 三、段 2：状态轮询 — ServerCheckJob + PushServerUpdateJob

状态轮询有**两条平行实现**，各自覆盖不同的服务器类型和运行场景。

### 两条路径的关系

| | SSH 主动轮询 (ServerCheckJob) | Sentinel 推送轮询 (PushServerUpdateJob) |
|---|---|---|
| 触发方式 | ServerManagerJob 定时派发 | Sentinel 容器 HTTP POST 回调 |
| 适用服务器 | 所有非 Swarm 非 Build 服务器 | 所有非 Swarm 非 Build 服务器 |
| Swarm 支持 | ✅ 支持 (`isSwarmWorker` 跳过, `isSwarmManager` 正常) | ❌ 不支持 (PushServerUpdateJob 注释: `TODO: Swarm is not supported yet`) |
| 数据源 | `docker container inspect` (SSH) | Sentinel 容器内采集 |
| 生效条件 | `sentinelOutOfSync == true` | `sentinelOutOfSync == false` (心跳正常) |
| 容器状态更新 | `GetContainersStatus::run()` | 直接在 Job 内遍历容器列表 |

**核心区别**：这两条路径是互斥的——当 Sentinel 心跳正常时走推送路径，心跳失同步时退回 SSH 路径。Swarm 服务器只能走 SSH 路径，因为 `PushServerUpdateJob` 尚不支持 Swarm。

### 路径 A：SSH 主动轮询 — ServerCheckJob

[ServerManagerJob::processServerTasks()](file:///d:/fz/0601-1/solo-dogfeeding/code/98-coolify/app/Jobs/ServerManagerJob.php#L119-L172) 中的判断：

```php
$sentinelOutOfSync = Carbon::parse($lastSentinelUpdate)->isBefore(
    $this->executionTime->copy()->subSeconds($waitTime)
);

if ($sentinelOutOfSync) {
    ServerCheckJob::dispatch($server);  // Sentinel 失同步 → 走 SSH
}
```

`waitTime` 由 [Server::waitBeforeDoingSshCheck()](file:///d:/fz/0601-1/solo-dogfeeding/code/98-coolify/app/Models/Server.php#L689-L697) 计算：`sentinel_push_interval_seconds × 3`，最低 120 秒。

即：如果 Sentinel 心跳超过 3 个推送周期未更新（至少 2 分钟），系统认为 Sentinel 失同步，需要通过 SSH 做完整的状态轮询。

[ServerCheckJob](file:///d:/fz/0601-1/solo-dogfeeding/code/98-coolify/app/Jobs/ServerCheckJob.php#L21-L118) 执行：

1. 调用 `server->serverStatus()` → SSH 检查可达性 + 功能性
2. 如果服务器不在线 → 直接返回（不更新容器状态）
3. 非 Swarm Worker 也非 Build Server 时，获取容器列表 (`docker container inspect`)
4. 调用 `GetContainersStatus::run()` 同步容器状态到数据库
5. 如果 Sentinel 已启用 (`isSentinelEnabled()`) → 派发 `CheckAndStartSentinelJob`
6. 检查 Log Drain 容器
7. 检查 Proxy 容器 → 不存在则自动启动

### 路径 B：Sentinel 推送轮询 — PushServerUpdateJob

当 Sentinel 正常运行时，它以固定间隔（默认 60 秒）向 Coolify 的 API 端点推送数据。

[SentinelController::push()](file:///d:/fz/0601-1/solo-dogfeeding/code/98-coolify/app/Http/Controllers/Api/SentinelController.php#L24-L108) 处理流程：

1. **Token 验证** → 解密 Bearer Token → 提取 `server_uuid` → 匹配数据库中的服务器
2. **云端未付费检查** → 云环境下，如果团队未付费 (`stripe_invoice_paid === false`) 且非内部团队 (`team_id !== 0`)，返回 401 Unauthorized
3. **功能性检查** → `server->isFunctional()` 必须为 true，否则返回 401
4. **Token 一致性** → 请求 token 必须与数据库存储的 `sentinel_token` 一致
5. **心跳更新** → `server->sentinelHeartbeat()` — **每次推送都更新 `sentinel_updated_at`**
6. **去重判断** → `shouldDispatchUpdate()` 决定是否派发 `PushServerUpdateJob`

去重机制 ([shouldDispatchUpdate](file:///d:/fz/0601-1/solo-dogfeeding/code/98-coolify/app/Http/Controllers/Api/SentinelController.php#L116-L141))：

- 计算容器状态 hash（仅 `name` + `state`，不含 metrics 和 health_status）
- 首次推送 / hash 变化 / 强制窗口过期（默认 300 秒）→ 派发
- hash 未变 + 强制窗口未过期 → 跳过（避免每分钟都做重量级数据库操作）

#### auditLog 记录路径

Sentinel push 的成功和失败都有审计日志，走不同的 log channel。

**成功日志**：`auditLog('sentinel.metrics_pushed', ...)` — 走 `audit` channel，级别 info
- 位置：[SentinelController::push() 末尾](file:///d:/fz/0601-1/solo-dogfeeding/code/98-coolify/app/Http/Controllers/Api/SentinelController.php#L102-L105)
- 记录内容：`event`, `ip`, `ua`, `user_id`, `user_email`, `team_id`, `token_id`, `token_name`, `method`, `path`, `server_uuid`, `team_id`

**失败日志**：`auditLogWebhookFailure('sentinel', $reason, ...)` — 走 `audit` channel，级别 warning
- 位置：每个 401/404 错误分支都有
- 失败原因包括：`token_missing`, `decrypt_failed`, `invalid_token_payload`, `server_not_found`, `subscription_unpaid`, `server_not_functional`, `token_mismatch`
- 每次失败都记录 `reason` + `ip` + `ua` + `method` + `path` + 上下文

**auditLog 实现**（[audit.php](file:///d:/fz/0601-1/solo-dogfeeding/code/98-coolify/bootstrap/helpers/audit.php#L16-L46)）：
- 通过 `Log::channel('audit')` 写入独立的审计日志通道
- 失败安全：审计日志本身出错不会影响主请求流程，catch 后用 warning 兜底
- 自动附加：IP、UA、用户 ID、token 信息、HTTP 方法、路径等上下文
- webhook 失败用 `warning` 级别，正常事件用 `info` 级别

#### shouldDispatchUpdate 的 Cache::lock 分布式锁细节

`shouldDispatchUpdate` 不是简单地读缓存比 hash，而是用 `Cache::lock($lockKey, 10)->block(5, ...)` 加分布式锁保护整个「读 hash → 判定 → 写缓存」的 read-modify-write 序列。

**为什么需要锁**：Sentinel 推送间隔很短（默认 60s），在多实例部署（Horizon 多 worker）下，同一个服务器的两次推送可能在不同 web 进程中几乎同时到达。如果不加锁，两个并发请求会同时读到旧 hash，都判定需要 dispatch，产生「同时派发两个 PushServerUpdateJob，浪费数据库资源，甚至可能产生重复的状态更新竞争。

**锁的参数**：
- 持有时间 10 秒：足够完成缓存读写（纯内存操作，实际 <1ms），留 10 秒是为了应对 Redis 延迟等异常
- 阻塞等待 5 秒：超过 5 秒还拿不到锁 → 捕获 `LockTimeoutException` → 返回 `false` → 这次推送直接跳过不派发
- 跳过策略：拿不到锁就放弃，不重试。因为 Sentinel 每分钟推送一次，漏一次不影响，下次推送自然会补上。宁可少派一次，绝不多派一次。

#### 两套 lock/节流的差异：Cache::lock vs shouldRunCronNow

系统中有两种完全不同的「去重/节流」机制，分别用在不同的场景，容易混淆：

| 维度 | Cache::lock（Sentinel push 去重） | shouldRunCronNow（ServerManager 调度节流） |
|------|----------------------------------|--------------------------------------------|
| 位置 | [SentinelController::shouldDispatchUpdate()](file:///d:/fz/0601-1/solo-dogfeeding/code/98-coolify/app/Http/Controllers/Api/SentinelController.php#L116-L141) | [shared.php](file:///d:/fz/0601-1/solo-dogfeeding/code/98-coolify/bootstrap/helpers/shared.php#L675-L696) |
| 本质 | 互斥锁（Mutex）——同一时刻只能一个进 | 周期性调度去重——同一 cron 周期内只触发一次 |
| 解决的问题 | 并发请求同时读写同一缓存导致重复 dispatch | 同一 cron 周期内多次调用导致重复 dispatch |
| 粒度 | 每服务器 1 把锁 | 每服务器每任务 1 个 dedup key（如 `server-check:123`） |
| 超时/持有时间 | lock 10s，block 等待 5s | 30 天 TTL（远超所有周期），存 ISO8601 时间戳 |
| 判定逻辑 | 拿不到锁 → 跳过（返回 false） | 上次执行时间 < 当前周期的 due 时间 → 触发 |
| 失败策略 | 超时就放弃，绝不重复派发 | 永远写入缓存，seed/refresh 都做，确保不漏 |
| 数据一致性 | 强一致（同一时刻只有一个执行者） | 最终一致（同一周期内只触发一次） |
| 典型场景 | HTTP 请求并发推送，同一秒多个请求 | 每分钟定时 Job，同一分钟内多次被调用 |

**为什么需要两套**：
- `Cache::lock` 解决**并发竞争**——同一个服务器的两次推送同时到达，争用同一个缓存键
- `shouldRunCronNow` 解决**周期去重**——同一个 cron 周期内（如同一个小时、同一天），同一项任务只触发一次，不管 ServerManagerJob 被调用了多少次

**调用方式的差异**：
```php
// Cache::lock —— 互斥，保护临界区
Cache::lock($key, 10)->block(5, function () {
    // 临界区代码：读 → 判断 → 写
});

// shouldRunCronNow —— 幂等，同一周期内多次调用只返回一次 true
shouldRunCronNow('0 0 * * *', $timezone, "task:{$server->id}", $executionTime);
```

**在 ServerManagerJob 中的实际应用**：
- `dispatchConnectionChecks`：全局共用一个 dedup key `server-connection-checks`，所有服务器一起判断
- `processServerTasks`：每个任务每个服务器一个 dedup key，如 `server-check:{id}`, `sentinel-restart:{id}`, `server-storage-check:{id}`, `server-patch-check:{id}`
- `sentinelOutOfSync` 是额外的前置判断——Sentinel 心跳正常的服务器，连 shouldRunCronNow 都不用调，直接跳过

#### Force Window 的真实意图

强制窗口（force window / push_force_interval_seconds，默认 300 秒）有两个 Cache key：

- `sentinel:push-hash:{server_id}` → 存上次派发过的状态 hash，TTL 1 天
- `sentinel:push-force:{server_id}` → 存一个布尔标记，TTL = 强制窗口时长

**判定逻辑**：
```php
$shouldDispatch = $cachedHash === null    // 首次推送
    || $cachedHash !== $hash             // 状态真的变了
    || !$forceActive;                   // 强制窗口过期了
```

**Force window 存在的真实意图**不是「定期刷新状态数据」——那是附带效果，而是**防漏检的安全网**。因为 hash 只比较 `name + state`，但 PushServerUpdateJob 里实际处理的内容远不止 state（还有健康状态、磁盘使用率、Proxy 连接网络等）。如果只靠 hash 变化触发，有两类变化会永远被漏掉：

1. **容器状态没变但健康状态在变（health_status 不进 hash）
2. **非容器的变化（磁盘使用率波动、Proxy 网络重连等）

强制窗口保证：即使状态 hash 完全没变，最多 5 分钟也会强制派发一次，确保那些没被 hash 捕捉到的变化也能最终同步到数据库。这是「eventual consistency via periodic refresh 的典型设计。

**两个 key 不同的 TTL 设计**：
- hash key TTL 1 天：服务器停推 1 天后缓存才过期，因为 1 天内恢复推送如果状态不变也不浪费。1 天是上限兜底，防止服务器永久失联后缓存永久占用内存。
- force key TTL = 窗口时长（300s）：每隔窗口过期一次，过期后下一次推送必然触发强制刷新。

**派发后的写入顺序**：先写 hash → 再写 force key（重置窗口计时起点。这样保证每次 dispatch 后，force 窗口从最近一次 dispatch 开始重新计时，而不是从任意时间点。

#### 容器 status 写入 state:health 拼接与 hash 只取 state 的语义差

Sentinel 推送的每个容器有两个独立但相关的字段：`state`（容器生命周期状态）和 `health_status`（健康检查状态）。在写入数据库和计算 hash 时，这两个字段的处理方式截然不同。

**写入数据库：state:health 拼接**

[PushServerUpdateJob.php#L222-L227](file:///d:/fz/0601-1/solo-dogfeeding/code/98-coolify/app/Jobs/PushServerUpdateJob.php#L222-L227)：

```php
$rawHealthStatus = data_get($container, 'health_status');
$containerHealth = $rawHealthStatus ?? 'unknown';
// Only append health status if container is not exited
if ($containerStatus !== 'exited') {
    $containerStatus = "$containerStatus:$containerHealth";
}
```

拼接规则：
- 容器 `exited` 时 → status 就是 `"exited"`（不加 health，因为死了不需要健康检查）
- 容器非 exited 时 → status 是 `"running:healthy"` 或 `"running:unhealthy"` 或 `"running:starting"` 或 `"running:unknown"`

这个拼接后的字符串被直接写入 `applications.status`、`databases.status`、`services.applications.status` 等字段，作为前端展示和业务判定的最终状态。

**计算 hash：只取 state**

[containerStateHash()](file:///d:/fz/0601-1/solo-dogfeeding/code/98-coolify/app/Http/Controllers/Api/SentinelController.php#L154-L166) 只取 `name + state`，完全忽略 `health_status`。

**为什么要有这个语义差？核心原因有三个：**

1. **health_status 波动太频繁**：健康检查可以是每 10 秒甚至更短间隔，状态可能在 `starting` → `healthy` → `unhealthy` → `healthy` 之间来回跳。如果 hash 包含 health_status，那每次健康检查状态变化都会触发 PushServerUpdateJob，每分钟可能产生多次 dispatch，完全失去去重意义。

2. **业务优先级不同**：`state` 代表容器生命周期（created / running / paused / exited / dead），是**容器存在性**的核心指标——容器在不在、跑没跑。`health_status` 是**服务可用性**的附加指标——容器跑着但服务健康不健康。state 变化是「大事」（容器启了/停了），health 变化是「小事」（健康状况波动）。大事要实时响应（hash 变化立即 dispatch），小事可以延后合并（靠 force window 定期刷新）。

3. **exited 的特殊处理**：exited 状态不加 health 后缀，这意味着只要容器退出了，不管之前是什么健康状态，统一表示为 `"exited"`。反过来，如果 hash 包含 health，exited 容器的 health（通常是 none / null）会造成 hash 抖动——容器刚退出时 health 可能还没清，之后又变成 none，导致额外的 dispatch。只取 state 避免了这个问题。

**整体设计哲学**：
- **hash（去重用）** → 粗粒度，只关心「容器有没有、跑没跑」→ 变化少，dispatch 少
- **status（展示用）** → 细粒度，关心「跑着 + 健康不健康」→ 信息全，用户体验好
- **force window** → 兜底机制，确保细粒度的 health 变化最多延迟 5 分钟也能同步到数据库

这是典型的「采样频率和展示精度的权衡」——高频变化的 health 信息靠低频强制刷新来最终一致，而不是实时同步。

[PushServerUpdateJob](file:///d:/fz/0601-1/solo-dogfeeding/code/98-coolify/app/Jobs/PushServerUpdateJob.php#L41-L807) 的核心同步逻辑：

1. 更新磁盘使用率 → 超阈值时派发 `ServerStorageCheckJob`
2. 遍历所有容器，按 `coolify.managed` 标签分类
3. 更新 Application 状态（含多容器聚合）
4. 更新 ApplicationPreview 状态
5. 更新 Database 状态 + TCP Proxy 管理
6. 更新 Service 子资源状态
7. 标记未找到的资源为 `exited`
8. 检查 Proxy 和 Log Drain 容器

#### 磁盘检查：阈值边沿 + 数值变化双重防抖与 Cache::forget 跨阈值重置

PushServerUpdateJob 中磁盘使用率的处理不是简单地「超阈值就发通知」，而是有两层防抖 + 跨阈值重置。

**判定逻辑**（[PushServerUpdateJob.php#L175-L189](file:///d:/fz/0601-1/solo-dogfeeding/code/98-coolify/app/Jobs/PushServerUpdateJob.php#L175-L189)）：

```php
if ($filesystemUsageRoot !== null
    && $filesystemUsageRoot >= $diskThreshold
    && (string)$lastPercentage !== (string)$filesystemUsageRoot) {
    Cache::put($storageCacheKey, $filesystemUsageRoot, 600);
    ServerStorageCheckJob::dispatch($this->server, $filesystemUsageRoot);
} elseif ($filesystemUsageRoot !== null && $filesystemUsageRoot < $diskThreshold) {
    Cache::forget($storageCacheKey);
}
```

**第一层防抖：阈值边沿触发（边沿检测）**
- 只有磁盘使用率 `>= 阈值` 时才可能派发检查任务
- 在阈值以下 (`< $diskThreshold`) 时不仅不派发，还 `Cache::forget($storageCacheKey)` 清除缓存
- 效果：刚超过阈值的那一刻才触发，之后是否再触发取决于第二层防抖

**第二层防抖：数值变化去重（同一边沿内的二次过滤）**
- 同处于阈值以上时，只有百分比数值发生了变化（字符串比较），才会重新派发
- 数值没变 → 不重复派发 → 节省队列资源
- 缓存 TTL 600 秒（10 分钟）：存的是上次派发时的百分比数值

**为什么要用字符串比较 `(string)$lastPercentage !== (string)$filesystemUsageRoot`**：
- 磁盘使用率通常是整数百分比，但也可能是浮点数
- 用严格字符串比较能避免因精度导致的重复触发（如 80.0 vs 80.00）
- 也防止 `null` 和 `0` 的类型转换问题

**Cache::forget 的跨阈值重置语义**：
- 当磁盘从「超阈值」降到「阈值以下」时，立即 forget 缓存
- 下次再超阈值时，因为缓存已清空，`$lastPercentage === null`，立即触发
- 这叫「下降沿重置」——确保每次越过阈值上升沿都能被检测到，不会因为之前超阈值时的缓存而漏掉

**整体效果**：
- 阈值以上：数值每变化一次，最多派发一次 Job，10 分钟内同一数值只发一次
- 阈值以下：清缓存，为下次上升沿做准备
- 不会因为磁盘在阈值线附近 1% 来回波动而疯狂发通知（与 ServerStorageCheckJob 内的 RateLimiter 1 小时 1 次通知配合，形成三级防抖）

**三级防抖链**：
```
Sentinel 推送（每 60s）
  → PushServerUpdateJob 磁盘阈值边沿 + 数值变化去重（10min 窗口）
    → ServerStorageCheckJob RateLimiter（1h 1次通知）
```

每一级都在过滤噪声，最终用户收到高磁盘告警的频率被严格控制在每小时最多 1 次。

#### 超时清 Horizon failed：静默失败的设计

[ServerStorageCheckJob::failed()](file:///d:/fz/0601-1/solo-dogfeeding/code/98-coolify/app/Jobs/ServerStorageCheckJob.php#L32-L43) 有一个特殊的失败处理：

```php
public function failed(?\Throwable $exception): void
{
    if ($exception instanceof \Illuminate\Queue\TimeoutExceededException) {
        Log::warning('ServerStorageCheckJob timed out', [...]);
        // Delete the queue job so it doesn't appear in Horizon's failed list.
        $this->job?->delete();
    }
}
```

**设计意图**：超时是一种「预期内的失败」——服务器 SSH 慢、网络抖动等都可能导致超时。这类失败不应该出现在 Horizon 的 failed jobs 列表里污染视图，也不需要人工介入。

**具体行为**：
- 只有 `TimeoutExceededException`（超时异常）才会触发 delete
- 其他异常（如逻辑错误、配置错误等）仍然正常进入 failed 列表
- 超时失败只写一条 warning 日志到默认 channel
- `$tries = 1` + `timeout = 60`，超时后直接标记失败，不重试
- backoff 1~3 秒，但因为 tries=1，backoff 实际上不会生效

**类似模式的 Job**：
这种「超时即静默删除，不进 Horizon failed」的模式在 Coolify 中不是孤例，是一种针对「网络/IO 类预期内失败」的通用处理方式。

**代价**：
- 开发者如果只看 Horizon failed 列表，可能看不到超时失败的问题
- 需要通过日志监控才能发现超时率上升的趋势
- 属于「用可观测性换用户体验」的权衡——不让用户看到预期内的失败

> **Swarm 不支持**：`PushServerUpdateJob` 开头有 `// TODO: Swarm is not supported yet`，Swarm 集群只能通过 SSH 路径 (ServerCheckJob) 获取容器状态。

### 其他周期任务：ServerPatchCheckJob 周日扫描

ServerManagerJob 的 `processServerTasks` 里还有一个独立的周期性任务——[ServerPatchCheckJob](file:///d:/fz/0601-1/solo-dogfeeding/code/98-coolify/app/Jobs/ServerPatchCheckJob.php#L16-L68)，每周日零点执行。

**调度条件**：
```php
$shouldRunPatchCheck = shouldRunCronNow('0 0 * * 0', $serverTimezone,
    "server-patch-check:{$server->id}", $this->executionTime);
```
- 每周日凌晨 0 点触发
- 每服务器一个 dedup key，确保只触发一次
- **不受 Sentinel 心跳状态影响**——不管 Sentinel 是否在线，每周都跑一次

**任务参数**：
- `$tries = 3` — 失败后重试 3 次（不同于其他 Job 的 tries=1）
- `$timeout = 600` — 10 分钟超时（检查系统更新可能比较慢）
- 中间件：`WithoutOverlapping('server-patch-check-'.$server->uuid)` — 每服务器不重叠执行，600 秒后自动释放锁，`dontRelease()` 表示重叠时不重新入队，直接丢弃

**执行流程**（[ServerPatchCheckJob::handle()](file:///d:/fz/0601-1/solo-dogfeeding/code/98-coolify/app/Jobs/ServerPatchCheckJob.php#L31-L67)）：

1. `$server->serverStatus() === false` → 服务器不在线 → 直接返回，不报错
2. 没 team → 直接返回
3. 调用 `CheckUpdates::run($server)` 检查系统更新
4. 有 error（如检查失败）→ 发通知 + 返回
5. `total_updates > 0` → 发 ServerPatchCheck 通知告知有更新待安装
6. 异常被 catch 住，只记 error 日志，不标记任务失败（`tries=3` 实际上由 Laravel 队列机制处理）

**设计特点**：
- **低优先级、低频**：每周一次，不急
- **容忍失败**：检查失败只发通知，不重试不休眠，服务器离线直接跳过
- **不重叠**：上一周没跑完的话，这周不会重复跑
- **有更新才通知**：系统已是最新版本时不打扰用户

**与 Sentinel 的关系**：
- 完全独立，不依赖 Sentinel 推送
- SSH 路径执行，所以 Swarm 也能跑
- 跟状态轮询、Sentinel 启动都没关系，是纯粹的系统维护任务

---

## 四、段 3：自动启动 — CheckAndStartSentinelJob

### isSentinelEnabled 的真实含义

[Server::isSentinelEnabled()](file:///d:/fz/0601-1/solo-dogfeeding/code/98-coolify/app/Models/Server.php#L704-L707) **不是** `server_settings.is_sentinel_enabled` 的简单读取，而是一个组合判定：

```php
public function isSentinelEnabled()
{
    return ($this->isMetricsEnabled() || $this->isServerApiEnabled()) && ! $this->isBuildServer();
}
```

其中：
- [isMetricsEnabled()](file:///d:/fz/0601-1/solo-dogfeeding/code/98-coolify/app/Models/Server.php#L709-L712) → `settings.is_metrics_enabled`
- [isServerApiEnabled()](file:///d:/fz/0601-1/solo-dogfeeding/code/98-coolify/app/Models/Server.php#L714-L717) → `settings.is_sentinel_enabled`

**两者是 OR 关系**：只要「指标采集」或「Sentinel API」任一启用，并且该服务器不是 Build Server，就视为 Sentinel 已启用。`is_sentinel_enabled` 只是 Sentinel 功能的一个子开关（控制 Server API 能力），不是总开关。

同时，[StartSentinel](file:///d:/fz/0601-1/solo-dogfeeding/code/98-coolify/app/Actions/Server/StartSentinel.php#L15-L17) 在 Swarm 服务器上直接返回，不做任何操作：

```php
if ($server->isSwarm() || $server->isBuildServer()) {
    return;
}
```

### Sentinel 的自动启动有 **三个触发点**：

### 触发点 1：ServerCheckJob 中触发（崩溃恢复）

[ServerCheckJob](file:///d:/fz/0601-1/solo-dogfeeding/code/98-coolify/app/Jobs/ServerCheckJob.php#L66-L68)：

```php
if ($this->server->isSentinelEnabled()) {
    CheckAndStartSentinelJob::dispatch($this->server);
}
```

当 SSH 轮询发现服务器在线但 Sentinel 心跳失同步时，顺带检查 Sentinel 是否在运行。这是**崩溃恢复**的核心路径。

### 触发点 2：ServerManagerJob 中触发（每日版本更新）

[ServerManagerJob::processServerTasks()](file:///d:/fz/0601-1/solo-dogfeeding/code/98-coolify/app/Jobs/ServerManagerJob.php#L142-L147)：

```php
$shouldRestartSentinel = $isSentinelEnabled
    && shouldRunCronNow('0 0 * * *', ...);  // 每天零点
if ($shouldRestartSentinel) {
    CheckAndStartSentinelJob::dispatch($server);
}
```

每天零点检查 Sentinel 是否需要更新版本。

### 触发点 3：配置变更触发（热重载）

见第五节场景 5 的完整链路展开。

### 执行流程

[CheckAndStartSentinelJob](file:///d:/fz/0601-1/solo-dogfeeding/code/98-coolify/app/Jobs/CheckAndStartSentinelJob.php#L14-L51)：

1. `docker inspect coolify-sentinel` → 检查容器是否存在及状态
2. 状态不是 `running` → 调用 `StartSentinel::run(restart: true)` 重新启动
3. 状态是 `running` → 检查版本：
   - 获取远程容器内版本 (`curl http://127.0.0.1:8888/api/version`)
   - 获取最新版本 (`get_latest_sentinel_version()`)
   - 如果远程版本 < 最新版本 → 重启更新
   - 如果两者都是 `0.0.0` → 用 `latest` 标签重启

### StartSentinel 的完整操作

[StartSentinel](file:///d:/fz/0601-1/solo-dogfeeding/code/98-coolify/app/Actions/Server/StartSentinel.php#L9-L70)：

1. Swarm/Build Server → 直接返回，不启动
2. 如果是重启 → 先 `StopSentinel::run()` (docker rm -f + 重置心跳)
3. 获取配置：metrics 历史天数、刷新率、推送间隔、token、端点 URL、debug 模式
4. `ensureValidSentinelToken()` → 确保 token 有效（无效则重新生成）
5. 构建 `docker run` 命令，配置：
   - 健康检查：`curl --fail http://127.0.0.1:8888/api/health`（每 10 秒，3 次失败）
   - 挂载 Docker socket 和数据目录
   - `--pid host` 共享 PID 命名空间
   - `COLLECTOR_ENABLED` 由 `isMetricsEnabled()` 决定（而非 `is_sentinel_enabled`）
6. 执行远程命令：删除旧容器 → 创建目录 → 启动新容器 → 修复权限
7. 更新数据库：`is_sentinel_enabled = true` + `sentinelHeartbeat()`
8. 广播 `SentinelRestarted` 事件（通知前端 UI）

#### Sentinel API 边车自洽点

Sentinel 容器本身是一个独立的边车（sidecar）进程，监听 `127.0.0.1:8888`，提供自己的 HTTP API。它的健康和版本管理都是**自洽**的——不依赖 Coolify 主动探测，靠 Docker 健康检查和自身 API 完成闭环。

**自洽点 1：Docker health check 用自己的 API**

[StartSentinel 的 docker run](file:///d:/fz/0601-1/solo-dogfeeding/code/98-coolify/app/Actions/Server/StartSentinel.php#L54) 配置：

```bash
--health-cmd "curl --fail http://127.0.0.1:8888/api/health || exit 1"
--health-interval 10s
--health-retries 3
```

- Docker 每 10 秒调用一次 Sentinel 自己的 `/api/health` 端点
- 连续 3 次失败 → Docker 标记容器为 `unhealthy`
- 这是容器级别的自监控——Sentinel 自己的健康状况由 Docker 自动检测，不需要 Coolify 参与

**自洽点 2：版本检查用自己的 API**

[CheckAndStartSentinelJob](file:///d:/fz/0601-1/solo-dogfeeding/code/98-coolify/app/Jobs/CheckAndStartSentinelJob.php#L36) 获取运行中版本：

```php
$runningVersion = instant_remote_process_with_timeout(
    ['docker exec coolify-sentinel sh -c "curl http://127.0.0.1:8888/api/version"'],
    $this->server, false
);
```

- 通过 `docker exec` 在容器内 curl 自己的 `/api/version`
- 不暴露端口给外部，全部在容器内部闭环
- 版本号用来和最新版本对比，决定是否需要更新

**自洽点 3：边车只监听本机，不对外暴露**

- Sentinel API 只绑定 `127.0.0.1:8888`，只在容器内可访问
- Coolify 不直接调用 Sentinel API，靠 Sentinel 主动推送数据（push 模式）
- 版本检查用 `docker exec` 绕进去，不映射端口
- 安全：没有额外的端口暴露到公网，减小攻击面

**整体自洽架构**：
```
Coolify 控制面 ←── push ── Sentinel 边车（127.0.0.1:8888）
     │                      │    │
     └── StartSentinel ─────┘    └── Docker health check（自监控）
         CheckAndStartSentinel        /api/health（自检测）
         docker exec curl             /api/version（自报告）
```

Sentinel 作为边车，自己管自己的健康、自己管自己的版本、自己推送数据。Coolify 只负责启动和接收推送，不需要主动轮询边车状态。这是典型的 sidecar 自洽设计。

#### ServerSetting 创建期 token 与 URL 自动生成

Sentinel 的 token 和推送端点 URL 不是在启动时才临时生成的，而是在 ServerSetting 创建时就自动初始化了。

[ServerSetting::booted()](file:///d:/fz/0601-1/solo-dogfeeding/code/98-coolify/app/Models/ServerSetting.php#L117-L130) 的 `creating` 事件：

```php
static::creating(function ($setting) {
    if (str($setting->sentinel_token)->isEmpty()) {
        $setting->generateSentinelToken(save: false, ignoreEvent: true);
    }
    if (str($setting->sentinel_custom_url)->isEmpty()) {
        $setting->generateSentinelUrl(save: false, ignoreEvent: true);
    }
});
```

**token 生成**（[generateSentinelToken()](file:///d:/fz/0601-1/solo-dogfeeding/code/98-coolify/app/Models/ServerSetting.php#L189-L205)）：
- 内容：`encrypt(json_encode(['server_uuid' => $this->server->uuid]))`
- 用 Laravel 的 `encrypt()` 加密，密文由 `base64_encode(iv + tag + payload)` 组成
- 结果是一串 `a-zA-Z0-9+/=` 的 Base64 字符（可能包含 `/` 和 `+` 和 `=`）

**URL 生成**（[generateSentinelUrl()](file:///d:/fz/0601-1/solo-dogfeeding/code/98-coolify/app/Models/ServerSetting.php#L207-L230)），优先级从高到低：
1. localhost 服务器 → `http://host.docker.internal:8000`
2. InstanceSettings 有 fqdn → 用 fqdn
3. 有 public_ipv4 → `http://{ipv4}:8000`
4. 有 public_ipv6 → `http://{ipv6}:8000`

`ignoreEvent: true` 的作用：创建期生成 token/URL 时用 `saveQuietly()` 保存，不触发 `updated` 事件，避免刚创建就触发 restartSentinel——因为这时候服务器上的 Sentinel 容器还根本不存在，重启没有意义。

#### 字符白名单防 shell 注入

`TOKEN` 和 `PUSH_ENDPOINT` 环境变量会通过命令行参数的形式传给 `docker run`，最终拼接到 shell 命令中执行。如果 token 或 URL 中包含特殊字符（如 `;`、`&`、`$`、空格等），可能造成命令注入。

为此有两层防护：

**第一层：token 字符白名单验证**（[isValidSentinelToken()](file:///d:/fz/0601-1/solo-dogfeeding/code/98-coolify/app/Models/ServerSetting.php#L148-L155)）：

```php
public static function isValidSentinelToken(?string $token): bool
{
    if ($token === null) { return false; }
    return (bool)preg_match('/\A[a-zA-Z0-9._\-+=\/]+\z/', $token);
}
```

白名单字符：`a-z`、`A-Z`、`0-9`、`.`、`_`、`-`、`+`、`=`、`/`

这些字符都是** shell 安全**的——没有分号、没有与号、没有美元符、没有空格、没有反引号、没有管道符。即使被拼接到 shell 命令里也不会产生注入。

**为什么是这些字符**：Laravel 的 `encrypt()` 输出是 base64 编码的密文（含 `+`、`/`、`=`），加上可能的前缀格式字符。白名单正好覆盖了加密 token 所有可能出现的字符，多一个都不行。

**第二层：ensureValidSentinelToken 主动修正**（[ensureValidSentinelToken()](file:///d:/fz/0601-1/solo-dogfeeding/code/98-coolify/app/Models/ServerSetting.php#L162-L187)）：

在 StartSentinel 启动前调用，做完整的校验链路：
1. 尝试读取 `sentinel_token`（经过 encrypted cast 解密）
2. 如果解密失败（DecryptException）→ token 无效
3. 用白名单正则验证 token 是否合法
4. 不合法 → 清空 raw attribute → `generateSentinelToken()` 重新生成
5. 重新生成后再验证一次，还不行就抛 RuntimeException
6. `ignoreEvent: true` 防止生成新 token 又触发 restartSentinel 死循环

**为什么需要清空 raw attribute**：
```php
$attrs = $this->getAttributes();
$attrs['sentinel_token'] = null;
$this->setRawAttributes($attrs, true);
```
因为 `sentinel_token` 有 `encrypted` cast，如果直接赋值再保存，Eloquent 的脏检查会读取原值（那个解不开密的坏值），解密失败导致异常。直接改 raw attributes 绕过了 cast 的脏检查。

**整体安全链路**：
```
创建期自动生成 token → 白名单字符集（encrypt 输出天然符合）
                        ↓
启动前 ensureValidSentinelToken → 白名单校验 → 不合法就重生成
                        ↓
docker run -e TOKEN="$token" ... → 拼接进 shell 命令，安全无注入风险
```

---

## 五、段 4：异常恢复 — 完整路径

异常恢复不是一个独立的 Job，而是前三段协作的结果。以下是各种异常场景的恢复路径：

### 场景 1：Sentinel 容器崩溃

```
Sentinel 停止推送
  → sentinel_updated_at 不再更新
  → ServerManagerJob 检测到 sentinelOutOfSync
  → 派发 ServerCheckJob（SSH 轮询）
  → ServerCheckJob 发现服务器在线 + isSentinelEnabled()=true
  → 派发 CheckAndStartSentinelJob
  → 检测到容器不在 running 状态
  → StartSentinel::run(restart: true) 重新启动
```

### 场景 2：服务器整体宕机

```
服务器不响应 SSH
  → ServerConnectionCheckJob 标记 is_reachable=false, is_usable=false
  → unreachable_count++
  → 退避策略生效，逐渐降低检查频率
  → unreachable_count >= 2 → ServerReachabilityChanged → 发送不可达通知
  → 7 天后 cleanup:unreachable-servers → 云环境改 IP / 自建 forceDisable
```

### 场景 3：服务器恢复上线

```
服务器重新响应 SSH
  → ServerConnectionCheckJob 标记 is_reachable=true, is_usable=true
  → unreachable_count 重置为 0
  → ServerReachabilityChanged → 发送恢复通知
  → 下一轮 ServerManagerJob 检测到 sentinelOutOfSync
  → ServerCheckJob 更新容器状态
  → CheckAndStartSentinelJob 恢复 Sentinel
```

### 场景 4：Sentinel 推送正常但 SSH 不通

```
Sentinel 心跳存活 → ServerManagerJob 跳过 SSH 连接检查
  → Sentinel 推送继续处理容器状态
  → 无需 SSH，系统正常运行
```

这是 Sentinel 存在的核心价值——**只要 Sentinel 能推送数据，即使 SSH 连接有问题，系统仍然能正常监控容器状态**。

### 场景 5：配置变更触发 Sentinel 重启（全链路展开）

[ServerSetting::booted()](file:///d:/fz/0601-1/solo-dogfeeding/code/98-coolify/app/Models/ServerSetting.php#L117-L142) 通过 Eloquent `updated` 事件监听以下 5 个字段的变更：

- `sentinel_token`
- `sentinel_custom_url`
- `sentinel_metrics_refresh_rate_seconds`
- `sentinel_metrics_history_days`
- `sentinel_push_interval_seconds`

**完整调用链**：

```
ServerSetting 字段变更 → Eloquent updated 事件触发
  → ServerSetting::booted() 中 wasChanged() 检测
  → $settings->server->restartSentinel()                [Server::restartSentinel()]
      → StartSentinel::dispatch($this, true, ...)       [默认 async=true]
          → StartSentinel::handle($server, restart=true)
              ├── $server->isSwarm() || $server->isBuildServer() → return
              ├── StopSentinel::run($server)             [先停]
              │   ├── docker rm -f coolify-sentinel      [远程执行]
              │   └── $server->sentinelHeartbeat(isReset=true)  [sentinel_updated_at = now()-6000min]
              ├── ensureValidSentinelToken()              [确保 token 有效]
              ├── docker run -d ... coolify-sentinel      [远程启动新容器]
              ├── $server->settings->is_sentinel_enabled = true + save()
              ├── $server->sentinelHeartbeat()            [sentinel_updated_at = now()]
              └── SentinelRestarted::dispatch($server)    [广播 WebSocket 事件]
                    └── PrivateChannel("team.{teamId}")   [前端 Livewire 收到通知]
```

注意：`restartSentinel()` 默认 `async=true`，通过队列异步执行 `StartSentinel`；如果 `async=false`，则同步调用 `StartSentinel::run()`。`StopSentinel::run()` 始终是同步的——先停后启，确保端口不冲突。

---

## 六、守护组件状态与服务器健康记录的同步逻辑

### 核心同步数据流

```
Sentinel 容器（远程服务器上）
    │
    │  每 push_interval_seconds（默认 60s）HTTP POST
    ▼
SentinelController::push()
    │
    ├── Token 缺失/解密失败/负载无效 → 401
    ├── 服务器不存在 → 404
    ├── 云端未付费 (isCloud && !stripe_invoice_paid && team_id !== 0) → 401
    ├── isFunctional() === false → 401
    ├── Token 不匹配 → 401
    ├── 验证 containers 数组 → 422
    │
    ├── server->sentinelHeartbeat()     → 更新 servers.sentinel_updated_at
    │
    └── shouldDispatchUpdate() 判定
         │
         └── PushServerUpdateJob        → 同步容器状态到各资源表
                                          → 同步磁盘使用率
                                          → 管理 Proxy/Log Drain

ServerManagerJob（每分钟）
    │
    ├── isSentinelLive()                → 读取 sentinel_updated_at
    │   └── 跳过/执行 SSH 连接检查
    │
    └── sentinelOutOfSync              → 读取 sentinel_updated_at
        └── 跳过/执行 SSH 状态轮询
```

### 心跳判定逻辑

[Server::isSentinelLive()](file:///d:/fz/0601-1/solo-dogfeeding/code/98-coolify/app/Models/Server.php#L699-L702)：

```php
public function isSentinelLive()
{
    return Carbon::parse($this->sentinel_updated_at)
        ->isAfter(now()->subSeconds($this->waitBeforeDoingSshCheck()));
}
```

[Server::waitBeforeDoingSshCheck()](file:///d:/fz/0601-1/solo-dogfeeding/code/98-coolify/app/Models/Server.php#L689-L697)：

```php
$wait = $this->settings->sentinel_push_interval_seconds * 3;
if ($wait < 120) { $wait = 120; }
return $wait;
```

| 推送间隔 | 等待阈值 | 含义 |
|----------|---------|------|
| 30s | 120s (最低) | 允许 4 次推送失败 |
| 60s | 180s | 允许 3 次推送失败 |
| 120s | 360s | 允许 3 次推送失败 |

### 心跳重置

[Server::sentinelHeartbeat()](file:///d:/fz/0601-1/solo-dogfeeding/code/98-coolify/app/Models/Server.php#L678-L682)：

- 正常心跳：`sentinel_updated_at = now()`
- 重置心跳（StopSentinel 时）：`sentinel_updated_at = now()->subMinutes(6000)` → 确保立即判定为失同步

### isFunctional 与 isSentinelLive 对照

[Server::isFunctional()](file:///d:/fz/0601-1/solo-dogfeeding/code/98-coolify/app/Models/Server.php#L1083-L1092) 和 [Server::isSentinelLive()](file:///d:/fz/0601-1/solo-dogfeeding/code/98-coolify/app/Models/Server.php#L699-L702) 是系统中判定服务器在线状态的两种互补机制：

| 维度 | isFunctional() | isSentinelLive() |
|------|---------------|-----------------|
| 判定依据 | `is_reachable` + `is_usable` + `!force_disabled` + `ip !== '1.2.3.4'` | `sentinel_updated_at` 是否在等待阈值内 |
| 数据来源 | SSH 连接检查 (ServerConnectionCheckJob) 写入的数据库字段 | Sentinel 推送 (SentinelController::push) 更新的时间戳 |
| 检测方式 | 主动探测（Coolify → SSH → 服务器） | 被动接收（服务器 → HTTP POST → Coolify） |
| 副作用 | 返回 false 时删除 `ssh-mux` 文件（`Storage::disk('ssh-mux')->delete(muxFilename())`） | 无副作用，纯读操作 |
| 使用场景 | ApplicationDeploymentJob 是否可执行、Sentinel push 是否接受、资源操作前置检查 | ServerManagerJob 是否跳过 SSH 连接检查、是否跳过 SSH 状态轮询 |
| 影响范围 | 全局——部署、API、所有资源操作都依赖此判定 | 局部——仅影响 ServerManagerJob 的调度决策 |
| 失效后果 | 所有部署失败（快速 fail）、SSH 多路复用 socket 被清理 | 回退到 SSH 轮询路径（降级而非中断） |
| 恢复方式 | ServerConnectionCheckJob 检测到 SSH 可达后设置 is_reachable=true | Sentinel 恢复推送后 sentinel_updated_at 自动刷新 |
| 适用服务器 | 所有服务器 | 仅 isSentinelEnabled()=true 的服务器 |

**关键区别**：`isFunctional()` 是服务器可操作性的**权威判定**——只要它返回 false，所有需要 SSH 的操作（部署、Sentinel push 接受等）都会被拒绝，并清理 SSH 多路复用文件。`isSentinelLive()` 是一种**优化手段**——仅用于决定是否跳过冗余的 SSH 检查，失效后系统降级运行而非中断。

### isFunctional 的 ssh-mux 清理细节

[Server::isFunctional()](file:///d:/fz/0601-1/solo-dogfeeding/code/98-coolify/app/Models/Server.php#L1083-L1092) 在返回 false 时会主动删除 SSH 多路复用控制文件：

```php
public function isFunctional()
{
    $isFunctional = data_get($this->settings, 'is_reachable')
        && data_get($this->settings, 'is_usable')
        && data_get($this->settings, 'force_disabled') === false
        && $this->ip !== '1.2.3.4';

    if ($isFunctional === false) {
        Storage::disk('ssh-mux')->delete($this->muxFilename());
    }

    return $isFunctional;
}
```

[muxFilename()](file:///d:/fz/0601-1/solo-dogfeeding/code/98-coolify/app/Models/Server.php#L1027-L1030) 返回 `mux_{server_uuid}`。SSH 多路复用控制文件是一个 Unix socket，用于复用 SSH 连接。当服务器被判定为不可操作时，删除该文件可以：
- 避免后续 SSH 命令尝试使用已失效的多路复用连接
- 强制下次连接时建立新的 SSH 会话
- 防止僵死的多路复用 socket 占用文件描述符

### 健康记录字段汇总

| 表 | 字段 | 更新者 | 含义 |
|----|------|--------|------|
| servers | sentinel_updated_at | SentinelController / StartSentinel / StopSentinel | Sentinel 心跳时间戳 |
| servers | unreachable_count | ServerConnectionCheckJob | 连续不可达计数 |
| servers | unreachable_notification_sent | isReachableChanged() | 是否已发送不可达通知 |
| server_settings | is_reachable | ServerConnectionCheckJob / validateConnection() | SSH 可达性 |
| server_settings | is_usable | ServerConnectionCheckJob | Docker 可用性 |
| server_settings | is_metrics_enabled | 用户操作 | 指标采集开关（影响 isSentinelEnabled 判定） |
| server_settings | is_sentinel_enabled | StartSentinel / 用户操作 | Sentinel Server API 开关（影响 isSentinelEnabled 判定） |

---

## 七、服务器掉线期间排队任务堆积的处理路径

### 7.1 入队时的保护

[queue_application_deployment()](file:///d:/fz/0601-1/solo-dogfeeding/code/98-coolify/bootstrap/helpers/applications.php#L15-L80) 在创建部署任务时有两道防线：

**防线 1：队列容量限制**

```php
$queue_limit = $serverForQueueCheck->settings->deployment_queue_limit ?? 25;
$queued_count = ApplicationDeploymentQueue::where('server_id', $server_id)
    ->where('status', ApplicationDeploymentStatus::QUEUED->value)
    ->count();

if ($queued_count >= $queue_limit) {
    return ['status' => 'queue_full', ...];
}
```

默认限制 25 个排队任务/服务器。当服务器掉线时，由于 `ApplicationDeploymentJob` 执行时会立即检测 `isFunctional()` 并失败，**新任务不会无限堆积**——它们会在执行时快速失败。

**防线 2：重复部署去重**

```php
$existing_deployment = ApplicationDeploymentQueue::where('application_id', ...)
    ->where('commit', ...)
    ->whereIn('status', [IN_PROGRESS, QUEUED])
    ->first();
// 如果已存在，跳过（除非 force_rebuild / rollback / no_questions_asked）
```

### 7.2 执行时的快速失败

[ApplicationDeploymentJob::handle()](file:///d:/fz/0601-1/solo-dogfeeding/code/98-coolify/app/Jobs/ApplicationDeploymentJob.php#L297-L302)：

```php
if ($this->server->isFunctional() === false) {
    $this->application_deployment_queue->addLogEntry('Server is not functional.');
    $this->fail('Server is not functional.');
    return;
}
```

`isFunctional()` 检查：`is_reachable && is_usable && !force_disabled && ip !== '1.2.3.4'`

这意味着：
- 掉线服务器的排队部署会**立即失败**，不会阻塞队列
- 失败的部署状态变为 `failed`，从队列中移除
- 不会重试 (`$tries = 1`)

### 7.3 掉线期间的完整任务生命周期

```
服务器掉线
  │
  ├── 新部署请求入队
  │   ├── 队列未满 → 创建 ApplicationDeploymentQueue (status=queued)
  │   └── 队列已满 → 返回 queue_full，拒绝入队
  │
  ├── Horizon worker 取出任务执行
  │   ├── isFunctional() = false → 立即 fail("Server is not functional")
  │   │   └── 同时删除 ssh-mux 控制文件 (Storage::disk('ssh-mux')->delete)
  │   └── status 变为 failed，释放队列位
  │
  ├── 后续部署请求继续入队/快速失败循环
  │   └── 队列不会无限堆积（快速失败 + 容量限制双重保护）
  │
  └── 服务器恢复
      ├── ServerConnectionCheckJob 标记 is_reachable=true
      ├── 后续新部署正常执行
      └── ServerCheckJob 更新容器状态（可能标记旧容器为 exited）
```

### 7.4 手动/自动清理机制

**手动清理命令：**

- [check:deployment-queue](file:///d:/fz/0601-1/solo-dogfeeding/code/98-coolify/app/Console/Commands/CheckApplicationDeploymentQueue.php#L9-L49)：查找超过指定秒数（默认 3600s）仍为 `in_progress` 或 `queued` 的部署，交互式或强制标记为 `failed`
- [cleanup:deployment-queue](file:///d:/fz/0601-1/solo-dogfeeding/code/98-coolify/app/Console/Commands/CleanupApplicationDeploymentQueue.php#L9-L25)：按 team_id 清理该团队所有服务器的 `in_progress/queued` 部署为 `failed`
- [cleanup:stucked-resources](file:///d:/fz/0601-1/solo-dogfeeding/code/98-coolify/app/Console/Commands/CleanupStuckedResources.php)：删除关联 Application 已不存在的 DeploymentQueue 记录

**自动清理：**

- [cleanup:unreachable-servers](file:///d:/fz/0601-1/solo-dogfeeding/code/98-coolify/app/Console/Commands/CleanupUnreachableServers.php#L8-L31)：每日运行，清理连续不可达 ≥ 3 次 + 已通知 + 更新于 7 天前的服务器（云环境改 IP 为 `1.2.3.4`，自建环境 `forceDisableServer`）

### 7.5 掉线期间容器状态的退化表示

当服务器掉线后，[Server::status()](file:///d:/fz/0601-1/solo-dogfeeding/code/98-coolify/app/Models/Server.php#L1204-L1233) 在 SSH 不可达时将所有资源标记为 `exited`：

```php
if ($uptime === false) {
    foreach ($this->applications() as $application) {
        $application->status = 'exited';
        $application->save();
    }
    foreach ($this->databases() as $database) {
        $database->status = 'exited';
        $database->save();
    }
    foreach ($this->services() as $service) {
        $apps = $service->applications()->get();
        $dbs = $service->databases()->get();
        foreach ($apps as $app) {
            $app->status = 'exited';
            $app->save();
        }
        foreach ($dbs as $db) {
            $db->status = 'exited';
            $db->save();
        }
    }

    return false;
}
```

Services 不是整体标记 `exited`，而是逐个遍历其 `applications()` 和 `databases()` 子资源分别标记。这是因为 Service 本身没有 `status` 字段——它的"状态"由子资源的状态聚合而来。

而 `PushServerUpdateJob` 中的 [updateNotFoundApplicationStatus](file:///d:/fz/0601-1/solo-dogfeeding/code/98-coolify/app/Jobs/PushServerUpdateJob.php#L612-L629) 等方法有额外的安全保护：

```php
if ($this->containers->isEmpty()) {
    return;  // 容器列表为空可能是 Sentinel 故障，不标记为 exited
}
```

这防止了 Sentinel 自身故障时误将所有容器标记为 `exited`。

---

## 八、四段协作时序图

```
每分钟 ──────────────────────────────────────────────────────────────────
  │
  └── ServerManagerJob::handle()
       │
       ├─── dispatchConnectionChecks()
       │     │
       │     ├── Sentinel Live? ──── YES ──→ 跳过 SSH 连接检查
       │     │                           （心跳即连通性证明）
       │     │
       │     └── Sentinel Not Live / Disabled
       │           │
       │           ├── 退避跳过? → YES → 跳过
       │           │
       │           └── dispatch ServerConnectionCheckJob
       │                 │
       │                 ├── SSH 可达? ── YES → is_reachable=true
       │                 │                 Docker 可用? → is_usable=true/false
       │                 │                 unreachable_count=0
       │                 │                 发恢复通知 (如需要)
       │                 │
       │                 └── SSH 不可达 → is_reachable=false, is_usable=false
       │                                 unreachable_count++
       │                                 退避策略升级
       │                                 发不可达通知 (count>=2)
       │
       └─── processScheduledTasks()
             │
             ├── sentinelOutOfSync? ──── YES → dispatch ServerCheckJob
             │     │                         │
             │     │                         ├── serverStatus() 失败 → 返回
             │     │                         ├── 获取容器列表 + 同步状态
             │     │                         ├── isSentinelEnabled()? → dispatch CheckAndStartSentinelJob
             │     │                         └── 检查 Proxy/Log Drain
             │     │
             │     └── NO → Sentinel 推送已覆盖状态同步，无需 SSH
             │
             ├── 每日零点 + isSentinelEnabled()? → dispatch CheckAndStartSentinelJob
             │                                      （版本更新检查）
             │
             └── sentinelOutOfSync? ──→ dispatch ServerStorageCheckJob
                                        （Sentinel 活跃时推送已包含磁盘数据）

持续 ──────────────────────────────────────────────────────────────────
  │
  └── Sentinel 容器（远程服务器上）
        │
        └── 每 push_interval_seconds → HTTP POST → SentinelController::push()
              │
              ├── Token 缺失/无效 → 401
              ├── 云端未付费 → 401
              ├── isFunctional()=false → 401
              ├── Token 不匹配 → 401
              ├── containers 校验失败 → 422
              │
              ├── sentinelHeartbeat() → 更新 sentinel_updated_at
              │
              └── shouldDispatchUpdate() ──→ dispatch PushServerUpdateJob
                    │                           │
                    │ (hash 变化或              ├── 同步容器状态到资源表
                    │  强制窗口过期)             ├── 磁盘超阈值 → ServerStorageCheckJob
                    │                           ├── Proxy 缺失 → StartProxy
                    │                           └── Log Drain 缺失 → StartLogDrain
                    │
                    └── (hash 未变 + 窗口未过) → 跳过，节省数据库开销
```

---

## 九、关键设计要点总结

1. **心跳驱动的 SSH 短路**：Sentinel 心跳存活时跳过 SSH 连接检查，大幅减少 SSH 连接数
2. **双重状态轮询（互斥）**：Sentinel 推送（轻量、高频）+ SSH 轮询（重量、低频、仅失同步时），二者互斥，Swarm 仅支持 SSH 路径
3. **isSentinelEnabled 是 OR 组合**：`isMetricsEnabled() || isServerApiEnabled()`，`is_sentinel_enabled` 只是子开关而非总开关
4. **去重推送（分布式锁保护）**：`Cache::lock(10s)->block(5s)` 保护 read-modify-write 序列，拿不到锁就跳过（宁可少派一次，绝不多派一次）
5. **force window 是漏检安全网**：hash 只抓 state 变化，health/disk/proxy 等变化靠 300s 强制窗口兜底，实现最终一致性
6. **hash 粗粒度 vs status 细粒度**：hash 只取 `name+state` 去重，status 写入 `state:health` 展示，二者语义不同——大事实时响应，小事定期刷新
7. **磁盘三级防抖链**：推送阈值边沿 → 10 分钟数值变化去重 → 1 小时 RateLimiter 通知，层层过滤噪声
8. **Cache::forget 跨阈值重置**：磁盘降到阈值以下时立即清缓存，确保下次上升沿一定能触发，不会漏掉
9. **退避分粒度时序**：自建 1min/周期、云环境 5min/周期，退避用周期倍数而非绝对时间，两套环境复用同一套逻辑
10. **哈希分散避惊群**：`crc32(server_id)` 分配检查槽位，同一时刻不会所有服务器同时检查
11. **两套 lock/节流机制**：Cache::lock 解决并发竞争（同一时刻互斥），shouldRunCronNow 解决周期去重（同一周期内幂等），用途完全不同
12. **超时清 Horizon failed**：TimeoutExceededException 时 `$this->job->delete()` 静默失败，预期内失败不进 failed 列表
13. **崩溃自愈**：Sentinel 失同步 → SSH 轮询 → 发现 Sentinel 挂 → 自动重启
14. **配置热重载全链**：ServerSetting updated 事件 → restartSentinel() → StartSentinel(dispatch) → StopSentinel(sync) + docker run + heartbeat + broadcast
15. **创建期静默初始化**：ServerSetting creating 事件自动生成 token 和 URL，用 `saveQuietly()` 不触发 updated 事件，避免刚创建就重启
16. **字符白名单防注入**：`isValidSentinelToken` 用正则 `[a-zA-Z0-9._\-+=\/]` 限制 token 字符集，拼 shell 命令时无注入风险
17. **Sentinel API 边车自洽**：Docker health check 调自己的 /api/health，版本检查调自己的 /api/version，只监听 127.0.0.1，安全不暴露
18. **auditLog 双通道**：成功走 info 级别、失败走 auditLogWebhookFailure 的 warning 级别，都进 audit channel，失败安全绝不打断主流程
19. **构造即副作用的 Event**：ServerReachabilityChanged 在构造函数里直接执行 isReachableChanged()，无 Listener，4 个 dispatch 点的副作用集中在构造函数中，导致加 Listener 无效、可扩展性差、Event::fake() 单测无法隔离副作用，名不副实更像 Action 而非 Event
20. **ServerPatchCheckJob 周日扫描**：每周日零点检查系统更新，tries=3、timeout=600s、WithoutOverlapping 不重叠，服务器离线直接跳过
21. **Console Kernel dev/prod 双轨制**：dev 环境去掉了所有 cleanup 类任务（unreachable-servers/database/preview 清理/redis 锁/sanctum 过期），只保留核心调度，避免开发时数据被清
22. **isFunctional 的 ssh-mux 清理**：服务器不可操作时删除多路复用控制文件，防止僵死连接
23. **快速失败部署**：服务器不可用时部署任务立即失败，不阻塞队列
24. **队列容量限制**：每服务器 `deployment_queue_limit`（默认 25）防堆积
25. **空容器列表保护**：PushServerUpdateJob 中容器列表为空时不标记资源 exited，防误判
26. **最终清理**：7 天不可达服务器自动禁用，防止永远重试已离线服务器
27. **云端付费校验**：Sentinel push 在云环境校验订阅付费状态，未付费返回 401
