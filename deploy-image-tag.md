# docker_registry_image_tag 去重维度取值规则分析

## 一、四维度去重机制总览

在 [queue_application_deployment()](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/bootstrap/helpers/applications.php#L48-L66) 中，去重查询使用以下四维 key：

```php
$existing_deployment = ApplicationDeploymentQueue::where('application_id', $application_id)
    ->where('commit', $commit)
    ->where('pull_request_id', $pull_request_id)
    ->where('docker_registry_image_tag', $docker_registry_image_tag)
    ->whereIn('status', [IN_PROGRESS, QUEUED])
    ->first();
```

四维去重 key：

| 维度 | 变量 | 默认值/来源 |
|---|---|---|
| 1 | `application_id` | `$application->id` |
| 2 | `commit` | 传入 `$commit`，否则 `$application->git_commit_sha`，否则 `'HEAD'` |
| 3 | `pull_request_id` | 传入 `$pull_request_id`，默认 `0`（正式部署） |
| 4 | `docker_registry_image_tag` | 传入 `$docker_registry_image_tag`，默认 `null` |

**例外条件**（强制跳过去重）：当 `force_rebuild === true` **或** `rollback === true` **或** `no_questions_asked === true` 时，即使查到已有部署记录也会创建新部署。

---

## 二、四种场景下的 docker_registry_image_tag 取值

### 2.1 场景一：常规部署（pull_request_id = 0）

#### 2.1.1 UI 手动部署

[Heading::deploy()](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/app/Livewire/Project/Application/Heading.php#L98-L102)：

```php
queue_application_deployment(
    application: $this->application,
    deployment_uuid: $this->deploymentUuid,
    force_rebuild: $force_rebuild,
    // docker_registry_image_tag 未传 → null
);
```

`docker_registry_image_tag` = **null**

#### 2.1.2 Webhook Push 事件

[Github.php#L148-L154](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/app/Http/Controllers/Webhook/Github.php#L148-L154) / GitLab / Bitbucket：

```php
queue_application_deployment(
    application: $application,
    deployment_uuid: $deployment_uuid,
    force_rebuild: false,
    commit: data_get($payload, 'after', 'HEAD'),
    is_webhook: true,
    // docker_registry_image_tag 未传 → null
);
```

`docker_registry_image_tag` = **null**

#### 2.1.3 UI 重启

[Heading::restart()](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/app/Livewire/Project/Application/Heading.php#L147-L151)：

```php
queue_application_deployment(
    application: $this->application,
    deployment_uuid: $this->deploymentUuid,
    restart_only: true,
    // docker_registry_image_tag 未传 → null
);
```

`docker_registry_image_tag` = **null**

#### 2.1.4 补充服务器部署（自动派发）

[ApplicationDeploymentJob::deploy_additional_destinations()](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/app/Jobs/ApplicationDeploymentJob.php#L2211-L2217)：

```php
queue_application_deployment(
    deployment_uuid: $deployment_uuid,
    application: $this->application,
    no_questions_asked: true,  // 会跳过去重
);
```

`docker_registry_image_tag` = **null**，但 `no_questions_asked=true` 跳过了去重检查。

#### 2.1.5 单服务器重新部署（Destination）

[Destination::redeploy()](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/app/Livewire/Project/Shared/Destination.php#L81-L88)：

```php
queue_application_deployment(
    deployment_uuid: $deployment_uuid,
    application: $this->resource,
    only_this_server: true,
    no_questions_asked: true,  // 跳过去重
);
```

`docker_registry_image_tag` = **null**，但 `no_questions_asked=true` 跳过了去重。

#### 2.1.6 API 创建应用时即时部署

[ApplicationsController](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/app/Http/Controllers/Api/ApplicationsController.php#L1198-L1203)：

```php
queue_application_deployment(
    application: $application,
    deployment_uuid: $deployment_uuid,
    no_questions_asked: true,
    is_api: true,
);
```

`docker_registry_image_tag` = **null**，但 `no_questions_asked=true` 跳过了去重。

### 2.2 场景二：PR 预览部署（pull_request_id > 0）

PR 预览部署中 `docker_registry_image_tag` 的使用取决于 **build_pack** 类型。

#### 2.2.1 非 dockerimage 类型（nixpacks/railpack/dockerfile/dockercompose/static）

**GitHub（通过 ProcessGithubPullRequestWebhook）**

[ProcessGithubPullRequestWebhook::handleOpenAction](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/app/Jobs/ProcessGithubPullRequestWebhook.php#L158-L168)：

```php
queue_application_deployment(
    application: $application,
    pull_request_id: $this->pullRequestId,
    deployment_uuid: $deployment_uuid,
    force_rebuild: false,
    commit: $this->commitSha,
    is_webhook: true,
    git_type: 'github'
    // docker_registry_image_tag 未传 → null
);
```

**GitLab / Bitbucket / Gitea（控制器直接调用）**

[Gitlab.php#L261-L269](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/app/Http/Controllers/Webhook/Gitlab.php#L261-L269) 等：

```php
queue_application_deployment(
    application: $application,
    pull_request_id: $pull_request_id,
    deployment_uuid: $deployment_uuid,
    commit: data_get($payload, 'object_attributes.last_commit.id', 'HEAD'),
    force_rebuild: false,
    is_webhook: true,
    git_type: 'gitlab'
    // docker_registry_image_tag 未传 → null
);
```

**UI 手动部署预览**

[Previews::deploy()](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/app/Livewire/Project/Application/Previews.php#L276-L283)：

```php
queue_application_deployment(
    application: $this->application,
    deployment_uuid: $this->deployment_uuid,
    force_rebuild: $force_rebuild,
    pull_request_id: $pull_request_id,
    git_type: $found->git_type ?? null,
    docker_registry_image_tag: $docker_registry_image_tag,
    // 非 dockerimage 类型 → $docker_registry_image_tag = null
);
```

**结论**：非 dockerimage 类型的预览部署中，`docker_registry_image_tag` 始终为 **null**。

#### 2.2.2 dockerimage 类型（Docker Image 应用）

**Webhook 入口**

所有平台 Webhook 中，dockerimage 类型的 PR 预览部署仍使用 `null` 作为 tag（与非 dockerimage 相同）。

**但在 ApplicationDeploymentJob 构造阶段会补充读取**

[ApplicationDeploymentJob::__construct](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/app/Jobs/ApplicationDeploymentJob.php#L262-L264)：

```php
if ($this->application->build_pack === 'dockerimage' && str($this->dockerImagePreviewTag)->isEmpty()) {
    $this->dockerImagePreviewTag = $this->preview?->docker_registry_image_tag;
}
```

如果队列中 `docker_registry_image_tag` 为空，则从 `ApplicationPreview.docker_registry_image_tag` 读取——这是**运行期补充**，不影响入队时的去重判断（因为去重发生在入队时，不是 Job 执行时）。

**UI 手动部署预览（dockerimage 类型）**

[Previews::deploy()](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/app/Livewire/Project/Application/Previews.php#L276-L283)：

```php
// 由 force_deploy_without_cache 或 add_and_deploy 传入
docker_registry_image_tag: $docker_registry_image_tag
// 从 ApplicationPreview 记录读取（如果是 dockerimage 类型）
```

[Previews::force_deploy_without_cache()](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/app/Livewire/Project/Application/Previews.php#L239-L246)：

```php
$dockerRegistryImageTag = null;
if ($this->application->build_pack === 'dockerimage') {
    $dockerRegistryImageTag = $this->application->previews()
        ->where('pull_request_id', $pull_request_id)
        ->value('docker_registry_image_tag');
}
```

**API Deploy 入口（dockerimage 类型）**

[DeployController::by_uuids](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/app/Http/Controllers/Api/DeployController.php#L409-L421)：

```php
if ($pr !== 0) {
    if ($resource instanceof Application && $resource->build_pack === 'dockerimage') {
        $preview = $this->upsertDockerImagePreview($resource, $pr, $dockerTag);
        $dockerTagForResource = $preview?->docker_registry_image_tag;
    }
}
// ...
$result = $this->deploy_resource($resource, $force, $pr, $dockerTagForResource);
```

[DeployController::deploy_resource](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/app/Http/Controllers/Api/DeployController.php#L515-L522)：

```php
$result = queue_application_deployment(
    application: $resource,
    deployment_uuid: $deployment_uuid,
    force_rebuild: $force,
    pull_request_id: $pr,
    docker_registry_image_tag: $docker_registry_image_tag,
);
```

**API DockerImagePreview 独立创建接口**

[DeployController::upsertDockerImagePreview](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/app/Http/Controllers/Api/DeployController.php#L577-L603)：

```php
// 创建时保存
ApplicationPreview::create([
    'docker_registry_image_tag' => $dockerTag,
]);
// 更新时同步
if ($dockerTag !== null && $preview->docker_registry_image_tag !== $dockerTag) {
    $preview->docker_registry_image_tag = $dockerTag;
    $preview->save();
}
```

**结论**：dockerimage 类型预览部署中，`docker_registry_image_tag` = **`ApplicationPreview.docker_registry_image_tag` 字段值**（由用户/API 在创建预览时指定的 Docker 镜像 tag）。

### 2.3 场景三：强制重建（force_rebuild = true）

#### 2.3.1 UI 强制重建（无缓存）

[Heading::force_deploy_without_cache()](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/app/Livewire/Project/Application/Heading.php#L66-L71)：

```php
$this->deploy(force_rebuild: true);
```

入参：`force_rebuild = true`，`docker_registry_image_tag = null`

#### 2.3.2 UI 预览强制重建

[Previews::force_deploy_without_cache()](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/app/Livewire/Project/Application/Previews.php#L235-L246)：

```php
$this->deploy($pull_request_id, $pull_request_html_url, force_rebuild: true, docker_registry_image_tag: $dockerRegistryImageTag);
```

入参：`force_rebuild = true`，`docker_registry_image_tag = 预览记录中的值`

#### 2.3.3 API 强制部署

[DeployController::deploy()](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/app/Http/Controllers/Api/DeployController.php#L369)：

```php
$force = $request->input('force') ?? false;
```

#### 2.3.4 对去重的影响

在 [queue_application_deployment()](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/bootstrap/helpers/applications.php#L55-L66) 中：

```php
if ($existing_deployment) {
    if (! $force_rebuild && ! $rollback && ! $no_questions_asked) {
        return ['status' => 'skipped', ...];
    }
    // force_rebuild 为 true，会继续创建新部署
}
```

**关键**：`force_rebuild = true` 时，即使四维 key 完全匹配已有 QUEUED/IN_PROGRESS 部署，也**不会跳过**，会创建新的部署记录。但 `docker_registry_image_tag` 的值本身仍然参与写入 `ApplicationDeploymentQueue` 表。

### 2.4 场景四：回滚（rollback = true）

#### 2.4.1 UI 回滚操作

[Rollback::rollbackImage()](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/app/Livewire/Project/Application/Rollback.php#L49-L63)：

```php
$commit = validateGitRef($commit, 'rollback commit');
queue_application_deployment(
    application: $this->application,
    deployment_uuid: $deployment_uuid,
    commit: $commit,
    rollback: true,
    force_rebuild: false,
    // docker_registry_image_tag 未传 → null
);
```

入参：`rollback = true`，`force_rebuild = false`，`commit = 目标回滚的 commit SHA`，`docker_registry_image_tag = null`

#### 2.4.2 对去重的影响

与 `force_rebuild` 一样，`rollback = true` 也会跳过去重检查：

```php
if (! $force_rebuild && ! $rollback && ! $no_questions_asked) {
    // 跳过
}
// rollback 为 true，创建新部署
```

**核心语义**：回滚的 `commit` 通常是一个**历史 commit SHA**（不同于当前 HEAD），但即使四维 key 恰好与某个正在进行的部署完全匹配（理论上可能发生在快速回滚时），也不会被去重拦截。

---

## 三、docker_registry_image_tag 在 Job 执行期的解析

虽然入队去重时 `docker_registry_image_tag` 只是一个静态值，但在 [ApplicationDeploymentJob](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/app/Jobs/ApplicationDeploymentJob.php) 执行过程中，它决定了实际要使用的 Docker 镜像名和 tag。

### 3.1 Job 初始化

[__construct](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/app/Jobs/ApplicationDeploymentJob.php#L222-L223)：

```php
$this->dockerImagePreviewTag = $this->application_deployment_queue->docker_registry_image_tag;
$this->validateDockerRegistryImageConfiguration();  // 校验 tag 格式
```

如果是预览部署且是 dockerimage 类型，而队列中 tag 为空，则补充从预览记录读取：

```php
if ($this->application->build_pack === 'dockerimage' && str($this->dockerImagePreviewTag)->isEmpty()) {
    $this->dockerImagePreviewTag = $this->preview?->docker_registry_image_tag;
}
```

### 3.2 resolveDockerImageTag（仅 dockerimage buildpack）

[resolveDockerImageTag()](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/app/Jobs/ApplicationDeploymentJob.php#L594-L605)：

```php
private function resolveDockerImageTag(): string
{
    // 1. 预览部署且有 preview tag → 用 preview tag
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

### 3.3 generate_image_names（非 dockerfile / 非 dockerimage buildpack）

[generate_image_names()](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/app/Jobs/ApplicationDeploymentJob.php#L1158-L1200) 根据 build_pack 和场景选择不同的 image tag 策略：

| 场景 | `dockerImageTag` 计算 | 镜像命名模式 |
|---|---|---|
| **dockerfile**（正式+预览共用） | 固定 `build` / `latest` | `{name}:build`, `{name}:latest` |
| **dockerimage**（正式部署） | `resolveDockerImageTag()` → 通常 `latest` | `{image}:{tag}` |
| **dockerimage**（预览部署） | `resolveDockerImageTag()` → `dockerImagePreviewTag` | `{image}:{preview_tag}` 或 `{image}@sha256:{hash}` |
| **预览部署**（非 dockerfile/dockerimage） | `previewImageTag()` → `pr-{pr_id}-{commit}` | `{name}:pr-{pr_id}-{commit}` |
| **正式部署**（非 dockerfile/dockerimage） | `substr($commit, 0, 128)` | `{name}:{commit}` |

### 3.4 previewImageTag 算法

[previewImageTag()](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/app/Jobs/ApplicationDeploymentJob.php#L1202-L1221)：

```php
private function previewImageTag(bool $build = false): string
{
    $prefix = "pr-{$this->pull_request_id}-";
    $suffix = $build ? '-build' : '';
    $maxCommitLength = max(1, 128 - strlen($prefix) - strlen($suffix));
    $commitSource = ($this->commit === 'HEAD' || blank($this->commit))
        ? $this->deployment_uuid
        : $this->commit;

    $commit = Str::of($commitSource)
        ->replaceMatches('/[^A-Za-z0-9_.-]/', '-')
        ->substr(0, $maxCommitLength)
        ->toString();

    return "{$prefix}{$commit}{$suffix}";
}
```

格式：`pr-{pr_id}-{sanitized_commit}[ -build]`

- commit 为空或 `'HEAD'` 时，用 `deployment_uuid` 代替
- 非法字符替换为 `-`
- 总长度不超过 128 字符（Docker tag 限制）

---

## 四、同一 PR 多次推送 commit 时的入队判定分析

### 4.1 非 dockerimage 类型预览

去重四维 key：`application_id` + `commit` + `pull_request_id` + `docker_registry_image_tag(null)`

| 情况 | commit | pull_request_id | docker_registry_image_tag | 已有 QUEUED/IN_PROGRESS | 判定结果 |
|---|---|---|---|---|---|
| PR #42 首次推送 commitA | commitA | 42 | null | 无 | ✅ 正常入队 |
| PR #42 首次推送 commitA（快速重复 Webhook） | commitA | 42 | null | 有（同 commitA） | ❌ 跳过：重复回调去重生效 |
| PR #42 推送新 commitB | commitB | 42 | null | 有（commitA 还在部署） | ✅ 正常入队（commit 不同） |
| PR #42 推送 commitB（commitA 已部署完成） | commitB | 42 | null | 无 | ✅ 正常入队 |
| PR #42 commitA 正在部署时再推送 commitA | commitA | 42 | null | 有 | ❌ 跳过（同一个 commit 的重复事件） |
| 同时有 PR #42 和 PR #43 推相同 commit | commitA | 42 / 43 | null | 都无 | ✅ 各入各的队（pull_request_id 不同） |

### 4.2 dockerimage 类型预览

四维 key：`application_id` + `commit` + `pull_request_id` + `docker_registry_image_tag`

由于 dockerimage 类型没有 `commit` 的概念（镜像已经预构建好了），commit 通常是 `'HEAD'` 或固定值。真正的去重维度差异来自 **`docker_registry_image_tag`**：

| 情况 | commit | pull_request_id | docker_registry_image_tag | 已有部署 | 判定结果 |
|---|---|---|---|---|---|
| PR #42 部署 tag `v1.0` | HEAD | 42 | v1.0 | 无 | ✅ 入队 |
| PR #42 快速重复部署 tag `v1.0` | HEAD | 42 | v1.0 | 有 | ❌ 跳过（四维全相同） |
| PR #42 更新到 tag `v1.1` | HEAD | 42 | v1.1 | 有（v1.0） | ✅ 入队（tag 不同） |
| PR #42 同 tag `v1.0` 但 force_rebuild | HEAD | 42 | v1.0 | 有 | ✅ 入队（force_rebuild 跳过去重） |
| PR #42 通过 Webhook 触发（无 tag） | HEAD | 42 | null | 无 | ✅ 入队，但 Job 执行期会从 preview 记录取实际 tag |

### 4.3 Webhook 重复回调的实际防护效果

**场景：GitHub 同一 synchronize 事件发送了两次（网络重放）**

```
第 1 次 Webhook:
  commit = abc123, pull_request_id = 42
  → 查不到 existing → 创建部署记录(status=IN_PROGRESS) → 派发 Job

第 2 次 Webhook（毫秒后到达）:
  commit = abc123, pull_request_id = 42
  → 查到 existing(status=IN_PROGRESS) → 返回 skipped
```

**场景：同一 PR 连续推送 3 个 commit**

```
事件1（synchronize, commit=A）:
  → 创建部署A（IN_PROGRESS）

事件2（synchronize, commit=B，部署A还未结束）:
  → commit 不同 → 创建部署B（QUEUED，等待next_queuable）

事件3（synchronize, commit=C，部署A还未结束）:
  → commit 不同 → 创建部署C（QUEUED）

部署A完成后:
  → queue_next_deployment() → 部署B → 部署C
```

**结论**：commit 维度保证了"同一个代码版本不重复部署"，但不会拦截真正的新版本部署。

### 4.4 与 next_queuable 的协同

[next_queuable()](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/bootstrap/helpers/applications.php#L142-L167) 控制并发：

```
同一 application_id + pull_request_id 同时最多 1 个 IN_PROGRESS 部署
```

这与四维去重形成**双层防护**：
- **四维去重**：防止"同一个 (app, commit, pr, tag)"重复入队
- **next_queuable**：防止"同一个 (app, pr)"并发执行多个不同 commit 的部署

---

## 五、各调用点取值汇总表

| 调用位置 | pull_request_id | commit | docker_registry_image_tag | force_rebuild / rollback / no_questions_asked |
|---|---|---|---|---|
| **Heading::deploy** | 0 | 默认 | null | `force_rebuild` 用户可选 |
| **Heading::restart** | 0 | 默认 | null | `restart_only=true` |
| **Heading::force_deploy_without_cache** | 0 | 默认 | null | `force_rebuild=true` |
| **Rollback::rollbackImage** | 0 | 用户指定历史 commit | null | `rollback=true` |
| **Destination::redeploy** | 0 | 默认 | null | `no_questions_asked=true` |
| **Previews::deploy** | >0 | 默认 | dockerimage 类型从 preview 读取，否则 null | `force_rebuild` 用户可选 |
| **Previews::add_and_deploy** | >0 | 默认 | 同 Previews::deploy | 无 |
| **GitHub Push Webhook** | 0 | `payload.after` | null | 无 |
| **GitHub PR Webhook (Job)** | >0 | `pull_request.head.sha` | null | 无 |
| **GitLab Push Webhook** | 0 | `payload.after` | null | 无 |
| **GitLab MR Webhook** | >0 | `last_commit.id` | null | 无 |
| **Bitbucket Push/PR Webhook** | 0/>0 | payload 中提取 | null | 无 |
| **Gitea Push/PR Webhook** | 0/>0 | payload 中提取 | null | 无 |
| **API Deploy (by_uuids)** | 0/>0 | 默认 | PR+dockerimage 时从 preview 读取 | `force` 用户可选 |
| **API 创建应用即时部署** | 0 | 默认 | null | `no_questions_asked=true` |
| **clone_application 自动恢复** | 0 | 默认 | null | `no_questions_asked=true` |
| **ApplicationDeploymentJob::deploy_additional** | 0 | 默认 | null | `no_questions_asked=true` |
