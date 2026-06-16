# Sentinel Watchdog 协作机制全解

本文档追踪 Coolify 中「远程守护进程检查」「状态轮询」「自动启动」「异常恢复」四个段的协作方式，厘清 Sentinel 守护组件状态与服务器健康记录的同步逻辑，并剖析服务器掉线期间排队任务堆积的完整处理路径。

---

## 一、四大组件概览与入口调度

所有定时检查的唯一入口是 [ServerManagerJob](file:///d:/fz/0601-1/solo-dogfeeding/code/98-coolify/app/Jobs/ServerManagerJob.php#L18-L208)，它由 [Console Kernel](file:///d:/fz/0601-1/solo-dogfeeding/code/98-coolify/app/Console/Kernel.php#L59) 每分钟调度一次 (`everyMinute()->onOneServer()`)。

`ServerManagerJob::handle()` 做两件事：

1. **`dispatchConnectionChecks()`** — 连接检查（远程守护进程检查）
2. **`processScheduledTasks()`** — 状态轮询 + 自动启动 + 异常恢复

下面按这四个段逐一拆解。

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

### 可达性变更事件

[ServerReachabilityChanged](file:///d:/fz/0601-1/solo-dogfeeding/code/98-coolify/app/Events/ServerReachabilityChanged.php#L8-L17) 在构造函数中直接调用 `server->isReachableChanged()`：

- 服务器恢复可达 + 之前发过不可达通知 → 发送恢复通知
- 服务器不可达 + `unreachable_count >= 2` + 尚未发过通知 → 发送不可达通知

阈值为 2：单次抖动不触发通知，连续 2 次不可达才发。

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

[PushServerUpdateJob](file:///d:/fz/0601-1/solo-dogfeeding/code/98-coolify/app/Jobs/PushServerUpdateJob.php#L41-L807) 的核心同步逻辑：

1. 更新磁盘使用率 → 超阈值时派发 `ServerStorageCheckJob`
2. 遍历所有容器，按 `coolify.managed` 标签分类
3. 更新 Application 状态（含多容器聚合）
4. 更新 ApplicationPreview 状态
5. 更新 Database 状态 + TCP Proxy 管理
6. 更新 Service 子资源状态
7. 标记未找到的资源为 `exited`
8. 检查 Proxy 和 Log Drain 容器

> **Swarm 不支持**：`PushServerUpdateJob` 开头有 `// TODO: Swarm is not supported yet`，Swarm 集群只能通过 SSH 路径 (ServerCheckJob) 获取容器状态。

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
4. **去重推送**：容器状态 hash + 强制窗口，避免每分钟执行重量级数据库操作
5. **退避策略**：不可达次数越多检查越稀疏，使用哈希分散避免惊群
6. **崩溃自愈**：Sentinel 失同步 → SSH 轮询 → 发现 Sentinel 挂 → 自动重启
7. **配置热重载全链**：ServerSetting updated 事件 → restartSentinel() → StartSentinel(dispatch) → StopSentinel(sync) + docker run + heartbeat + broadcast
8. **isFunctional 的 ssh-mux 清理**：服务器不可操作时删除多路复用控制文件，防止僵死连接
9. **快速失败部署**：服务器不可用时部署任务立即失败，不阻塞队列
10. **队列容量限制**：每服务器 `deployment_queue_limit`（默认 25）防堆积
11. **空容器列表保护**：PushServerUpdateJob 中容器列表为空时不标记资源 exited，防误判
12. **最终清理**：7 天不可达服务器自动禁用，防止永远重试已离线服务器
13. **云端付费校验**：Sentinel push 在云环境校验订阅付费状态，未付费返回 401
