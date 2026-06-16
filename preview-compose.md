# Docker Compose 预览部署资源拓扑分析

## 一、总体架构

Docker Compose 类型应用在预览部署时，整条处理链路为：

```
Webhook/手动触发
  → ApplicationPreview 创建 + generate_preview_fqdn_compose() 生成预览域名
  → queue_application_deployment() 入队
  → ApplicationDeploymentJob::deploy_pull_request()
    → 转发至 deploy_docker_compose_buildpack()
      → Application::parse(pull_request_id) → applicationParser()
        → 服务名添加 -pr-{pr_id} 后缀
        → 卷名添加 -pr-{pr_id} 后缀
        → 网络名 {uuid}-{pr_id}
        → Traefik/Caddy 反代标签注入（基于预览域名）
        → 依赖关系重写（服务名加后缀）
      → 写入 yaml → docker compose up
```

---

## 二、generate_preview_fqdn_compose 域名生成

### 2.1 入口与数据结构

[ApplicationPreview::generate_preview_fqdn_compose()](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/app/Models/ApplicationPreview.php#L126-L204) 在以下时机被调用：

1. **Webhook 触发创建预览记录时**（GitLab/Bitbucket/Gitea 控制器、ProcessGithubPullRequestWebhook）
2. **ApplicationDeploymentJob 初始化时**（[L266-L267](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/app/Jobs/ApplicationDeploymentJob.php#L266-L267)）：每次部署都会重新调用，确保域名是最新的

核心数据来源是 `Application.docker_compose_domains` JSON 字段，结构为：

```json
{
  "app": { "domain": "https://app.example.com" },
  "api": { "domain": "https://api.example.com,https://api2.example.com" },
  "worker": { "domain": "" }
}
```

### 2.2 解析 compose 获取完整服务列表

```php
$parsedServices = $this->application->parse(pull_request_id: $this->pull_request_id);
```

通过 `parse()` 解析 compose 文件后，获取所有服务名。对每个非数据库服务：

1. 去除 PR 后缀（`-pr-{pr_id}`）还原为原始服务名
2. 如果该服务名不在 `docker_compose_domains` 中，则插入空域名条目

> 这保证了 compose 文件中新增的服务也会有域名条目，即使域名初始为空。

### 2.3 占位符替换逻辑

对每个有域名的服务，逐个处理其域名（支持逗号分隔的多域名）：

```
原始域名 → 解析出 scheme/host/port/path
         → 应用 preview_url_template 模板
         → 替换 {{random}} / {{domain}} / {{pr_id}}
         → 组装为完整预览 URL
```

具体替换步骤（[L168-L181](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/app/Models/ApplicationPreview.php#L168-L181)）：

| 步骤 | 操作 | 示例 |
|---|---|---|
| 1 | 从原始域名解析出 host | `app.example.com` |
| 2 | 获取 `preview_url_template` | `pr-{{pr_id}}-{{random}}.{{domain}}` |
| 3 | `{{random}}` → Cuid2 随机串 | `pr-42-cljk9a2b3d.example.com` |
| 4 | `{{domain}}` → 原始 host | `pr-42-cljk9a2b3d.app.example.com` |
| 5 | `{{pr_id}}` → PR 编号 | `pr-42-cljk9a2b3d.app.example.com` |
| 6 | 组装 `scheme://preview_host:port/path` | `https://pr-42-cljk9a2b3d.app.example.com` |

> **关键点**：每个域名的 `{{random}}` 是独立生成的，同一服务的多域名会有不同的随机串。

### 2.4 无域名服务的处理

如果服务在 `docker_compose_domains` 中域名为空或不存在：
- 不会自动生成预览域名
- 保留空字符串条目用于表单绑定

### 2.5 fqdn 聚合

生成完所有服务域名后，将所有域名聚合到 `ApplicationPreview.fqdn` 字段（逗号分隔），供 PR 评论通知等场景读取。

### 2.6 与 generate_preview_fqdn 的区别

| 特性 | `generate_preview_fqdn`（普通应用） | `generate_preview_fqdn_compose`（Compose 应用） |
|---|---|---|
| 域名来源 | `application.fqdn`（单域名） | `docker_compose_domains`（多服务多域名） |
| 域名数量 | 1 个预览域名 | N 个服务 × M 个域名 |
| 存储 | 写入 `preview.fqdn` | 写入 `preview.docker_compose_domains`（JSON）+ `preview.fqdn`（聚合） |
| 数据库服务识别 | 无 | 通过 `isDatabaseImage()` 跳过数据库服务 |

---

## 三、applicationParser 中的预览资源拓扑变换

[applicationParser()](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/bootstrap/helpers/parsers.php#L361-L1494) 是 compose 解析的核心，当 `$pull_request_id != 0` 时会执行一系列变换。

### 3.1 基础网络变换

```php
// L400-L403
$baseNetwork = collect([$uuid]);              // 正式部署
$baseNetwork = collect(["{$uuid}-{$pullRequestId}"]);  // 预览部署
```

预览部署使用 `{application_uuid}-{pr_id}` 作为基础网络名。该网络会被标记为 `external: true` 并添加到顶层 `networks` 定义中。

在 [ApplicationDeploymentJob](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/app/Jobs/ApplicationDeploymentJob.php#L784-L800) 中，部署时会显式创建并连接该网络到 `coolify-proxy`：

```php
$networkId = "{$this->application->uuid}-{$this->pull_request_id}";
// docker network create --attachable '{networkId}'
// docker network connect {networkId} coolify-proxy
```

### 3.2 服务名变换

在 [applicationParser L1443-L1447](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/bootstrap/helpers/parsers.php#L1443-L1447)：

```php
if ($isPullRequest) {
    $serviceName = addPreviewDeploymentSuffix($serviceName, $pullRequestId);
}
// 结果: "app" → "app-pr-42"
```

[addPreviewDeploymentSuffix()](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/bootstrap/helpers/shared.php#L3778-L3781) 的逻辑很简单：`$name . '-pr-' . $pull_request_id`。

这意味着 compose 文件中所有服务在预览部署时都会被重命名，例如：

```yaml
# 正式部署
services:
  app:
    ...
  api:
    ...

# 预览部署 (PR #42)
services:
  app-pr-42:
    ...
  api-pr-42:
    ...
```

### 3.3 容器名变换

在 [applicationParser L686-L690](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/bootstrap/helpers/parsers.php#L686-L690)：

```php
$baseName = generateApplicationContainerName(
    application: $resource,
    pull_request_id: $pullRequestId
);
$containerName = "$serviceName-$baseName";
```

[generateApplicationContainerName()](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/bootstrap/helpers/docker.php#L184-L199) 在预览部署时返回 `{uuid}-pr-{pr_id}`。

最终容器名格式：`{原始服务名}-{uuid}-pr-{pr_id}`，例如 `app-abc123-pr-42`。

### 3.4 依赖关系变换

在 [applicationParser L875-L889](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/bootstrap/helpers/parsers.php#L875-L889)：

```php
if ($isPullRequest) {
    $depends_on->each(function ($dependency, $condition) use ($pullRequestId, $newDependsOn) {
        $dependency = addPreviewDeploymentSuffix($dependency, $pullRequestId);
        // 或
        $condition = addPreviewDeploymentSuffix($condition, $pullRequestId);
    });
}
```

所有 `depends_on` 引用的服务名都会加上 `-pr-{pr_id}` 后缀，确保依赖指向的是同 PR 下的服务。

### 3.5 SERVICE_NAME 环境变量

[generateDockerComposeServiceName()](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/bootstrap/helpers/shared.php#L3783-L3791) 为每个服务生成 `SERVICE_NAME_XXX` 环境变量：

```php
$collection->put(
    'SERVICE_NAME_' . str($serviceName)->replace('-', '_')->replace('.', '_')->upper(),
    addPreviewDeploymentSuffix($serviceName, $pullRequestId)
);
// 结果: SERVICE_NAME_APP=app-pr-42
```

这些变量会被注入到所有服务的 `environment` 中（[L1412](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/bootstrap/helpers/parsers.php#L1412)）。

---

## 四、卷命名变换

### 4.1 Bind Mount 卷

在 [applicationParser L774-L824](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/bootstrap/helpers/parsers.php#L774-L824)：

对于 bind mount（主机路径映射），源路径会经过以下变换：

1. `replaceLocalSource()` 将相对路径替换为绝对路径（`{base_configuration_dir}/applications/{uuid}/...`）
2. 如果 `isPreviewSuffixEnabled`（默认 `true`），追加 `-pr-{pr_id}` 后缀

```
正式: /data/coolify/applications/abc123/data → /app/data
预览: /data/coolify/applications/abc123/data-pr-42 → /app/data
```

> 注意：容器内挂载路径（target）不变，只是主机路径加后缀，使预览部署的数据与正式环境隔离。

`is_preview_suffix_enabled` 可在 `LocalFileVolume` 记录上单独控制，允许某些 bind mount 在预览部署中共享数据。

### 4.2 Named Volume 卷

在 [applicationParser L825-L868](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/bootstrap/helpers/parsers.php#L825-L868)：

命名卷的变换规则：

```
正式: {uuid}_{slugWithoutUuid}
预览: {uuid}_{slugWithoutUuid}-pr-{pr_id}
```

例如：

```yaml
# 正式部署
volumes:
  app_data:
    name: abc123_app_data

# 预览部署 (PR #42)
volumes:
  app_data-pr-42:
    name: abc123_app_data-pr-42
```

同时会在 `LocalPersistentVolume` 表中创建对应记录，并在顶层 `volumes` 定义中注册。

---

## 五、Traefik 反代标签注入

### 5.1 标签注入入口

在 [applicationParser L1256-L1383](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/bootstrap/helpers/parsers.php#L1256-L1383)：

1. 首先生成 Coolify 默认标签（`defaultLabels`）
2. 如果服务有域名且非数据库服务，生成反代标签

### 5.2 Coolify 默认标签

[defaultLabels()](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/bootstrap/helpers/docker.php#L233-L254) 为每个服务注入：

```yaml
labels:
  - coolify.managed=true
  - coolify.version=x.x.x
  - coolify.applicationId={id}
  - coolify.type=application
  - coolify.name={container_name_slug}
  - coolify.resourceName={app_name_slug}
  - coolify.projectName={project_slug}
  - coolify.serviceName={resource_name_slug}
  - coolify.environmentName={env_slug}
  - coolify.pullRequestId={pr_id}    # 预览部署时 > 0
```

`coolify.pullRequestId` 标签是 `CleanupOrphanedPreviewContainersJob` 识别预览容器的关键标识。

### 5.3 预览域名获取

在 [applicationParser L1222-L1254](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/bootstrap/helpers/parsers.php#L1222-L1254)：

```php
if ($isPullRequest) {
    $preview = $resource->previews()->find($preview_id);
    $docker_compose_domains = collect(json_decode(data_get($preview, 'docker_compose_domains')));
    if ($docker_compose_domains->count() > 0) {
        $found_fqdn = data_get($docker_compose_domains, "$changedServiceName.domain");
    }
}
```

直接从 `ApplicationPreview.docker_compose_domains` 获取当前服务的预览域名。

### 5.4 Traefik 标签生成

[fqdnLabelsForTraefik()](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/bootstrap/helpers/docker.php#L414-L563) 对预览部署的关键区别：

| 参数 | 正式部署 | 预览部署 |
|---|---|---|
| `uuid` | `{application_uuid}` | `{application_uuid}-pr-{pr_id}` |
| `domains` | 来自 `application.fqdn` | 来自 `preview.docker_compose_domains` |
| `service_name` | 无 | compose 服务名（用于标签去重） |

生成的关键标签：

```yaml
labels:
  # 启用 Traefik
  - traefik.enable=true

  # HTTPS 路由
  - traefik.http.routers.https-0-{uuid}-pr-42-{service}.rule=Host(`pr-42-xxx.app.example.com`)
  - traefik.http.routers.https-0-{uuid}-pr-42-{service}.entryPoints=https
  - traefik.http.routers.https-0-{uuid}-pr-42-{service}.tls=true
  - traefik.http.routers.https-0-{uuid}-pr-42-{service}.tls.certresolver=letsencrypt

  # HTTP → HTTPS 重定向
  - traefik.http.routers.http-0-{uuid}-pr-42-{service}.rule=Host(`pr-42-xxx.app.example.com`)
  - traefik.http.routers.http-0-{uuid}-pr-42-{service}.entryPoints=http
  - traefik.http.routers.http-0-{uuid}-pr-42-{service}.middlewares=redirect-to-https

  # 服务端口（如指定）
  - traefik.http.services.https-0-{uuid}-pr-42-{service}.loadbalancer.server.port=3000
```

> **重要**：路由名中包含 `service_name`，这确保同一应用下不同 compose 服务的 Traefik 路由不会冲突。

### 5.5 Caddy 标签（备选代理）

如果服务器使用 Caddy 而非 Traefik，会同时/单独生成 Caddy 格式标签（[fqdnLabelsForCaddy](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/bootstrap/helpers/docker.php)），预览部署中网络名也会带 PR 后缀：`{destination_network}-{pr_id}`。

### 5.6 环境变量中的 SERVICE_FQDN / SERVICE_URL

在 [applicationParser L1176-L1218](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/bootstrap/helpers/parsers.php#L1176-L1218)：

每个有域名的 compose 服务会自动获得：

```
SERVICE_FQDN_{SERVICE_NAME_UPPER} = host-only（无 scheme）
SERVICE_URL_{SERVICE_NAME_UPPER}  = full URL（含 scheme）
```

预览部署时，这些值会被更新为预览域名。

---

## 六、部署执行流程

### 6.1 deploy_pull_request → deploy_docker_compose_buildpack

[ApplicationDeploymentJob::deploy_pull_request()](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/app/Jobs/ApplicationDeploymentJob.php#L2049-L2060) 对 Docker Compose 应用直接转发到 `deploy_docker_compose_buildpack()`。

### 6.2 deploy_docker_compose_buildpack 核心步骤

[deploy_docker_compose_buildpack()](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/app/Jobs/ApplicationDeploymentJob.php#L607-L883) 的核心流程：

1. **准备构建环境**：`prepare_builder_image()` → 启动 helper 容器
2. **克隆代码**：`check_git_if_build_needed()` → `clone_repository()` → `cleanup_git()`
3. **解析 compose**：
   - `loadComposeFile()` — 重新从仓库加载 compose 原始内容
   - `Application::parse(pull_request_id, preview_id, commit)` — 调用 `applicationParser()` 执行上述所有变换
4. **写入 compose 文件**：将解析后的 YAML base64 编码后写入工作目录
5. **创建网络**：
   ```bash
   docker network create --attachable '{uuid}-{pr_id}'
   docker network connect '{uuid}-{pr_id}' coolify-proxy
   ```
6. **启动服务**：`docker compose --project-name {uuid} up -d`
7. **写入配置**：`write_deployment_configurations()` — 持久化 compose 文件到服务器

### 6.3 compose 文件位置

预览部署的 compose 文件名在 [ApplicationDeploymentJob L1053-L1057](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/app/Jobs/ApplicationDeploymentJob.php#L1053-L1057) 变换：

```
正式: docker-compose.yaml
预览: docker-compose-pr-42.yaml
```

---

## 七、资源拓扑总结

### 7.1 命名规则汇总

| 资源 | 正式部署 | 预览部署 (PR #42) |
|---|---|---|
| 服务名 | `app` | `app-pr-42` |
| 容器名 | `app-{uuid}-{timestamp}` | `app-{uuid}-pr-42` |
| 基础网络 | `{uuid}` | `{uuid}-42` |
| Named Volume | `{uuid}_app_data` | `{uuid}_app_data-pr-42` |
| Bind Mount 源路径 | `/data/applications/{uuid}/data` | `/data/applications/{uuid}/data-pr-42` |
| Compose 文件名 | `docker-compose.yaml` | `docker-compose-pr-42.yaml` |
| 预览域名 | `app.example.com` | `pr-42-{random}.app.example.com` |
| Traefik Router UUID | `{uuid}` | `{uuid}-pr-42` |
| COOLIFY_BRANCH | `main` | `pull/42/head` |

### 7.2 多服务 Compose 示例

假设一个 compose 应用有 3 个服务：`app`（Web）、`api`（API）、`db`（PostgreSQL）：

```
正式部署:
  服务: app, api, db
  网络: abc123 (external)
  卷:   abc123_app_data, abc123_db_data
  域名: app.example.com → app, api.example.com → api

预览部署 (PR #42):
  服务: app-pr-42, api-pr-42, db-pr-42
  网络: abc123-42 (external)
  卷:   abc123_app_data-pr-42, abc123_db_data-pr-42
  域名: pr-42-xxx.app.example.com → app-pr-42
        pr-42-yyy.api.example.com → api-pr-42
        (db 为数据库服务，不生成域名和反代标签)
  标签: coolify.pullRequestId=42
        traefik.http.routers.https-0-abc123-pr-42-app.rule=Host(`pr-42-xxx.app.example.com`)
        traefik.http.routers.https-0-abc123-pr-42-api.rule=Host(`pr-42-yyy.api.example.com`)
  依赖: app-pr-42 depends_on api-pr-42, db-pr-42
```

### 7.3 PR 关闭时的资源清理

参考 [ApplicationPreview::forceDeleting](file:///d:/fz/0601-1/solo-dogfeeding/code/95-coolify/app/Models/ApplicationPreview.php#L34-L71)：

1. 重新解析 compose 文件（带 `pull_request_id`）获取所有卷和网络名
2. 逐个执行 `docker volume rm -f` 删除命名卷
3. 逐个执行 `docker network disconnect ... coolify-proxy` + `docker network rm` 删除网络
4. 删除 `persistentStorages` 数据库记录
5. 最终 `forceDelete()` 删除 `ApplicationPreview` 记录本身
