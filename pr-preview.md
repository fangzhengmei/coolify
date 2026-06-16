# Pull Request 预览部署全链路分析

## 一、整体架构概览

Coolify 支持 4 个代码托管平台（GitHub / GitLab / Bitbucket / Gitea）的 Pull Request 事件触发预览部署。整条链路的核心流程为：

```
代码平台 Webhook → Webhook 控制器 → Job/Action → ApplicationPreview 模型 → ApplicationDeploymentJob → ApplicationPullRequestUpdateJob（PR 评论通知）
```

### 关键模型与数据表

| 模型 | 数据表 | 作用 |
|---|---|---|
| `ApplicationPreview` | `application_previews` | 每个 PR 对应一条预览记录，存储 `pull_request_id`、`fqdn`、`status` 等 |
| `ApplicationDeploymentQueue` | `application_deployment_queues` | 部署队列表，`pull_request_id` 字段区分正式部署与预览部署 |
| `ApplicationSetting` | `application_settings` | `is_preview_deployments_enabled` / `is_pr_deployments_public_enabled` 控制预览部署开关 |

---

## 二、Webhook 接收入口

路由定义在 [webhooks.php](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/routes/webhooks.php)：

| 平台 | 路由 | 控制器方法 |
|---|---|---|
| GitHub（App 模式） | `POST /source/github/events` | [Github::normal](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/app/Http/Controllers/Webhook/Github.php#L246-L458) |
| GitHub（手动模式） | `POST /source/github/events/manual` | [Github::manual](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/app/Http/Controllers/Webhook/Github.php#L27-L244) |
| GitLab | `POST /source/gitlab/events/manual` | [Gitlab::manual](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/app/Http/Controllers/Webhook/Gitlab.php#L21-L325) |
| Bitbucket | `POST /source/bitbucket/events/manual` | [Bitbucket::manual](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/app/Http/Controllers/Webhook/Bitbucket.php#L20-L275) |
| Gitea | `POST /source/gitea/events/manual` | [Gitea::manual](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/app/Http/Controllers/Webhook/Gitea.php#L21-L278) |

### GitHub 两种模式差异

- **App 模式（normal）**：通过 `GithubApp` 模型匹配，使用 `repository_project_id` + `source_id` 查找应用，签名校验基于 GitHub App 的 `webhook_secret`。
- **手动模式（manual）**：通过 `full_name`（仓库全名）+ `git_branch` 匹配应用，签名校验基于每个应用单独配置的 `manual_webhook_secret_github`。

### 各平台事件到 Action 的映射

| 平台 | 事件 | Action 值 | 含义 |
|---|---|---|---|
| GitHub | `pull_request` | `opened` / `synchronize` / `reopened` | 创建/更新预览 |
| GitHub | `pull_request` | `closed` / `close` | 关闭预览并清理 |
| GitLab | `merge_request` | `open` / `opened` / `synchronize` / `reopened` / `reopen` / `update` | 创建/更新预览 |
| GitLab | `merge_request` | `closed` / `close` / `merge` | 关闭预览并清理 |
| Bitbucket | `pullrequest:created` / `pullrequest:updated` | — | 创建/更新预览 |
| Bitbucket | `pullrequest:rejected` / `pullrequest:fulfilled` | — | 关闭预览并清理 |
| Gitea | `pull_request` | `opened` / `synchronized` / `reopened` | 创建/更新预览 |
| Gitea | `pull_request` | `closed` | 关闭预览并清理 |

---

## 三、PR 状态与预览部署记录的同步规则

### 3.1 应用匹配逻辑

收到 PR 事件后，控制器根据 **目标分支（base branch）** 匹配应用：

- GitHub App 模式：`Application::where('repository_project_id', $id)->where('source_id', $githubApp->id)->where('git_branch', $base_branch)`
- 手动模式（所有平台）：`Application::where('git_branch', $base_branch)` + 仓库全名过滤

> 注意：匹配的是 `base_branch`（目标分支），不是 PR 的 `head`（源分支）。这是因为应用配置的 `git_branch` 是部署目标分支。

### 3.2 预览部署开关检查

- **创建/更新预览时**：必须满足 `$application->isPRDeployable()` 为 `true`（即 `application_settings.is_preview_deployments_enabled = true`），否则直接拒绝。
- **关闭预览时**：即使预览部署功能被关闭，仍然允许 `closed` action 进入清理流程（见 [Github.php#L206](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/app/Http/Controllers/Webhook/Github.php#L206-L214) 注释："but allow 'closed' action to cleanup"）。

### 3.3 权限与安全校验（仅 GitHub ProcessGithubPullRequestWebhook）

在 [ProcessGithubPullRequestWebhook::handleOpenAction](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/app/Jobs/ProcessGithubPullRequestWebhook.php#L84-L169) 中：

1. **`is_pr_deployments_public_enabled` 为 `false` 时**：
   - Fork PR 永远不会自动部署（因为携带来自不受控仓库的不可信代码）
   - 同仓库 PR 只允许 `OWNER` / `MEMBER` / `COLLABORATOR` 身份触发
2. **`is_pr_deployments_public_enabled` 为 `true` 时**：所有 PR 均可自动部署

> GitLab / Bitbucket / Gitea 的控制器目前没有实现 `isForkPullRequest` 检查，权限控制不如 GitHub 严格。

### 3.4 Watch Path 过滤

在 [ProcessGithubPullRequestWebhook](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/app/Jobs/ProcessGithubPullRequestWebhook.php#L112-L130) 中：

- `synchronize` 事件：通过 GitHub API 获取 `before..after` 之间的变更文件
- `opened` / `reopened` 事件：通过 GitHub API 获取 PR 全部变更文件
- 如果应用配置了 `watch_paths` 且变更文件不匹配，则跳过部署

> GitLab / Bitbucket / Gitea 控制器目前没有实现 watch path 过滤（仅在 push 事件中有）。

### 3.5 Skip Deploy 检查

所有平台都使用 `DetectsSkipDeployCommits` trait 检查 `[skip cd]` / `[skip ci]` 标记：

- GitHub：检查 PR 标题
- GitLab：检查 PR 标题 + 最后一条 commit 消息
- Bitbucket / Gitea：检查 PR 标题

### 3.6 ApplicationPreview 记录创建与 FQDN 生成

当决定创建预览部署时，控制器/Job 先查找是否已有对应记录：

```php
$found = ApplicationPreview::where('application_id', $application->id)
    ->where('pull_request_id', $pull_request_id)
    ->first();
```

- 如果不存在，创建新记录并生成预览域名
- 如果已存在，跳过创建，直接进入部署队列

**域名生成**（[ApplicationPreview](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/app/Models/ApplicationPreview.php#L99-L204)）：

- 普通应用：`generate_preview_fqdn()` 基于主域名 + `preview_url_template`（支持 `{{random}}`、`{{domain}}`、`{{pr_id}}` 占位符）
- Docker Compose 应用：`generate_preview_fqdn_compose()` 为每个服务生成独立域名

### 3.7 部署入队

调用 [queue_application_deployment()](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/bootstrap/helpers/applications.php#L15-L109)：

1. 检查服务器部署队列是否已满
2. **去重检查**：如果同 `application_id` + `commit` + `pull_request_id` + `docker_registry_image_tag` 已有 QUEUED/IN_PROGRESS 的部署且非强制重建，则跳过（返回 `skipped`）
3. 创建 `ApplicationDeploymentQueue` 记录
4. 如果服务器并发数未满，直接派发 `ApplicationDeploymentJob`

### 3.8 部署过程中的 PR 评论同步

[ApplicationDeploymentJob](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/app/Jobs/ApplicationDeploymentJob.php) 在关键节点派发 [ApplicationPullRequestUpdateJob](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/app/Jobs/ApplicationPullRequestUpdateJob.php)：

| 部署状态 | 触发位置 | 评论内容 |
|---|---|---|
| `IN_PROGRESS` | 部署开始时 | "preview deployment is in progress 🟡" |
| `FINISHED` | 部署成功时 | "preview deployment is ready 🟢" + 预览链接 |
| `ERROR` | 部署失败时 | "preview deployment is failed 🔴" |
| `CLOSED` | PR 关闭时（同步派发） | 删除评论 |

评论更新机制：
- 如果已有 `pull_request_issue_comment_id`，则更新该评论
- 如果评论不存在（返回 404），则创建新评论
- 如果还没有评论过，则创建新评论并保存 `comment_id`

---

## 四、重复回调事件的处理

### 4.1 部署队列去重

[queue_application_deployment()](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/bootstrap/helpers/applications.php#L47-L66) 中的核心去重逻辑：

```php
$existing_deployment = ApplicationDeploymentQueue::where('application_id', $application_id)
    ->where('commit', $commit)
    ->where('pull_request_id', $pull_request_id)
    ->where('docker_registry_image_tag', $docker_registry_image_tag)
    ->whereIn('status', [ApplicationDeploymentStatus::IN_PROGRESS->value, ApplicationDeploymentStatus::QUEUED->value])
    ->first();

if ($existing_deployment) {
    if (!$force_rebuild && !$rollback && !$no_questions_asked) {
        return ['status' => 'skipped', 'message' => 'Deployment already queued for this commit.'];
    }
}
```

**去重维度**：`application_id` + `commit` + `pull_request_id` + `docker_registry_image_tag`。

**例外**：`force_rebuild` / `rollback` / `no_questions_asked` 为 `true` 时，不去重，直接创建新部署。

### 4.2 ApplicationPreview 记录去重

在所有平台的控制器和 `ProcessGithubPullRequestWebhook` 中，创建预览记录前都会先查找：

```php
$found = ApplicationPreview::where('application_id', $application->id)
    ->where('pull_request_id', $pull_request_id)
    ->first();
```

如果 `$found` 已存在，则跳过创建，只重新入队部署。这保证了同一个 PR 只会有一条 `ApplicationPreview` 记录。

### 4.3 队列并发控制

[next_queuable()](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/bootstrap/helpers/applications.php#L142-L167) 保证：

- 同一 `application_id` + `pull_request_id` 最多只有 1 个 IN_PROGRESS 部署
- 服务器并发构建数不超过 `concurrent_builds` 设置

### 4.4 GitHub Webhook Job 的重试机制

[ProcessGithubPullRequestWebhook](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/app/Jobs/ProcessGithubPullRequestWebhook.php#L24-L28)：

- `$tries = 3`：最多重试 3 次
- `$backoff = [30, 60, 120]`：指数退避
- `$timeout = 60`：单次执行超时 60 秒

### 4.5 评论幂等

[ApplicationPullRequestUpdateJob](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/app/Jobs/ApplicationPullRequestUpdateJob.php#L61-L65) 使用 `pull_request_issue_comment_id` 实现评论幂等：

- 首次评论时创建，保存返回的 `comment_id`
- 后续状态更新时修改同一评论，而不是创建新评论
- 如果评论被删除（返回 404），自动回退到创建新评论

---

## 五、PR 关闭时的资源清理

### 5.1 入口路径

PR 关闭的清理有两个入口：

1. **Webhook 自动触发**：各平台控制器收到 `closed` / `merge` / `rejected` / `fulfilled` 事件
2. **UI 手动触发**：用户在 Previews 页面点击删除按钮，调用 [Previews::delete](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/app/Livewire/Project/Application/Previews.php#L374-L401)

### 5.2 GitHub 的清理流程（Webhook 自动）

[ProcessGithubPullRequestWebhook::handleClosedAction](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/app/Jobs/ProcessGithubPullRequestWebhook.php#L67-L82)：

```
1. 查找 ApplicationPreview 记录
2. 同步派发 ApplicationPullRequestUpdateJob(status: CLOSED) → 删除 PR 评论
3. 调用 CleanupPreviewDeployment::run() → 执行完整清理
```

### 5.3 GitLab / Bitbucket / Gitea 的清理流程（Webhook 自动）

直接在控制器中调用 [CleanupPreviewDeployment::run()](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/app/Actions/Application/CleanupPreviewDeployment.php#L29-L76)：

```
1. 查找 ApplicationPreview 记录
2. 如果存在，调用 CleanupPreviewDeployment::run()
```

> 注意：这三个平台不会删除 PR 评论（因为它们可能不支持评论 API 或未实现此功能）。

### 5.4 CleanupPreviewDeployment 的三步清理

[CleanupPreviewDeployment](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/app/Actions/Application/CleanupPreviewDeployment.php#L29-L76)：

**Step 1 — 取消活跃部署**（[cancelActiveDeployments](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/app/Actions/Application/CleanupPreviewDeployment.php#L81-L114)）：
- 查找该 PR 的所有 QUEUED / IN_PROGRESS 部署记录
- 将状态改为 `CANCELLED_BY_USER`
- 记录取消日志
- 杀死对应的 helper 容器（通过 `deployment_uuid` 查找 Docker 容器并 `docker rm -f`）

**Step 2 — 停止运行中的 PR 容器**（[stopRunningContainers](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/app/Actions/Application/CleanupPreviewDeployment.php#L137-L175)）：
- Swarm 模式：`docker stack rm {app_uuid}-{pr_id}`
- 非 Swarm 模式：通过 `getCurrentApplicationContainerStatus()` 获取容器列表，逐个 `docker rm -f`

**Step 3 — 派发 DeleteResourceJob**：
- 查找或使用传入的 `ApplicationPreview` 记录
- 派发 `DeleteResourceJob` 进行深度清理

### 5.5 DeleteResourceJob 对 ApplicationPreview 的处理

[DeleteResourceJob::deleteApplicationPreview](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/app/Jobs/DeleteResourceJob.php#L116-L179)：

1. 如果未软删除，先执行软删除（`$this->resource->delete()`）
2. 再次取消活跃部署（与 Step 1 相同逻辑，作为安全兜底）
3. 停止预览容器（Swarm 用 `docker stack rm`，非 Swarm 用 `docker stop` + `docker rm -f`）
4. 执行 `forceDelete()` 触发模型事件

### 5.6 ApplicationPreview::forceDeleting 模型事件

[ApplicationPreview::booted](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/app/Models/ApplicationPreview.php#L32-L77) 注册了 `forceDeleting` 事件：

**Docker Compose 应用**：
- 解析 compose 文件获取卷和网络列表
- 逐个执行 `docker volume rm -f` 删除卷
- 逐个执行 `docker network disconnect` + `docker network rm` 删除网络

**普通应用**：
- 查找关联的 `LocalPersistentVolume` 记录
- 逐个执行 `docker volume rm -f` 删除卷

**最后**：删除 `persistentStorages` 数据库记录

### 5.7 UI 手动删除

[Previews::delete](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/app/Livewire/Project/Application/Previews.php#L374-L401)：

1. 软删除（即时 UI 反馈）
2. 派发 `DeleteResourceJob`（异步深度清理）

> 注意：UI 手动删除没有先调用 `CleanupPreviewDeployment`，而是直接走 `DeleteResourceJob`。`DeleteResourceJob` 内部已包含取消活跃部署和停止容器的逻辑，因此也能完成清理。

### 5.8 孤儿容器定时清理

[CleanupOrphanedPreviewContainersJob](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/app/Jobs/CleanupOrphanedPreviewContainersJob.php) 作为安全网：

- **调度**：每天执行一次（[Kernel.php#L93](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/app/Console/Kernel.php#L93)）
- **原理**：扫描所有功能正常的服务器，查找带有 `coolify.pullRequestId` 标签的容器
- **判定孤儿**：如果 `ApplicationPreview::withTrashed()` 中找不到对应的 `application_id` + `pull_request_id` 记录，则认为是孤儿容器
- **清理**：直接 `docker rm -f` 删除
- **防重入**：使用 `WithoutOverlapping` 中间件，10 分钟超时

---

## 六、完整链路时序图

### 6.1 PR 打开/更新流程（以 GitHub 为例）

```
GitHub Webhook (pull_request: opened/synchronize/reopened)
  │
  ▼
Github::normal() / Github::manual()
  ├─ 签名验证
  ├─ 匹配 Application (base_branch)
  ├─ 检查 isPRDeployable()（closed 除外）
  └─ 派发 ProcessGithubPullRequestWebhook
       │
       ▼
ProcessGithubPullRequestWebhook::handleOpenAction()
  ├─ 检查 isPRDeployable()
  ├─ 检查 skip deploy（PR 标题）
  ├─ 检查 is_pr_deployments_public_enabled + fork/author_association
  ├─ Watch path 过滤
  ├─ 查找/创建 ApplicationPreview
  │    ├─ 不存在 → 创建 + 生成 FQDN
  │    └─ 已存在 → 跳过创建
  └─ queue_application_deployment()
       ├─ 队列满 → 返回 429
       ├─ 去重检查 → 已有同 commit 部署 → 返回 skipped
       └─ 创建 ApplicationDeploymentQueue 记录
            └─ 派发 ApplicationDeploymentJob
                 ├─ IN_PROGRESS → ApplicationPullRequestUpdateJob (评论 "🟡")
                 ├─ FINISHED → ApplicationPullRequestUpdateJob (评论 "🟢" + 链接)
                 └─ ERROR → ApplicationPullRequestUpdateJob (评论 "🔴")
```

### 6.2 PR 关闭流程（以 GitHub 为例）

```
GitHub Webhook (pull_request: closed)
  │
  ▼
Github::normal() / Github::manual()
  ├─ 签名验证
  ├─ 匹配 Application (base_branch)
  ├─ 允许 closed 事件（即使 isPRDeployable 为 false）
  └─ 派发 ProcessGithubPullRequestWebhook
       │
       ▼
ProcessGithubPullRequestWebhook::handleClosedAction()
  ├─ 查找 ApplicationPreview
  ├─ ApplicationPullRequestUpdateJob::dispatchSync(CLOSED)
  │    └─ 删除 PR 评论
  └─ CleanupPreviewDeployment::run()
       ├─ Step 1: cancelActiveDeployments()
       │    ├─ QUEUED/IN_PROGRESS → CANCELLED_BY_USER
       │    └─ 杀死 helper 容器
       ├─ Step 2: stopRunningContainers()
       │    └─ docker rm -f 所有 PR 容器
       └─ Step 3: DeleteResourceJob::dispatch($preview)
            ├─ 软删除（如尚未软删除）
            ├─ 再次取消活跃部署（兜底）
            ├─ 停止预览容器
            ├─ forceDelete() → 触发模型事件
            │    ├─ docker volume rm -f (删除卷)
            │    ├─ docker network rm (删除网络)
            │    └─ persistentStorages()->delete()
            └─ CleanupDocker::dispatch() + cleanup:stucked-resources
```

---

## 七、平台差异汇总

| 特性 | GitHub | GitLab | Bitbucket | Gitea |
|---|---|---|---|---|
| App 模式 | ✅ `normal()` | ❌ | ❌ | ❌ |
| 手动模式 | ✅ `manual()` | ✅ | ✅ | ✅ |
| 异步 Job 处理 | ✅ `ProcessGithubPullRequestWebhook` | ❌（同步） | ❌（同步） | ❌（同步） |
| Fork PR 检测 | ✅ `isForkPullRequest()` | ❌ | ❌ | ❌ |
| `author_association` 校验 | ✅ | ❌ | ❌ | ❌ |
| Watch Path 过滤（PR 事件） | ✅ | ❌ | ❌ | ❌ |
| Skip Deploy（commit 消息） | ✅（push 事件） | ✅（push 事件） | ✅（push 事件） | ✅（push 事件） |
| Skip Deploy（PR 标题） | ✅ | ✅ | ✅ | ✅ |
| PR 评论通知 | ✅ | ❌ | ❌ | ❌ |
| `isPRDeployable` 为 false 时允许 closed | ✅ | ❌（GitLab 未处理 closed） | N/A | N/A |
| 清理使用 `CleanupPreviewDeployment` | ✅（通过 Job） | ✅（直接调用） | ✅（直接调用） | ✅（直接调用） |
