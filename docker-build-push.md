# Docker 镜像构建与推送 Registry 流程解析

本文档对照 Coolify 源代码，详细说明 Docker 镜像从**构建上下文准备** → **缓存复用决策** → **Registry 推送鉴权** → **失败重试处理** 的完整协作路径。

---

## 核心文件定位

| 模块 | 文件 | 关键函数/类 |
|------|------|------------|
| 部署调度 | [ApplicationDeploymentJob.php](file:///d:/fz/0601-2/solo-dogfeeding/code/40-coolify/app/Jobs/ApplicationDeploymentJob.php) | `handle()`、`decide_what_to_do()`、`build_image()`、`push_to_docker_registry()` |
| 远程执行 | [ExecuteRemoteCommand.php](file:///d:/fz/0601-2/solo-dogfeeding/code/40-coolify/app/Traits/ExecuteRemoteCommand.php) | `execute_remote_command()`、`executeCommandWithProcess()` |
| SSH 重试 | [SshRetryable.php](file:///d:/fz/0601-2/solo-dogfeeding/code/40-coolify/app/Traits/SshRetryable.php) | `executeWithSshRetry()`、`isRetryableSshError()`、`calculateRetryDelay()` |
| Docker 辅助 | [docker.php](file:///d:/fz/0601-2/solo-dogfeeding/code/40-coolify/bootstrap/helpers/docker.php) | `generateDockerBuildArgs()`、`generateDockerEnvFlags()`、`escapeBashEnvValue()` |
| 应用设置 | [ApplicationSetting.php](file:///d:/fz/0601-2/solo-dogfeeding/code/40-coolify/app/Models/ApplicationSetting.php) | `$fillable['disable_build_cache']`、`$fillable['use_build_secrets']` |

---

## 一、构建上下文（Build Context）

### 1.1 部署入口与 Job 初始化

部署请求进入 `ApplicationDeploymentJob::__construct()` [L198-L281](file:///d:/fz/0601-2/solo-dogfeeding/code/40-coolify/app/Jobs/ApplicationDeploymentJob.php#L198-L281)，完成以下关键初始化：

```php
// 关键属性初始化
$this->disableBuildCache = $this->application->settings->disable_build_cache;
$this->force_rebuild = $this->application_deployment_queue->force_rebuild;
if ($this->disableBuildCache) {
    $this->force_rebuild = true;  // 缓存禁用 = 强制重建
}
$this->dockerBuildkitSupported = false;  // 后续探测
$this->dockerBuildxAvailable = false;    // 后续探测
```

### 1.2 BuildKit 能力探测

`detectBuildKitCapabilities()` [L410-L482](file:///d:/fz/0601-2/solo-dogfeeding/code/40-coolify/app/Jobs/ApplicationDeploymentJob.php#L410-L482) 在构建前探测目标服务器（或 build server）的 Docker 能力：

| 探测项 | 条件 | 影响 |
|--------|------|------|
| BuildKit 支持 | Docker ≥ 18.09 | 可用 `DOCKER_BUILDKIT=1` 加速构建 |
| Buildx 可用 | `docker buildx version` 成功 | Railpack 必须，可用 `docker-container` 驱动 |
| Build Secrets | `docker build --help` 含 `secret` | 敏感变量通过 `--secret` 传入，不进构建层 |

### 1.3 Helper 容器（构建执行环境）

所有 Docker 构建操作都在 **coolify-helper 容器**内执行，该容器由 `prepare_builder_image()` [L2125-L2168](file:///d:/fz/0601-2/solo-dogfeeding/code/40-coolify/app/Jobs/ApplicationDeploymentJob.php#L2125-L2168) 启动。

**容器挂载关系图：**

```
coolify-helper 容器
├── /var/run/docker.sock    ← 宿主机 Docker Socket（调用 docker build/push）
├── ~/.docker/config.json   ← 宿主机 Registry 登录凭证（只读挂载）
├── ~/.docker/buildx        ← Buildx 构建器元数据卷
└── 宿主机网络 (--network host)  ← 用于拉取代码和 registry 通信
```

**关键启动参数：**

```bash
docker run -d \
  --name ${deployment_uuid} \
  --network ${destination_network} \
  --rm \
  -v ~/.docker/config.json:/root/.docker/config.json:ro \  # 鉴权传递
  -v ~/.docker/buildx:/root/.docker/buildx \               # buildx 状态
  -v /var/run/docker.sock:/var/run/docker.sock \           # Docker-in-Docker
  ${env_flags} \                                            # 构建时环境变量
  ghcr.io/coollabsio/coolify-helper:${version}
```

> **构建服务器（Build Server）模式差异**：使用独立构建服务器时，`config.json` **必须存在**，否则直接抛出 `DeploymentException` 要求先 `docker login`。

### 1.4 工作目录结构

构建过程中涉及三层目录：

| 目录变量 | 生成位置 | 用途 |
|----------|----------|------|
| `$basedir` | `/artifacts` | 构建容器内根目录，存放 build.sh、build-time.env |
| `$workdir` | `$basedir` + `base_directory` | Docker 构建上下文目录，包含源代码、Dockerfile |
| `$configuration_dir` | 宿主机 `application_configuration_dir()` | 持久化配置目录，存放 docker-compose.yaml、.env |

---

## 二、缓存复用机制（Cache Reuse）

Coolify 采用**三层缓存决策**，避免无意义的重复构建。

### 2.1 第一层：Git Commit SHA 命中检查

`check_image_locally_or_remotely()` [L1288-L1307](file:///d:/fz/0601-2/solo-dogfeeding/code/40-coolify/app/Jobs/ApplicationDeploymentJob.php#L1288-L1307) 检查是否已存在相同 commit 的镜像：

```
Step 1: docker images -q ${production_image_name}   # 查本地
Step 2: 若无 && 配置了 registry → docker pull        # 查远端
Step 3: 再查一次本地                                   # pull 成功则有
```

镜像命名规则见 `generate_image_names()` [L1158-L1200](file:///d:/fz/0601-2/solo-dogfeeding/code/40-coolify/app/Jobs/ApplicationDeploymentJob.php#L1158-L1200)：

| 场景 | build_image_name | production_image_name |
|------|-----------------|----------------------|
| 有 registry + 生产 | `registry.io/app:{commit}-build` | `registry.io/app:{commit}` |
| 无 registry + 生产 | `{app_uuid}:{commit}-build` | `{app_uuid}:{commit}` |
| PR 部署 | `registry.io/app:pr-{id}-{commit}-build` | `registry.io/app:pr-{id}-{commit}` |
| 简单 Dockerfile | `registry.io/app:build` | `registry.io/app:latest` |

### 2.2 第二层：构建配置变更检查

`should_skip_build()` [L1245-L1286](file:///d:/fz/0601-2/solo-dogfeeding/code/40-coolify/app/Jobs/ApplicationDeploymentJob.php#L1245-L1286) 调用 `pendingDeploymentConfigurationDiff()->requiresBuild()` 判断：

- 若**镜像存在** + **构建配置未变更** → 跳过 build，直接推送 + 滚动更新
- 若**镜像存在** + **构建配置变更** → 继续构建
- 若**镜像不存在** → 继续构建

### 2.3 第三层：Docker 构建层缓存

`build_image()` [L3517-L3839](file:///d:/fz/0601-2/solo-dogfeeding/code/40-coolify/app/Jobs/ApplicationDeploymentJob.php#L3517-L3839) 根据不同构建包（Buildpack）生成差异化的缓存参数：

#### 2.3.1 Dockerfile / Nixpacks 构建

```bash
# 正常构建（使用 BuildKit 层缓存）
DOCKER_BUILDKIT=1 docker build \
  --network host \
  --progress plain \
  --pull \                          # 拉取最新基础镜像
  -f ${dockerfile_location} \
  --build-arg KEY1 \                # 变量名，值通过环境变量传入
  --build-arg COOLIFY_BUILD_SECRETS_HASH=${hmac_sha256} \  # 防缓存穿透
  -t ${build_image_name} \
  ${workdir}

# 强制重建（disable_build_cache 或 force_rebuild）
DOCKER_BUILDKIT=1 docker build --no-cache --pull ...
```

#### 2.3.2 Nixpacks 专属缓存

Nixpacks 先通过 `--cache-key` 自身管理缓存，再交由 Docker BuildKit：

```bash
nixpacks build \
  -c /artifacts/thegameplan.json \
  --cache-key '${application_uuid}' \    # Nixpacks 层缓存
  --no-error-without-start \
  -n ${build_image_name} \
  ${workdir} \
  -o ${workdir}
```

#### 2.3.3 Railpack Buildx 缓存

Railpack 使用 Buildx 的 `docker-container` 驱动，缓存参数由 `railpack_build_command()` [L2658-L2686](file:///d:/fz/0601-2/solo-dogfeeding/code/40-coolify/app/Jobs/ApplicationDeploymentJob.php#L2658-L2686) 生成：

```bash
# 创建独立构建器（隔离状态）
docker buildx create --name coolify-railpack --driver docker-container 2>/dev/null || true

# 构建命令
docker buildx build --builder coolify-railpack \
  --network host \
  --build-arg BUILDKIT_SYNTAX="ghcr.io/railwayapp/railpack-frontend:v${version}" \
  --build-arg cache-key='${application_uuid}' \
  --build-arg secrets-hash=${hmac_sha256} \
  --secret id=KEY1,env=KEY1 \              # BuildKit Secret
  -f /artifacts/railpack-plan.json \
  --progress plain \
  --load \                                  # 导入到本地 Docker daemon
  -t ${image_name} \
  ${workdir}
```

### 2.4 防止缓存穿透的 Secrets Hash

构建时变量通过 `--build-arg COOLIFY_BUILD_SECRETS_HASH` 传递 hash 而非明文：

```
generate_secrets_hash() [L4086-L4114]:
  输入: 排序后的所有构建时变量 key=value 用 | 连接
  算法: hash_hmac('sha256', $secrets_string, config('app.key'))
  效果: 变量未变 → hash 不变 → Docker 命中缓存；变量变化 → 精确失效
```

### 2.5 缓存旁路开关（ApplicationSetting）

| 设置字段 | 默认值 | 效果 |
|----------|--------|------|
| `disable_build_cache` | `false` | 为 `true` 时全局 `--no-cache` |
| `include_source_commit_in_build` | `false` | 为 `false` 时 SOURCE_COMMIT 不进构建层，避免每次 commit 都破坏缓存 |
| `use_build_secrets` | `false` | 为 `true` 时用 `--secret` 代替 `--build-arg` 传敏感值 |

---

## 三、Registry 推送鉴权（Push Authentication）

### 3.1 鉴权传递链

Coolify **不在代码中处理账号密码**，而是通过文件挂载复用宿主机的 `docker login` 状态：

```
用户在服务器执行: docker login registry.example.com
    ↓
写入 ~/.docker/config.json (auths 字段含 base64 凭证)
    ↓
prepare_builder_image() 以 :ro 模式挂载进 helper 容器
    ↓
容器内 docker push / docker pull 自动读取 /root/.docker/config.json
```

### 3.2 推送执行流程

`push_to_docker_registry()` [L1076-L1132](file:///d:/fz/0601-2/solo-dogfeeding/code/40-coolify/app/Jobs/ApplicationDeploymentJob.php#L1076-L1132) 完整逻辑：

#### 前置条件检查（任一不满足则跳过）

1. `docker_registry_image_name` 未配置 → **跳过**
2. `restart_only` 模式 → **跳过**
3. `build_pack === 'dockerimage'`（直接使用外部镜像，无需构建）→ **跳过**
4. 附加服务器部署（`is_this_additional_server`）→ **跳过**

#### 推送流程

```
Step 1: docker images --format '{{json .}}' ${production_image_name}
        验证镜像存在
Step 2: docker push ${production_image_name}
        推送 commit 标签
Step 3: 若配置了 docker_registry_image_tag + 非 PR 部署
        → docker tag ${production_image_name} {reg}:{user_tag}
        → docker push {reg}:{user_tag}
```

#### `forceFail` 标志位行为

在以下场景中，推送失败将**直接终止部署**并抛出异常（`DeploymentException`，错误码 69420 有特殊语义）：

| 场景 | forceFail | 原因 |
|------|-----------|------|
| 使用独立构建服务器 | `true` | 目标服务器需从 registry 拉取，推送失败则无法部署 |
| Swarm 集群部署 | `true` | Swarm 节点需要从 registry 拉取镜像 |
| 存在 additional_servers | `true` | 附加服务器需要 registry 镜像 |
| 单机部署，无附加服务器 | `false` | 镜像已在本地，推送失败不影响部署 |

> 错误码 `69420` 在 `failed()` [L4879-L4892](file:///d:/fz/0601-2/solo-dogfeeding/code/40-coolify/app/Jobs/ApplicationDeploymentJob.php#L4879-L4892) 中有特殊含义：不会删除新启动的容器，因为 registry 推送失败时新容器可能已经在正常运行。

---

## 四、失败重试机制（Failure Retry）

Coolify 实行**双层重试策略**：Job 层不重试，SSH 命令层精细重试。

### 4.1 Job 级属性（不重试）

```php
public $tries = 1;           // 仅尝试 1 次，不通过队列重试
public $timeout = 3600;      // 最长执行 1 小时
```

设计意图：部署操作通常不可逆，重复执行可能导致状态不一致。重试粒度下沉到**每条 SSH 命令**。

### 4.2 SSH 命令级重试（SshRetryable Trait）

#### 可重试错误判断

`isRetryableSshError()` [SshRetryable.php L12-L54](file:///d:/fz/0601-2/solo-dogfeeding/code/40-coolify/app/Traits/SshRetryable.php#L12-L54) 维护 40+ 种模式：

| 类别 | 示例模式 |
|------|----------|
| 连接中断 | `Connection reset by peer`、`Broken pipe`、`Connection closed by remote host` |
| 网络不可达 | `No route to host`、`Network is unreachable`、`Host is down` |
| 超时 | `Connection timed out`、`Operation timed out`、`Timeout, server not responding` |
| 认证瞬态 | `Permission denied, please try again`、`Too many authentication failures` |
| SSH 协议 | `kex_exchange_identification`、`ssh_exchange_identification`、`Host key verification failed` |
| 资源不足 | `No buffer space available`、`Cannot assign requested address` |

#### 指数退避算法

`calculateRetryDelay()` [SshRetryable.php L58-L68](file:///d:/fz/0601-2/solo-dogfeeding/code/40-coolify/app/Traits/SshRetryable.php#L58-L68)：

```
delay = min(baseDelay * (multiplier ^ attempt), maxDelay)

默认值 (config/constants.php):
  baseDelay = 1s
  multiplier = 2
  maxDelay = 30s
  maxRetries = 5 次

重试序列: 1s → 2s → 4s → 8s → 16s (第 5 次失败后放弃)
```

#### 重试执行流程

`execute_remote_command()` [ExecuteRemoteCommand.php L61-L151](file:///d:/fz/0601-2/solo-dogfeeding/code/40-coolify/app/Traits/ExecuteRemoteCommand.php#L61-L151) 内嵌 while 循环：

```
for attempt = 0; attempt < maxRetries; attempt++:
  try:
    executeCommandWithProcess()  # 通过 SSH 多路复用执行
    break                        # 成功跳出
  catch RuntimeException:
    if isRetryableSshError() && attempt < maxRetries - 1:
      addRetryLogEntry()         # 写入部署日志
      sleep(delay)               # 指数退避等待
      期间检查用户取消 → 抛 69420 中断
    else:
      throw e                    # 非重试错误或已达上限
```

### 4.3 健康检查重试（非 SSH 错误）

应用启动后的健康检查有独立的重试配置：

```
health_check_start_period: 启动宽限期（不检查）
health_check_interval:    每次检查间隔
health_check_retries:     最大重试次数

例: start_period=30s, interval=5s, retries=10
    → 0~30s: 不检查
    → 30s 后每 5s 检查一次，最多连续失败 10 次判定 unhealthy
```

健康检查失败会触发 `failDeployment()` + 旧容器回滚（`rolling_update()` 中处理）。

### 4.4 构建环境变量包装与重试

`wrap_build_command_with_env_export()` [L3512-L3515](file:///d:/fz/0601-2/solo-dogfeeding/code/40-coolify/app/Jobs/ApplicationDeploymentJob.php#L3512-L3515) 确保 build-time 环境变量在构建命令执行前被正确加载：

```bash
cd ${workdir} && \
set -a && source /artifacts/build-time.env && set +a && \
docker build ... ${build_args}
```

这种模式的好处是：**变量插值**（如 `APP_URL=$COOLIFY_URL`）在 shell 层完成，`--build-arg KEY` 只传键名，Docker 自动从环境变量取值。即使 SSH 重试，脚本也能幂等地重新 source。

---

## 五、全流程协作时序图

```
部署请求
  │
  ▼
ApplicationDeploymentJob::handle()
  │
  ├─► 初始化属性 (disable_build_cache → force_rebuild)
  │
  ├─► detectBuildKitCapabilities()
  │     └─ 探测 BuildKit / Buildx / Secrets 支持
  │
  ├─► decide_what_to_do()
  │     │
  │     ├─ deploy_dockerfile_buildpack()  [示例路径]
  │     │     │
  │     │     ├─► prepare_builder_image()
  │     │     │     ├─ 检查 ~/.docker/config.json
  │     │     │     └─ 启动 coolify-helper (挂载凭证+sock)
  │     │     │
  │     │     ├─► clone_repository()
  │     │     │
  │     │     ├─► check_image_locally_or_remotely()
  │     │     │     ├─ docker images -q (本地)
  │     │     │     └─ docker pull + images -q (远端)
  │     │     │
  │     │     ├─► should_skip_build()
  │     │     │     └─ 检查配置变更 → 命中则走 skip 分支
  │     │     │
  │     │     ├─► save_buildtime_environment_variables()
  │     │     │     └─ 写入 /artifacts/build-time.env
  │     │     │
  │     │     ├─► generate_build_env_variables()
  │     │     │     ├─ build_args 或 build_secrets
  │     │     │     └─ generate_secrets_hash() (缓存签名)
  │     │     │
  │     │     ├─► build_image()
  │     │     │     ├─ BuildKit 命令拼接 (含 --no-cache 分支)
  │     │     │     └─ execute_remote_command() [SSH 自动重试 ×5]
  │     │     │
  │     │     ├─► push_to_docker_registry()
  │     │     │     ├─ 条件检查 (forceFail 决策)
  │     │     │     ├─ docker push (凭证来自挂载的 config.json)
  │     │     │     └─ 可选: 额外 tag 推送
  │     │     │
  │     │     └─► rolling_update()
  │     │           ├─ start_by_compose_file() (新容器)
  │     │           ├─ health_check() (健康检查 + 回滚决策)
  │     │           └─ stop_running_container() (旧容器清理)
  │     │
  │     └─► post_deployment()
  │           ├─ completeDeployment() → 状态 FINISHED
  │           └─ 通知 + 附加服务器部署
  │
  └─► finally:
        ├─ graceful_shutdown_container() (清理 helper 容器)
        └─ ServiceStatusChanged 事件广播

异常路径:
  failed(Throwable $e)
    ├─ failDeployment() → 状态 FAILED
    ├─ 写详细错误日志 (类型 + 代码 + 堆栈)
    └─ 非 69420 错误 → 删除新容器
```

---

## 六、关键设计决策总结

| 决策点 | 选择 | 理由 |
|--------|------|------|
| 构建环境 | coolify-helper 容器 + docker.sock | 隔离构建过程，同时复用宿主机 Docker 状态 |
| Registry 鉴权 | 挂载 `~/.docker/config.json` | 不触碰用户凭证明文，兼容所有 `docker login` 支持的 registry 类型 |
| 缓存粒度 | SHA-commit 检查 + 配置 diff + BuildKit 层 | 三层过滤，优先跳过整个 build 过程而非仅复用层 |
| 重试粒度 | SSH 命令级，不超过 Job 级 | 网络瞬态问题自动恢复，避免脏状态的重复部署 |
| 环境变量传递 | `source .env` + `--build-arg KEY`（键名） + `--secret`（敏感值） | 兼顾 shell 插值、Docker 缓存、Secret 安全三类需求 |
| forceFail 策略 | 跨节点场景必失败，单机场景可容忍 | 单机镜像已本地存在，registry 失败不应影响业务可用性 |
