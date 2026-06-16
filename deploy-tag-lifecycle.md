# 部署镜像 tag 分支与队列生命周期分析

## 一、generate_image_names 的四分支决策树

[generate_image_names()](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/app/Jobs/ApplicationDeploymentJob.php#L1158-L1200) 是镜像命名的核心分发器，其分支优先级和判定条件如下：

### 1.1 分支优先级（从上到下）

```
generate_image_names()
  ├─ 分支 1: $this->application->dockerfile 非空
  │    └─ 优先级最高，覆盖 build_pack 枚举
  │
  ├─ 分支 2: $this->application->build_pack === 'dockerimage'
  │    └─ 处理 Docker Image 类型应用
  │
  ├─ 分支 3: $this->pull_request_id !== 0
  │    └─ 处理 PR 预览部署（非 dockerfile / 非 dockerimage）
  │
  └─ 分支 4: 其他所有场景
       └─ 正式部署（nixpacks/railpack/static/dockerfile/dockercompose）
```

**重要**：分支 1 使用的是 `$this->application->dockerfile` 字段（数据库中存储的 Dockerfile 内容），而不是 `build_pack === 'dockerfile'` 枚举判断。这意味着即使 `build_pack` 是 `nixpacks`，只要 `dockerfile` 字段有内容，就会走分支 1 的逻辑。

### 1.2 BuildPackTypes 枚举

[BuildPackTypes.php](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/app/Enums/BuildPackTypes.php#L5-L12)：

```php
enum BuildPackTypes: string
{
    case NIXPACKS = 'nixpacks';
    case STATIC = 'static';
    case DOCKERFILE = 'dockerfile';
    case DOCKERCOMPOSE = 'dockercompose';
    case RAILPACK = 'railpack';
}
```

注意枚举中**没有** `dockerimage`，这是一个历史遗留的特殊 build_pack 值，直接存储为字符串。

### 1.3 dockerfile 字段与 build_pack 的同步逻辑

在 [Application 模型 saved 事件](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/app/Models/Application.php#L295-L322) 中：

```php
if ($application->isDirty('build_pack')) {
    $originalBuildPack = $application->getOriginal('build_pack');

    // 切换 away from dockerfile 时，清空 dockerfile 相关字段
    if ($originalBuildPack === 'dockerfile') {
        $application->dockerfile = null;
        $application->dockerfile_location = null;
        $application->dockerfile_target_build = null;
    }
}
```

这保证了从 dockerfile 切换到其他 build_pack 时，`dockerfile` 字段会被清空，从而不会再误走分支 1。

### 1.4 分支 1：`dockerfile` 字段非空（内联 Dockerfile）

**条件**：`$this->application->dockerfile !== null && $this->application->dockerfile !== ''`

无论 `build_pack` 是什么（即使是 `dockercompose`！），只要 `dockerfile` 字段有内容，就走这个分支。

| 场景 | `docker_registry_image_name` 存在 | 镜像名 |
|---|---|---|
| 自定义 registry | 是 | `build: {registry_name}:build` <br> `production: {registry_name}:latest` |
| 本地镜像 | 否 | `build: {uuid}:build` <br> `production: {uuid}:latest` |

**注意**：这个分支完全忽略 `pull_request_id`，预览部署也使用固定的 `build` / `latest` tag！这意味着如果一个 PR 预览部署的应用 `dockerfile` 字段有内容，所有 PR 都会构建相同 tag 的镜像，可能互相覆盖。

**调用路径**：

```
decide_what_to_do()
  └─ $this->application->dockerfile (非 build_pack 判断)
       └─ deploy_simple_dockerfile()
            └─ generate_image_names() → 分支 1
```

### 1.5 分支 2：`build_pack === 'dockerimage'`

**条件**：`dockerfile` 为空 + `build_pack === 'dockerimage'`

[resolveDockerImageTag()](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/app/Jobs/ApplicationDeploymentJob.php#L594-L605) 先计算 tag：

```php
private function resolveDockerImageTag(): string
{
    // 1. PR 预览 + 有 preview tag → 用 preview tag
    if ($this->pull_request_id !== 0 && str($this->dockerImagePreviewTag)->isNotEmpty()) {
        return $this->dockerImagePreviewTag;
    }
    // 2. 应用配置了 docker_registry_image_tag → 用它
    if (str($this->application->docker_registry_image_tag)->isNotEmpty()) {
        return $this->application->docker_registry_image_tag;
    }
    // 3. 默认 → 'latest'
    return 'latest';
}
```

然后：

| 场景 | tag 格式 | 最终镜像名 |
|---|---|---|
| SHA256 部署（tag 以 `sha256-` 开头） | `sha256-{hash}` | `{image}@sha256:{hash}` |
| 普通 tag 部署 | `v1.0`、`latest` 等 | `{image}:{tag}` |

**调用路径**：

```
decide_what_to_do()
  └─ build_pack === 'dockerimage'
       └─ deploy_dockerimage_buildpack()
            └─ resolveDockerImageTag()
            └─ generate_image_names() → 分支 2
```

### 1.6 分支 3：PR 预览部署（非 dockerfile / 非 dockerimage）

**条件**：`dockerfile` 为空 + `build_pack !== 'dockerimage'` + `pull_request_id !== 0`

[previewImageTag()](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/app/Jobs/ApplicationDeploymentJob.php#L1202-L1221) 计算 tag：

```php
// 格式: pr-{pr_id}-{sanitized_commit}[-build]
$prefix = "pr-{$this->pull_request_id}-";
$suffix = $build ? '-build' : '';
$commit = substr(commit_or_deployment_uuid, 0, max_length);
return "{$prefix}{$commit}{$suffix}";
```

然后：

| 场景 | 镜像名 |
|---|---|
| 自定义 registry | `build: {registry_name}:pr-{pr_id}-{commit}-build` <br> `production: {registry_name}:pr-{pr_id}-{commit}` |
| 本地镜像 | `build: {uuid}:pr-{pr_id}-{commit}-build` <br> `production: {uuid}:pr-{pr_id}-{commit}` |

**调用路径**：

```
decide_what_to_do()
  └─ pull_request_id !== 0
       └─ deploy_pull_request()
            └─ (针对非 compose) deploy_dockerfile_buildpack() / ...
                 └─ generate_image_names() → 分支 3
```

### 1.7 分支 4：正式部署（其他所有场景）

**条件**：`dockerfile` 为空 + `build_pack !== 'dockerimage'` + `pull_request_id === 0`

tag 计算：
```php
$this->dockerImageTag = str($this->commit)->substr(0, 128);
```

然后：

| 场景 | 镜像名 |
|---|---|
| 自定义 registry | `build: {registry_name}:{commit}-build` <br> `production: {registry_name}:{commit}` |
| 本地镜像 | `build: {uuid}:{commit}-build` <br> `production: {uuid}:{commit}` |

注意 L1189-L1191 有一段被注释掉的代码，说明曾经计划支持 `docker_registry_image_tag` 覆盖 commit tag，但后来被取消了。

**调用路径**：

```
decide_what_to_do()
  └─ build_pack in {nixpacks, railpack, static, dockerfile, dockercompose}
       └─ deploy_*_buildpack()
            └─ generate_image_names() → 分支 4
```

### 1.8 决策树真值表

| dockerfile 非空 | build_pack | pull_request_id | 走哪一分支 |
|---|---|---|---|
| ✅ 是 | *（任意）* | *（任意）* | **分支 1**（内联 Dockerfile） |
| ❌ 否 | `dockerimage` | *（任意）* | **分支 2**（Docker Image 类型） |
| ❌ 否 | `nixpacks` 等 | > 0 | **分支 3**（PR 预览） |
| ❌ 否 | `nixpacks` 等 | = 0 | **分支 4**（正式部署） |

### 1.9 特殊情况：build_pack = 'dockercompose' 且 dockerfile 非空

理论上，如果一个 `dockercompose` 类型应用的 `dockerfile` 字段被意外填充了内容，`generate_image_names` 会走分支 1（内联 Dockerfile），而 `decide_what_to_do` 会走 `deploy_docker_compose_buildpack()` 路径。这会导致 compose 部署使用 `{uuid}:build` / `{uuid}:latest` 作为镜像 tag。

---

## 二、部署队列生命周期

### 2.1 ApplicationDeploymentStatus 枚举

[ApplicationDeploymentStatus.php](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/app/Enums/ApplicationDeploymentStatus.php#L5-L12)：

```php
enum ApplicationDeploymentStatus: string
{
    case QUEUED = 'queued';           // 排队中
    case IN_PROGRESS = 'in_progress'; // 执行中
    case FINISHED = 'finished';       // 成功完成
    case FAILED = 'failed';           // 失败
    case CANCELLED_BY_USER = 'cancelled-by-user'; // 用户取消
}
```

**FINISHED / FAILED / CANCELLED_BY_USER** 为终态，一旦进入不可变更。

### 2.2 入队前置门

#### 2.2.1 queue_full 门

[queue_application_deployment() L33-L45](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/bootstrap/helpers/applications.php#L33-L45)：

```php
$queue_limit = $serverForQueueCheck->settings->deployment_queue_limit ?? 25;
$queued_count = ApplicationDeploymentQueue::where('server_id', $server_id)
    ->where('status', ApplicationDeploymentStatus::QUEUED->value)
    ->count();

if ($queued_count >= $queue_limit) {
    return [
        'status' => 'queue_full',
        'message' => 'Deployment queue is full.',
    ];
}
```

- 按 **server** 维度计数（不是 application）
- 只统计 **QUEUED** 状态（不包括 IN_PROGRESS）
- 默认限制 25，可通过 `server.settings.deployment_queue_limit` 调整
- `queue_full` 返回后，不会创建 `ApplicationDeploymentQueue` 记录，也不会派发 Job

#### 2.2.2 重复部署去重门

[queue_application_deployment() L48-L66](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/bootstrap/helpers/applications.php#L48-L66)：

四维 key + 状态检查：

```php
$existing_deployment = ApplicationDeploymentQueue::where('application_id', $application_id)
    ->where('commit', $commit)
    ->where('pull_request_id', $pull_request_id)
    ->where('docker_registry_image_tag', $docker_registry_image_tag)
    ->whereIn('status', [IN_PROGRESS, QUEUED])
    ->first();
```

- 四维完全匹配且状态为 QUEUED/IN_PROGRESS 时，返回 `skipped`
- `force_rebuild` / `rollback` / `no_questions_asked` 任一为 true 时跳过此检查

#### 2.2.3 立即执行 or 排队

[queue_application_deployment() L88-L102](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/bootstrap/helpers/applications.php#L88-L102)：

```php
if ($no_questions_asked) {
    // 无询问模式：直接 IN_PROGRESS + dispatch
    $deployment->update(['status' => IN_PROGRESS]);
    ApplicationDeploymentJob::dispatch($deployment->id);
} elseif (next_queuable($server_id, $application_id, $commit, $pull_request_id)) {
    // 满足并发条件：立即执行
    $deployment->update(['status' => IN_PROGRESS]);
    ApplicationDeploymentJob::dispatch($deployment->id);
}
// 否则：保持 QUEUED 状态，等待 queue_next_deployment() 调度
```

### 2.3 next_queuable 并发控制

[next_queuable()](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/bootstrap/helpers/applications.php#L142-L167)：

两层并发检查：

1. **Application 维度**：同一 `application_id + pull_request_id` 只能有一个 IN_PROGRESS
   - 正式部署（pull_request_id=0）与预览部署（pull_request_id>0）可并行
   - 不同 PR 之间可并行

2. **Server 维度**：同一 server 的 IN_PROGRESS 数量不能超过 `server.settings.concurrent_builds`
   - 默认值未明确给出，由服务器设置控制

两层检查都通过才返回 `true`。

### 2.4 Job 执行阶段

[ApplicationDeploymentJob::handle()](file:///d:/fz\0601-1\solo-dogfeeding\code\95-coolify\app\Jobs\ApplicationDeploymentJob.php#L283-L408)：

```
1. 前置检查：是否已被 CANCELLED_BY_USER？是 → 直接 return
2. 更新状态为 IN_PROGRESS，记录 horizon_job_worker
3. 检查 server 是否 functional
4. 执行构建/部署流程（decide_what_to_do）
5. post_deployment → completeDeployment() → transitionToStatus(FINISHED)
6. catch 异常 → fail($e) → transitionToStatus(FAILED)
7. finally：更新 finished_at、清理 build 容器、写配置、发事件
```

执行过程中 `ExecuteRemoteCommand` trait 会在关键点检查 `CANCELLED_BY_USER` 状态，如发现则抛出异常终止执行。

### 2.5 成功路径：completeDeployment

[completeDeployment()](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/app/Jobs/ApplicationDeploymentJob.php#L4829-L4832)：

```php
private function completeDeployment(): void
{
    $this->transitionToStatus(ApplicationDeploymentStatus::FINISHED);
}
```

### 2.6 失败路径：failed

[failed()](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/app/Jobs/ApplicationDeploymentJob.php#L4843-L4893) 是 Laravel 队列的失败回调方法：

```
1. transitionToStatus(FAILED)
2. 详细记录错误日志（错误类型、代码、位置、链式异常、堆栈前5行）
3. 非 dockercompose 且非 69420 错误码时，尝试删除新容器
```

### 2.7 transitionToStatus 核心转换

[transitionToStatus()](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/app/Jobs/ApplicationDeploymentJob.php#L4721-L4730)：

```php
private function transitionToStatus(ApplicationDeploymentStatus $status): void
{
    if ($this->isInTerminalState()) {
        return;  // 终态不可逆转
    }
    $this->updateDeploymentStatus($status);
    $this->handleStatusTransition($status);
    queue_next_deployment($this->application);  // 关键：调度下一个
}
```

**终态检查**（isInTerminalState）：
- FINISHED → return true
- FAILED → return true
- CANCELLED_BY_USER → 抛出 DeploymentException(69420) 终止执行

**状态转换副作用**（handleStatusTransition）：
- FINISHED → reset restart_count、mark configuration applied、deploy to additional servers、发送成功通知
- FAILED → 发送失败通知
- CANCELLED_BY_USER → 无副作用（由外部取消者处理）

### 2.8 完成后调度：queue_next_deployment

[queue_next_deployment()](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/bootstrap/helpers/applications.php#L120-L140)：

```
1. 找到该 server 下所有 QUEUED 部署，按 created_at 排序（FIFO）
2. 遍历检查每个部署的 next_queuable()
3. 通过检查 → 更新为 IN_PROGRESS + dispatch Job
4. 不通过 → 跳过，等待下一次调度
```

**关键点**：每次有部署结束（无论成功或失败），都会尝试启动下一个排队中的部署。

### 2.9 取消流程

取消操作可在多个入口触发：

#### 2.9.1 UI 取消（DeploymentNavbar）

[DeploymentNavbar::cancelDeployment()](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/app/Livewire/Project/Application/DeploymentNavbar.php#L80-L87)：

```php
$this->deployment->update([
    'status' => ApplicationDeploymentStatus::CANCELLED_BY_USER->value,
]);
$this->deployment->addLogEntry('Cancelling deployment...');
next_after_cancel($this->application->destination->server);
```

#### 2.9.2 API 取消

[DeployController::cancel_deployment()](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/app/Http/Controllers/Api/DeployController.php#L247-L254)：

```php
$deployment->update([
    'status' => ApplicationDeploymentStatus::CANCELLED_BY_USER->value,
]);
next_after_cancel($deployment->server);
```

#### 2.9.3 PR 预览清理（CleanupPreviewDeployment）

[CleanupPreviewDeployment](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/app/Actions/Application/CleanupPreviewDeployment.php#L97-L101)：

```php
ApplicationDeploymentQueue::where('application_id', $preview->application_id)
    ->where('pull_request_id', $preview->pull_request_id)
    ->whereIn('status', [QUEUED, IN_PROGRESS])
    ->update(['status' => ApplicationDeploymentStatus::CANCELLED_BY_USER->value]);
```

#### 2.9.4 资源删除（DeleteResourceJob）

[DeleteResourceJob](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/app/Jobs/DeleteResourceJob.php#L137-L141) 取消应用删除时相关的活跃部署。

### 2.10 取消后调度：next_after_cancel

[next_after_cancel()](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/bootstrap/helpers/applications.php#L168-L191)：

```
1. 找到该 server 下所有 QUEUED 部署，按 created_at 排序
2. 遍历检查 next_queuable()
3. 通过检查 → 更新为 IN_PROGRESS + dispatch Job
```

与 `queue_next_deployment()` 的区别：
- `queue_next_deployment()` 以 `Application` 为参数，只调度同一 application 的下一个部署？不，代码看都是按 server 的。实际上两者逻辑几乎相同，唯一区别是传入参数不同（一个是 Application，一个是 Server）。

### 2.11 超时守护：CheckApplicationDeploymentQueue

[CheckApplicationDeploymentQueue.php](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/app/Console/Commands/CheckApplicationDeploymentQueue.php)：

每天（或按需）执行的 Artisan 命令：
- 查找 created_at 超过 N 秒（默认 3600）且状态为 QUEUED/IN_PROGRESS 的部署
- 交互式或强制（--force）取消它们
- 取消时标记为 FAILED，并 `docker rm -f` 相关容器

---

## 三、完整生命周期状态机

### 3.1 状态转换图

```
               queue_full → 直接拒绝（无记录）
                  ▲
                  │
queue_application_deployment()
                  │
                  ├─ 去重跳过 → skipped（返回现有 deployment）
                  │
                  ▼
               QUEUED ◀──────────┐
                  │               │
                  │ next_queuable │ 不满足并发条件
                  ▼               │
               IN_PROGRESS        │
                  │               │
            ┌─────┼─────┐         │
            ▼     ▼     ▼         │
        FINISHED FAILED CANCELLED_BY_USER
            │     │     │
            └─────┼─────┘
                  │
                  └─ queue_next_deployment() → 启动下一个 QUEUED
```

### 3.2 典型场景时序

#### 场景 1：单应用串行部署（同一 PR 连续推送 3 个 commit）

```
时间点 T1: commit A 入队
  → next_queuable=true → IN_PROGRESS → dispatch Job

时间点 T2: commit B 入队（commit A 还在 IN_PROGRESS）
  → next_queuable=false（同 application_id 已有 IN_PROGRESS）→ 保持 QUEUED

时间点 T3: commit C 入队（commit A 还在 IN_PROGRESS）
  → next_queuable=false → 保持 QUEUED

时间点 T4: commit A 完成（FINISHED）
  → transitionToStatus 调用 queue_next_deployment()
  → 遍历 server 下所有 QUEUED，找到 commit B（最早）
  → next_queuable=true → commit B → IN_PROGRESS → dispatch

时间点 T5: commit B 完成
  → queue_next_deployment() → commit C → IN_PROGRESS → dispatch
```

#### 场景 2：用户取消正在执行的部署

```
时间点 T1: 部署 D 在 IN_PROGRESS

时间点 T2: 用户点击取消
  → 状态更新为 CANCELLED_BY_USER
  → 调用 next_after_cancel(server)
  → 如有 QUEUED 部署，满足条件的立即启动

时间点 T3: Job 下一次调用 execute_remote_command
  → trait 检测到 CANCELLED_BY_USER
  → 抛出异常 → catch 块 fail()
  → fail() 调用 transitionToStatus(FAILED)
  → isInTerminalState() 检测到 CANCELLED_BY_USER → 抛 69420 异常
  → Laravel 队列捕获异常，调用 failed() 方法
  → failed() 再次调用 transitionToStatus(FAILED)
  → isInTerminalState() 检测到 CANCELLED_BY_USER → 抛 69420 异常
  → 最终状态为 CANCELLED_BY_USER（数据库中已在 T2 更新）
```

> 注意：取消操作有一个"先标记状态，后终止执行"的异步间隙。Job 会在下次远程命令执行时检测到取消状态并退出。

#### 场景 3：正式部署和预览部署并行

```
时间点 T1: 正式部署（pr=0）入队
  → next_queuable(pr=0) → IN_PROGRESS

时间点 T2: PR #42 预览部署入队
  → next_queuable(pr=42) → 检查同 application+pr=42 是否有 IN_PROGRESS → 无
  → 检查 server 并发限制 → 还有空余
  → IN_PROGRESS，与正式部署并行执行
```

#### 场景 4：队列已满（queue_full）

```
时间点 T1: 已有 25 个 QUEUED 部署（默认限制）

时间点 T2: 新部署请求
  → queued_count(25) >= queue_limit(25) → 返回 queue_full
  → 不创建记录，不派发 Job
  → 调用方需自行重试
```

---

## 四、关键发现与潜在问题

### 4.1 generate_image_names 分支 1 优先级过高

`dockerfile` 字段非空就走分支 1，完全忽略 `build_pack` 枚举。这意味着：
- 如果一个 `dockercompose` 应用意外填充了 `dockerfile` 字段，会使用错误的镜像命名
- 预览部署也会使用固定的 `build` / `latest` tag，不同 PR 可能互相覆盖

### 4.2 分支 1 忽略 pull_request_id

分支 1 中预览部署和正式部署使用相同的镜像 tag（`build` / `latest`），这对于多 PR 并行预览是一个隐患。

### 4.3 dockerimage 不在 BuildPackTypes 枚举中

`dockerimage` 是一个特殊的 build_pack 值，但没有出现在 `BuildPackTypes` 枚举中，只作为字符串处理。这是历史遗留问题。

### 4.4 被注释掉的 docker_registry_image_tag 覆盖

L1189-L1191 有一段被注释的代码，说明曾经计划支持 `docker_registry_image_tag` 覆盖 commit tag，但最终取消了。现在只有 `dockerimage` 类型支持这个字段。

### 4.5 queue_next_deployment 和 next_after_cancel 逻辑重复

两个函数的核心逻辑几乎完全相同（遍历 server 下 QUEUED 部署，检查 next_queuable 后 dispatch），只是入口参数不同（Application vs Server）。

### 4.6 取消操作的异步性

用户点击取消后，只是更新了数据库状态，Job 需要执行到下一个远程命令检查点才会真正终止。如果 Job 卡在某个不检查取消状态的长操作中（比如构建过程），可能需要等待较长时间才会响应取消。

### 4.7 CheckApplicationDeploymentQueue 标记为 FAILED 而非 CANCELLED

定时命令超时取消的部署被标记为 FAILED，而不是 CANCELLED_BY_USER。这在语义上不一致，可能导致统计偏差。

### 4.8 queue_full 与 next_queuable 之间的 race condition

高并发下可能出现：
1. 部署 A 检查 queue_full → 通过（queued_count = 24）
2. 部署 B 检查 queue_full → 通过（queued_count = 24）
3. 部署 A 创建记录 → queued_count = 25
4. 部署 B 创建记录 → queued_count = 26（超出限制）

但在实际使用中，部署入队通常不是高频操作，这个 race condition 影响较小。
