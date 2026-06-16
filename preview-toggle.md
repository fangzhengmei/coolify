# 预览部署开关切换行为分析

## 一、两个开关的定义与存储

| 开关 | 字段 | 存储位置 | 默认值 | 作用 |
|---|---|---|---|---|
| 预览部署总开关 | `is_preview_deployments_enabled` | `application_settings` 表 | `false` | 控制是否启用 PR 预览部署功能 |
| 公开 PR 部署开关 | `is_pr_deployments_public_enabled` | `application_settings` 表 | `false` | 控制是否允许来自 Fork 或非成员的 PR 自动部署（仅 GitHub 实现） |

两个开关都存储在 `application_settings` 表中，通过 `ApplicationSetting` 模型的 `boolean` cast 访问。在 UI 上通过 [Advanced](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/app/Livewire/Project/Application/Advanced.php) Livewire 组件的 `instantSave()` 方法保存。

---

## 二、is_preview_deployments_enabled 关闭后的各入口响应

### 2.1 Webhook 入口

**GitHub（App 模式 + 手动模式）**

[Github.php#L206](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/app/Http/Controllers/Webhook/Github.php#L206-L214) / [Github.php#L418](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/app/Http/Controllers/Webhook/Github.php#L418-L426)：

```php
if (! $application->isPRDeployable() && $action !== 'closed') {
    return 'Preview deployments are not enabled for this application.';
}
```

- **opened / synchronize / reopened**：直接拒绝，返回 200 和提示信息
- **closed**：**仍然允许通过**，以便执行清理（注释："but allow 'closed' action to cleanup"）
- 闭包后再派发 [ProcessGithubPullRequestWebhook](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/app/Jobs/ProcessGithubPullRequestWebhook.php)

**GitLab**

[Gitlab.php#L229](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/app/Http/Controllers/Webhook/Gitlab.php#L229-L240)：

```php
if ($application->isPRDeployable()) {
    // 创建/更新预览并部署
}
```

- **open / synchronize / reopen**：仅当 `isPRDeployable() === true` 时才执行创建/更新预览和部署
- **closed / merge**：没有检查 `isPRDeployable()`，**直接执行清理**
- 在控制器中同步执行，不通过 Job

**Bitbucket**

[Bitbucket.php#L185](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/app/Http/Controllers/Webhook/Bitbucket.php#L185-L213)：

- **pullrequest:created / pullrequest:updated**：仅当 `isPRDeployable() === true` 时才执行
- **pullrequest:rejected / pullrequest:fulfilled**：未检查 `isPRDeployable()`，直接执行清理

**Gitea**

[Gitea.php#L187](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/app/Http/Controllers/Webhook/Gitea.php#L187-L201)：

- **opened / synchronized / reopened**：仅当 `isPRDeployable() === true` 时才执行
- **closed**：未检查 `isPRDeployable()`，直接执行清理

### 2.2 ProcessGithubPullRequestWebhook 中的二次校验

在 [ProcessGithubPullRequestWebhook::handleOpenAction](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/app/Jobs/ProcessGithubPullRequestWebhook.php#L84-L90) 中还有一层防御性检查：

```php
if (! $application->isPRDeployable()) {
    Log::info('Preview deployments are not enabled for application: '.$application->id);
    return;
}
```

即使 Webhook 控制器漏过了，Job 执行时也会再次检查。

### 2.3 UI 入口（Livewire）

[Previews](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/app/Livewire/Project/Application/Previews.php) 和 [PreviewsCompose](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/app/Livewire/Project/Application/PreviewsCompose.php) 组件**没有检查 `is_preview_deployments_enabled`**：

- `mount()`：直接加载所有预览记录列表
- `add()` / `add_and_deploy()`：可以手动添加预览
- `deploy()`：可以手动触发部署
- `stop()`：可以停止预览容器
- `delete()`：可以删除预览
- `save_preview()` / `generate_preview()`：可以编辑预览域名

路由 [project.application.preview-deployments](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/routes/web.php#L227) 始终可访问。

> **结论**：`is_preview_deployments_enabled` 只影响 Webhook 自动触发，不影响 UI 手动操作。用户仍然可以在 Previews 页面手动添加、部署、停止、删除预览。

### 2.4 API 入口

[DeployController::by_uuids](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/app/Http/Controllers/Api/DeployController.php#L395-L442) 中：

- 传入 `pull_request_id` 参数时，会查找或创建预览记录
- **没有检查 `isPRDeployable()`**
- 只要 API token 有部署权限，就可以触发预览部署

### 2.5 ApplicationDeploymentJob 执行期

[ApplicationDeploymentJob](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/app/Jobs/ApplicationDeploymentJob.php) 在部署执行过程中**不检查** `is_preview_deployments_enabled`。

这意味着：
- 开关关闭前已经入队的部署会继续执行
- 开关关闭后通过 UI/API 手动触发的部署也会正常执行

---

## 三、is_pr_deployments_public_enabled 的校验逻辑

### 3.1 仅 GitHub 实现

`is_pr_deployments_public_enabled` 的检查**只在 GitHub 的 ProcessGithubPullRequestWebhook 中实现**（[L95-L110](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/app/Jobs/ProcessGithubPullRequestWebhook.php#L95-L110)）：

```php
if (! $application->settings->is_pr_deployments_public_enabled) {
    if ($this->webhook_payload['pull_request']['head']['repo']['fork'] === true) {
        Log::info('Fork PRs are not allowed for application: '.$application->id);
        return;
    }
    $authorAssociation = strtolower(data_get($this->webhook_payload, 'pull_request.author_association', ''));
    if (! in_array($authorAssociation, ['owner', 'member', 'collaborator'])) {
        Log::info('PR author is not allowed to trigger preview deployments for application: '.$application->id);
        return;
    }
}
```

| 场景 | `is_pr_deployments_public_enabled = false` | `is_pr_deployments_public_enabled = true` |
|---|---|---|
| 同仓库 PR + OWNER/MEMBER/COLLABORATOR | ✅ 允许 | ✅ 允许 |
| 同仓库 PR + CONTRIBUTOR/NONE 等 | ❌ 拒绝 | ✅ 允许 |
| Fork PR（任何人） | ❌ 拒绝 | ✅ 允许 |

### 3.2 其他平台无此检查

- **GitLab**：控制器中无 `is_pr_deployments_public_enabled` 相关逻辑
- **Bitbucket**：控制器中无此检查
- **Gitea**：控制器中无此检查

> **结论**：`is_pr_deployments_public_enabled` 是 GitHub 专属的安全开关，用于防御来自 Fork PR 的不受信代码自动部署。其他平台的 Webhook 没有实现这个保护。

### 3.3 不影响手动操作

与 `is_preview_deployments_enabled` 类似，`is_pr_deployments_public_enabled` 只影响 Webhook 自动触发，不影响：
- UI 手动部署
- API 手动部署

---

## 四、关闭开关后既有资源的状态

### 4.1 ApplicationPreview 记录

**不会自动删除**。关闭 `is_preview_deployments_enabled` 后：
- 已有的 `ApplicationPreview` 记录保留在数据库中
- 在 Previews 页面仍然可见
- 可以手动删除（触发 `DeleteResourceJob`）

### 4.2 运行中的预览容器

**不会自动停止**。关闭开关后：
- 已经在运行的预览容器继续运行
- `CleanupOrphanedPreviewContainersJob` 不会清理它们（因为有对应的 ApplicationPreview 记录）
- 可以在 UI 上手动 stop 或 delete

### 4.3 PR 评论

**不会自动删除**。关闭开关后：
- 已经发布的 PR 评论保留在 GitHub/GitLab 上
- `ApplicationPullRequestUpdateJob` 不会因为开关关闭而删除评论
- 只有 PR 关闭事件（`closed` action）才会触发评论删除

### 4.4 数据卷与网络

**不会自动清理**。关闭开关后：
- 预览部署创建的 named volume 和网络继续存在
- 只有当预览被显式删除（手动 delete 或 PR closed Webhook）时才会通过 `forceDeleting` 模型事件清理

---

## 五、孤儿容器定时清理 Job 与开关的关系

[CleanupOrphanedPreviewContainersJob](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/app/Jobs/CleanupOrphanedPreviewContainersJob.php) 的判定逻辑与两个开关**完全无关**。

### 5.1 孤儿判定标准

[isOrphanedContainer()](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/app/Jobs/CleanupOrphanedPreviewContainersJob.php#L156-L174)：

```php
$previewExists = ApplicationPreview::withTrashed()
    ->where('application_id', $applicationId)
    ->where('pull_request_id', $pullRequestId)
    ->exists();

return ! $previewExists;
```

唯一判定依据：**`application_previews` 表中是否存在对应记录**（包括软删除的）。

### 5.2 为什么与开关无关

| 场景 | 有 ApplicationPreview 记录 | 无 ApplicationPreview 记录 |
|---|---|---|
| 开关开启，预览正常运行 | ❌ 不是孤儿 | — |
| 开关关闭，预览仍在运行 | ❌ 不是孤儿（记录还在） | — |
| PR 关闭，正常清理中 | ❌ 不是孤儿（软删除了但 withTrashed 能查到） | — |
| 清理失败但记录已删 | — | ✅ 是孤儿，会被清理 |
| 数据库记录丢失 / 手动从数据库删除 | — | ✅ 是孤儿，会被清理 |

> 关键点：`withTrashed()` 意味着即使记录被软删除了，也不算孤儿，应该由 `DeleteResourceJob` 负责清理。只有完全找不到记录的容器才算孤儿。

### 5.3 调度频率

在 [Kernel.php](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/app/Console/Kernel.php) 中每天执行一次。

使用 `WithoutOverlapping` 中间件防止重复执行，10 分钟超时。

---

## 六、开关切换的完整行为矩阵

### 6.1 is_preview_deployments_enabled: true → false

| 资源/行为 | 变化 | 说明 |
|---|---|---|
| Webhook opened/synchronize/reopened | ❌ 被拒绝 | 所有平台均检查 |
| Webhook closed | ✅ 仍可执行 | GitHub/GitLab/Bitbucket/Gitea 均允许 closed 事件清理 |
| UI Previews 页面 | ✅ 仍可访问 | 页面不检查开关 |
| UI 手动 add/deploy/stop/delete | ✅ 仍可操作 | 操作不检查开关 |
| API 部署预览 | ✅ 仍可触发 | API 不检查开关 |
| 已入队的部署 | ✅ 继续执行 | Job 执行期不检查 |
| 已有 ApplicationPreview 记录 | ✅ 保留 | 不会自动删除 |
| 运行中的预览容器 | ✅ 继续运行 | 不会自动停止 |
| PR 评论 | ✅ 保留 | 不会自动删除 |
| 数据卷/网络 | ✅ 保留 | 不会自动清理 |
| 孤儿容器清理 Job | — 不受影响 | 只看记录是否存在，不看开关 |

### 6.2 is_preview_deployments_enabled: false → true

| 资源/行为 | 变化 | 说明 |
|---|---|---|
| Webhook 事件 | ✅ 恢复正常处理 | 新的 PR 事件会正常创建/更新预览 |
| 已有的预览记录 | ✅ 继续有效 | 记录一直都在 |
| 已有的预览容器 | ✅ 继续运行 | 不受影响 |
| 新的 synchronize 事件 | ✅ 会触发新部署 | 开关打开后，PR 更新会重新触发部署 |

### 6.3 is_pr_deployments_public_enabled: true → false

| 场景 | 变化 |
|---|---|
| 同仓库成员 PR | ✅ 不受影响 |
| 同仓库非成员 PR | ❌ 被拒绝 |
| Fork PR | ❌ 被拒绝 |
| 已在运行的预览 | ✅ 不受影响 |
| UI / API 手动部署 | ✅ 不受影响 |
| （仅 GitHub 生效，其他平台无此检查） | — |

---

## 七、Livewire 表单提交流程

### 7.1 Advanced 组件的 instantSave

[Advanced::instantSave()](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/app/Livewire/Project/Application/Advanced.php#L173-L210) 是开关保存的入口：

```
1. authorize('update', $application) — 权限校验
2. 检查 log drain 等依赖项
3. 检查是否需要重置标签（force_https/gzip/stripprefix 变化时）
4. 调用 parse() 或 oldRawParser() 重新解析 compose（如果是 raw compose 模式）
5. syncData(true) — 将 Livewire 属性同步到模型并保存
6. dispatch('success') — 前端提示
7. dispatch('configurationChanged') — 通知其他组件刷新
```

### 7.2 syncData 双向同步

[Advanced::syncData()](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/app/Livewire/Project/Application/Advanced.php#L102-L161)：

- **`$toModel = true`**：Livewire 属性 → 模型 → 保存到数据库
- **`$toModel = false`**：模型 → Livewire 属性（用于初始化和同步）

两个开关的初始值：
```php
$this->isPreviewDeploymentsEnabled = $this->application->settings->is_preview_deployments_enabled;
$this->isPrDeploymentsPublicEnabled = $this->application->settings->is_pr_deployments_public_enabled ?? false;
```

> 注意 `is_pr_deployments_public_enabled` 使用了 `?? false` 兜底，说明该字段可能为 null（迁移较新，老数据可能没有值）。

### 7.3 保存后无清理动作

保存开关后**没有**任何副作用：
- 不停止已有的预览容器
- 不删除已有的预览记录
- 不清理 PR 评论

只是纯粹地修改数据库字段值。

---

## 八、Job 调度链路总结

### 8.1 开关关闭对 Job 调度的影响

```
Webhook (PR opened/synchronize)
  ├─ isPRDeployable() = false
  │    └─ 直接返回，不派发任何 Job
  └─ isPRDeployable() = true
       └─ 派发 ProcessGithubPullRequestWebhook（仅 GitHub）
            ├─ isPRDeployable() 二次校验
            ├─ is_pr_deployments_public_enabled 校验（fork/author）
            └─ 一切通过 → queue_application_deployment() → ApplicationDeploymentJob
                 └─ 过程中派发 ApplicationPullRequestUpdateJob（PR 评论）
```

### 8.2 PR 关闭时的 Job 调度

```
Webhook (PR closed)
  └─ 允许通过（不检查 isPRDeployable）
       ├─ GitHub: ProcessGithubPullRequestWebhook
       │    ├─ ApplicationPullRequestUpdateJob::dispatchSync(status: CLOSED) → 删除评论
       │    └─ CleanupPreviewDeployment::run()
       │         ├─ 取消活跃部署
       │         ├─ 停止容器
       │         └─ DeleteResourceJob::dispatch
       │              ├─ 软删除 → 停止容器 → forceDelete
       │              └─ forceDeleting 模型事件 → 清理卷/网络
       └─ GitLab/Bitbucket/Gitea: 直接调用 CleanupPreviewDeployment
            └─ （同上，但不删除 PR 评论）
```

### 8.3 手动删除预览的 Job 调度

```
UI Delete 按钮
  └─ Previews::delete()
       ├─ $preview->delete() — 软删除（即时 UI 反馈）
       └─ DeleteResourceJob::dispatch($preview)
            └─ 异步执行完整清理
```

---

## 九、潜在盲区与风险点

### 9.1 开关关闭后容器继续运行

`is_preview_deployments_enabled = false` 只是阻止新的 Webhook 触发，已经运行的预览**不会被自动停止**。如果用户以为关闭开关就等于停止所有预览，可能产生意外的资源消耗。

### 9.2 多平台实现不一致

- `is_pr_deployments_public_enabled` 仅 GitHub 实现，GitLab/Bitbucket/Gitea 都没有 fork PR 保护
- GitHub 的 closed 事件通过 Job 异步处理并删除 PR 评论，其他平台同步处理且不删除评论
- GitHub 有 watch path 过滤，其他平台没有

### 9.3 UI/API 绕过开关

UI 和 API 都可以绕过 `is_preview_deployments_enabled` 开关手动部署预览。如果将开关视为安全边界，这是一个漏洞。

### 9.4 孤儿清理不检查软删除时间

`CleanupOrphanedPreviewContainersJob` 使用 `withTrashed()` 查找记录，这意味着：
- 如果 `DeleteResourceJob` 失败且记录停留在软删除状态，容器永远不会被孤儿清理 Job 处理
- 需要依赖 `DeleteResourceJob` 的重试或 `cleanup:stucked-resources` 命令来兜底
