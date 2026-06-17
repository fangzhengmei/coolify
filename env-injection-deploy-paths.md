# Coolify 环境变量部署路径深度分析

本文档聚焦环境变量在**部署执行阶段**的流动路径，重点理清以下容易混淆的概念：

1. **build-time env vs runtime env** — 两个文件、两个阶段、两套传递方式
2. **CLI 的 `--env-file` vs Compose 文件的 `env_file`** — 两个不同层面的 env-file
3. **`coolify_variables` shell 前缀 vs `.env` 文件** — 两种注入 CLI 环境的方式
4. **Raw Compose 模式** — 哪些环节被跳过、哪些仍然生效
5. **Preserve Repository（保留仓库路径）** — 工作目录切换如何影响 .env 路径
6. **分支（branch）变量** — 为什么它同时出现在三个地方

---

## 一、构建时 vs 运行时：两套独立的 env 体系

### 1.1 时间线概览

```
部署开始
  │
  ├─ generate_buildtime_environment_variables()
  │    └─ 构造 build-time.env 的内容（仅含 is_buildtime=true 的变量）
  │
  ├─ save_buildtime_environment_variables()
  │    └─ 写入 /artifacts/build-time.env
  │
  ├─ docker compose --env-file /artifacts/build-time.env build  ← 构建阶段
  │
  ├─ save_runtime_environment_variables()
  │    └─ 用完整变量覆盖 <workdir>/.env（含 runtime-only 变量）
  │
  └─ docker compose --env-file <workdir>/.env up -d          ← 启动阶段
```

### 1.2 build-time.env：在 `/artifacts` 目录下的特殊文件

**生成函数：** [save_buildtime_environment_variables()](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/app/Jobs/ApplicationDeploymentJob.php#L1812-L1848)

**文件路径：** `/artifacts/build-time.env`（常量 `self::BUILD_TIME_ENV_PATH`，见 [L46](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/app/Jobs/ApplicationDeploymentJob.php#L46-L46)）

**为什么放在 `/artifacts` 而不是工作目录？**

> 代码注释写得很清楚："outside Docker context to prevent it from being in the image"
> — 放在 Docker 构建上下文之外，防止 `.env` 文件被意外 COPY 进镜像造成机密泄露。

**包含哪些变量？** 由 [generate_buildtime_environment_variables()](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/app/Jobs/ApplicationDeploymentJob.php#L1593-L1810) 生成：

| 变量来源 | 优先级 | 说明 |
|---------|-------|------|
| Nixpacks plan 自动检测 | 最低 | 仅 nixpacks 构建器 |
| `COOLIFY_*` 系统变量 | 中 | `generate_coolify_env_variables(forBuildTime: true)` |
| `SERVICE_*` 魔术变量 | 较高 | 从 Compose 解析 |
| 用户 `is_buildtime=true` 的变量 | 最高 | 数据库中标记为构建时可用的 |

**哪些 COOLIFY_* 变量故意不进入构建阶段？**

[generate_coolify_env_variables()](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/app/Jobs/ApplicationDeploymentJob.php#L2942-L3037) 的 `forBuildTime` 参数控制：

- ❌ `SOURCE_COMMIT` — 默认不进构建（每次 commit 都变，会破坏 Docker 缓存），除非用户在设置中开启 `include_source_commit_in_build`
- ❌ `COOLIFY_CONTAINER_NAME` — 不进构建（每次部署都变，破坏缓存）

### 1.3 runtime .env：工作目录下的 `.env`

**生成函数：** [save_runtime_environment_variables()](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/app/Jobs/ApplicationDeploymentJob.php#L1484-L1591)

**文件路径：** `<workdir>/.env`

**写入时机：构建完成之后**（[L778-L780](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/app/Jobs/ApplicationDeploymentJob.php#L778-L780)）

> **关键设计**：先构建（用 build-time.env），构建成功后再用完整 runtime 变量覆盖 `.env`。
> 这样做的好处是 runtime-only 的变量（如密钥、连接串等）不会出现在构建阶段的 Docker layer 中。

### 1.4 非 Compose 构建器的构建时变量传递

对于 `dockerfile` / `nixpacks` / `railpack` 等非 Compose 构建器，构建时环境变量通过 `ARG` 指令注入 Dockerfile：

```
save_buildtime_environment_variables()   ← 生成 .env 文件（也用于后续步骤）
  → generate_build_env_variables()       ← 生成 build_args 集合
    → add_build_env_variables_to_dockerfile()   ← 修改 Dockerfile 加 ARG 行
```

**修改 Dockerfile 的逻辑：** [add_build_env_variables_to_dockerfile()](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/app/Jobs/ApplicationDeploymentJob.php#L4130-L4232)

- 在 `FROM` 之后、`RUN` 之前插入 `ARG VAR=value`
- 使用 build secrets 模式时跳过 ARG 注入（改为 `--secret` 传递）

---

## 二、`--env-file`（CLI 参数） vs `env_file`（Compose 字段）

这是最容易混淆的概念。它们名字类似、都叫 env-file，但**作用层面完全不同**。

### 2.1 三层环境变量体系

```
┌──────────────────────────────────────────────────────────────────┐
│  第 1 层：Docker Compose CLI 的 Shell 环境                        │
│  （docker compose 命令执行时所在 shell 的环境变量）               │
│                                                                   │
│  来源：                                                            │
│    - $ coolify_variables 前缀                                      │
│    - 主机本身的环境变量                                            │
│    - --env-file 参数指定的文件                                     │
│                                                                   │
│  作用：解析 docker-compose.yml 中的 ${VAR} 引用                    │
│  优先级（Docker Compose 自身规则）：                               │
│    shell 环境 > --env-file > .env（项目根目录自动加载）            │
└───────────────────────────────┬───────────────────────────────────┘
                                │
                                ▼
┌──────────────────────────────────────────────────────────────────┐
│  第 2 层：Compose 文件 services.<name>.environment 内联值          │
│                                                                   │
│  作用：直接定义容器环境变量（优先级最高）                           │
│  注意：这里写的 ${VAR} 是用第 1 层的环境来解析的                    │
└───────────────────────────────┬───────────────────────────────────┘
                                │
                                ▼
┌──────────────────────────────────────────────────────────────────┐
│  第 3 层：Compose 文件 services.<name>.env_file 指定的文件         │
│                                                                   │
│  作用：作为容器环境变量的来源之一                                   │
│  优先级：低于第 2 层的内联值                                       │
└──────────────────────────────────────────────────────────────────┘
```

### 2.2 Coolify 中两层 env-file 的使用

#### CLI 层的 `--env-file`

代码位置：
- 构建命令：[L760](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/app/Jobs/ApplicationDeploymentJob.php#L760-L760) — `--env-file /artifacts/build-time.env`
- 启动命令：[L832](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/app/Jobs/ApplicationDeploymentJob.php#L832-L832) — `--env-file <server_workdir>/.env`
- 自定义命令自动注入：[injectDockerComposeFlags()](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/bootstrap/helpers/docker.php#L1414-L1433)

**作用**：让 docker compose 命令在解析 YAML 文件中的 `${VAR}` 引用时，能从这个文件里找值。

#### Compose 文件层的 `env_file`

代码位置：[parsers.php L1422-L1428](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/bootstrap/helpers/parsers.php#L1422-L1428)

```php
$existingEnvFiles = data_get($service, 'env_file');
$envFiles = collect(...)
    ->push('.env')    // Coolify 自动追加 .env
    ->unique()
    ->values();
$payload['env_file'] = $envFiles;
```

**作用**：让每个服务的容器从 `.env` 文件读取环境变量。

### 2.3 为什么两个层面都要用？

| 层面 | 为什么需要 | 如果不用会怎样 |
|-----|-----------|---------------|
| CLI `--env-file` | Compose YAML 里有 `${VAR}` 引用（比如 `image: myimage:${TAG}`、`ports: - "${PORT}:80"`），需要在解析 YAML 时就知道值 | YAML 中的 `${VAR}` 会变成空串或报错 |
| Compose `env_file` | 容器内运行的程序需要读取环境变量 | 容器内拿不到变量值 |

> 可以这样理解：CLI 层的 `--env-file` 是给 **Docker Compose 这个程序** 用的；Compose 文件里的 `env_file` 是给 **你的应用容器** 用的。

---

## 三、`coolify_variables` Shell 前缀：第三条注入路径

除了 `.env` 文件和 `environment` 字段，Coolify 还有第三条注入路径——**Shell 变量前缀**。

### 3.1 代码实现

**设置函数：** [set_coolify_variables()](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/app/Jobs/ApplicationDeploymentJob.php#L2227-L2256)

```php
$this->coolify_variables = '';
$this->coolify_variables .= "SOURCE_COMMIT={$this->commit} ";
$this->coolify_variables .= "COOLIFY_URL={$url} ";
$this->coolify_variables .= "COOLIFY_FQDN={$fqdn} ";
$this->coolify_variables .= 'COOLIFY_BRANCH='.escapeShellValue($this->application->git_branch).' ';
$this->coolify_variables .= "COOLIFY_RESOURCE_UUID={$this->application->uuid} ";
```

**使用方式：** 拼接到 `docker compose` 命令前面

```php
$command = "{$this->coolify_variables} docker compose --env-file ... build";
// 实际执行： COOLIFY_URL=xxx COOLIFY_FQDN=yyy docker compose --env-file .env build
```

### 3.2 为什么需要 Shell 前缀？它和 `--env-file` 什么关系？

**Shell 前缀属于第 1 层（CLI Shell 环境），优先级高于 `--env-file`。**

根据 Docker Compose 的规则：
```
Shell 环境变量（最高）
  → --env-file 参数指定的文件
    → 项目目录下的 .env（自动加载）
```

如果 `--env-file` 文件里也有 `COOLIFY_URL`，Shell 前缀的值会 **覆盖** 它。

**为什么不用 `--env-file` 统一搞定，还要多此一举？**

因为 `coolify_variables` 在整个部署过程中会被多次使用，不只是 docker compose 命令。比如：
- 直接执行的其他 shell 命令也可能需要这些变量
- 它在部署很早期就设置好了（git clone 之前），而 `.env` 文件生成得更晚
- 它总是包含最新值（`COOLIFY_FQDN` 会因预览部署而变化），不受 `.env` 文件是否生成的影响

### 3.3 三个地方都有 COOLIFY_* 变量？

是的，`COOLIFY_URL`、`COOLIFY_FQDN` 等变量同时存在于：

1. **Shell 前缀** `coolify_variables` — 给 docker compose CLI 用（解析 YAML 引用）
2. **`environment:` 内联** — 给容器用（最高优先级，走第 2 层）
3. **`.env` 文件** — 给容器用（通过 `env_file`，走第 3 层）

其中第 2、3 层的变量是通过 `generate_coolify_env_variables()` 生成的，它会检查数据库里是否已有用户定义的同名变量：

```php
if ($this->application->environment_variables->where('key', 'COOLIFY_FQDN')->isEmpty()) {
    $coolify_envs->put('COOLIFY_FQDN', ...);  // 用户没设才加
}
```

**用户定义的同 key 变量优先级更高**，会覆盖系统自动生成的值。

---

## 四、Raw Compose 模式的影响

**开关：** `$this->application->settings->is_raw_compose_deployment_enabled`

### 4.1 Raw Compose 模式下跳过了什么

代码位置：[ApplicationDeploymentJob.php L668-L700](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/app/Jobs/ApplicationDeploymentJob.php#L668-L700)

| 环节 | 非 Raw Compose | Raw Compose | 代码证据 |
|-----|---------------|-------------|---------|
| Compose 解析 | 经过 `parse()` 完整处理 | 直接用 `docker_compose_raw` | L668-670 |
| `env_file` 自动追加 | ✅ 自动给每个服务加 `env_file: ['.env']` | ❌ 不加（用户写的啥就是啥） | L680-686 |
| `environment:` 变量处理 | ✅ 解析 `${VAR}`、识别 `SERVICE_*`、自动补全 | ❌ 完全不处理 | 不经过 parsers.php |
| 共享变量 `{{...}}` 解析 | ✅ 在 parser 中解析 | ❌ 不解析（因为不走 parser） | parsers.php L1301-1306 |
| 构建 secrets 自动配置 | ✅ 自动注入 | ❌ 需要用户手动写 | L672-676 |
| COOLIFY_* 注入 environment | ✅ parser 中加 | ❌ 不加（但 Shell 前缀仍有） | parsers.php L1412 |

### 4.2 Raw Compose 模式下仍然生效的环境变量机制

| 机制 | 是否生效 | 说明 |
|-----|---------|------|
| Shell 前缀 `coolify_variables` | ✅ 是 | `docker compose` 命令前仍然有 `COOLIFY_URL=xxx ...` |
| CLI `--env-file` 参数 | ✅ 是 | 启动命令仍然带 `--env-file <server_workdir>/.env`（[L832](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/app/Jobs/ApplicationDeploymentJob.php#L832-L832)） |
| `.env` 文件生成 | ✅ 是 | `save_runtime_environment_variables()` 仍然执行 |
| 构建时 `build-time.env` | ✅ 是 | 构建命令仍然带 `--env-file /artifacts/build-time.env` |
| 自定义命令自动注入 flag | ✅ 是 | `injectDockerComposeFlags()` 自动加 `-f` 和 `--env-file` |

### 4.3 一个重要区别：SERVICE_* 魔术变量的生成方式

非 Raw Compose 模式下，`SERVICE_URL_*` / `SERVICE_FQDN_*` 是在 parser 阶段从 fqdn 配置生成的，同时写入 `environment:` 字段和 `.env` 文件。

Raw Compose 模式下，parser 不执行，所以：
- ❌ `environment:` 里没有这些变量
- ✅ `.env` 文件里仍然有（由 `generate_runtime_environment_variables()` 生成，见 [L1331-L1346](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/app/Jobs/ApplicationDeploymentJob.php#L1331-L1346)）
- ✅ 容器内仍然能拿到（因为走 `env_file` 路径，第 3 层）

但如果 Raw Compose 的 YAML 里引用了 `${SERVICE_URL_POSTGRES}`，那这个引用是在第 1 层（CLI 环境）解析的——而 CLI 环境的变量来自 Shell 前缀 + `--env-file`，所以 `.env` 里有就**能解析到**。

---

## 五、Preserve Repository（保留仓库路径）的影响

**开关：** `$this->application->settings->is_preserve_repository_enabled`（[L237](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/app/Jobs/ApplicationDeploymentJob.php#L237-L237)）

### 5.1 两个工作目录的区别

| 目录 | 变量名 | 生命周期 | 路径示例 |
|-----|-------|---------|---------|
| 临时构建目录 | `$this->workdir` | 部署结束后可能被清理 | `/tmp/deploy-<uuid>/<base_dir>/` |
| 持久化工作目录 | `$this->application->workdir()` | 跨部署持久保留 | `/var/lib/coolify/applications/<app-uuid>/` |

两者关系：
```php
$this->basedir = $this->application->generateBaseDir($this->deployment_uuid);
$this->workdir = "{$this->basedir}" . rtrim($baseDir, '/');
```

### 5.2 对 `.env` 路径的影响

**构建阶段：永远用 `$this->workdir`**（不管 preserve 是否开启）

```
构建命令路径：
  --env-file /artifacts/build-time.env  ← 固定路径，和 workdir 无关
  --project-directory $this->workdir
  -f $this->workdir/docker-compose.yaml
```

**启动阶段：取决于 preserve 开关**

| 模式 | `.env` 路径 | `--project-directory` | 代码位置 |
|-----|------------|----------------------|---------|
| Preserve=OFF | `$this->workdir/.env` | `$this->workdir` | [L872-L873](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/app/Jobs/ApplicationDeploymentJob.php#L872-L873) |
| Preserve=ON | `$server_workdir/.env` | `$server_workdir` | [L863-L864](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/app/Jobs/ApplicationDeploymentJob.php#L863-L864) |

**`.env` 文件的两个写入位置**

`save_runtime_environment_variables()` 会把 `.env` 同时写入两个位置（[L1558-L1590](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/app/Jobs/ApplicationDeploymentJob.php#L1558-L1590)）：

| 位置 | 路径 | 用途 |
|-----|------|------|
| helper 容器内 | `$this->workdir/.env` | 构建阶段使用，以及 Preserve=OFF 时启动阶段使用 |
| 服务器文件系统 | `$this->configuration_dir/.env` | 配置目录持久化存储 |

Preserve=ON 时，启动命令使用 `$server_workdir/.env` 作为 `--env-file` 路径（[L863](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/app/Jobs/ApplicationDeploymentJob.php#L863-L863)），其中 `$server_workdir = $this->application->workdir()`。

此外，`write_deployment_configurations()` 在 Preserve=ON 时会通过 `docker cp` 把 helper 容器内的构建产物复制到 `configuration_dir`（[L1020-L1032](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/app/Jobs/ApplicationDeploymentJob.php#L1020-L1032)），用于配置持久化。

### 5.3 自定义命令的路径处理

对于用户自定义的 docker compose 命令，Coolify 通过 `injectDockerComposeFlags()` 自动注入正确的 `-f` 和 `--env-file` 路径：

```php
$workdir_path = $this->preserveRepository ? $server_workdir : $this->workdir;
$start_command = injectDockerComposeFlags(
    $this->docker_compose_custom_start_command,
    "{$workdir_path}{$this->docker_compose_location}",
    "{$workdir_path}/.env"
);
```

---

## 六、分支（branch）变量的三个注入点

`COOLIFY_BRANCH` 是一个特殊的变量，它出现在三个地方：

### 6.1 注入点 1：Shell 前缀

```php
// set_coolify_variables()
$this->coolify_variables .= 'COOLIFY_BRANCH='.escapeShellValue($this->application->git_branch).' ';
```

**作用**：让 docker compose CLI 在解析 YAML 时能拿到分支名。

### 6.2 注入点 2：Compose `environment:` 内联

```php
// parsers.php 的 add_coolify_default_environment_variables
if ($resource->environment_variables->where('key', 'COOLIFY_BRANCH')->isEmpty()) {
    $coolifyEnvironments->put('COOLIFY_BRANCH', "\"{$branch}\"");
}
```

**作用**：直接进入容器环境（第 2 层，最高优先级）。
**注意**：用户没定义时才自动加；用户定义了就以用户为准。

### 6.3 注入点 3：`.env` 文件

```php
// generate_coolify_env_variables()
if ($this->application->environment_variables->where('key', 'COOLIFY_BRANCH')->isEmpty()) {
    $coolify_envs->put('COOLIFY_BRANCH', $local_branch);
}
```

**作用**：通过 `env_file` 进入容器（第 3 层）。

### 6.4 为什么需要三处？

| 注入点 | 解决的问题 |
|-------|-----------|
| Shell 前缀 | Compose YAML 中如果有 `${COOLIFY_BRANCH}` 引用（比如用于 image tag），需要 CLI 层就知道值 |
| `environment:` 内联 | 确保容器内一定能拿到，且优先级最高（不会被 `.env` 覆盖） |
| `.env` 文件 | 如果用户没有通过 UI 设置变量，`.env` 里也需要有默认值（Raw Compose 模式或 env_file 路径时依赖它） |

**实际效果**：三个地方的值是一样的（都是 git_branch），所以不存在冲突。但如果用户在 UI 里自定义了 `COOLIFY_BRANCH`，则：
- Shell 前缀的值**仍然是真实分支名**（不受用户设置影响）
- `environment:` 和 `.env` 里的值**是用户设置的值**（用户设置覆盖系统默认）

这是一个**设计细节**：Shell 前缀的 `COOLIFY_BRANCH` 始终反映真实 git 分支（用于 compose YAML 解析），而容器内的 `COOLIFY_BRANCH` 可以被用户覆盖。

---

## 七、完整优先级链（综合所有因素）

### 7.1 对 Compose YAML 解析阶段（第 1 层）

```
优先级从高到低：

1. Shell 前缀变量（coolify_variables）
   如：COOLIFY_URL=xxx COOLIFY_FQDN=yyy docker compose ...
   
2. 命令行 --env-file 指定的文件
   如：docker compose --env-file /artifacts/build-time.env build
   
3. --project-directory 目录下的 .env（自动加载，Docker Compose 原生行为）
   （Coolify 一般不依赖这个，而是显式用 --env-file）
   
4. 主机 Shell 原有的环境变量
   （通常没什么相关变量）
```

### 7.2 对容器内环境变量（第 2 + 3 层）

```
优先级从高到低：

1. services.<name>.environment 内联具体值
   （非 Raw Compose 模式下由 parser 写入，含 COOLIFY_* 等系统变量和用户在模板中定义的值）
   
2. 主机 Shell 环境（如果 YAML 里写了 KEY 而不写值，会继承主机环境）
   （Coolify 基本不用这个模式）
   
3. services.<name>.env_file 列出的文件（按顺序，后者覆盖前者）
   非 Raw Compose 模式下，Coolify 自动追加 '.env'
   
4. Dockerfile 中 ENV 指令
   
5. 容器内 OS 默认值
```

### 7.3 `.env` 文件内部的优先级（同一文件内多次出现，后者覆盖前者）

```
① COOLIFY_* 系统变量                          (最容易被覆盖)
② SERVICE_* 魔术变量
③ 普通用户变量（不引用 ${SERVICE_*}）
④ 引用 ${SERVICE_*} 的用户变量                 (最不会被覆盖)
⑤ 预览部署：预览变量 → 生产回退值
⑥ PORT / HOST 兜底（仅当完全未定义时才追加）
```

---

## 八、关键代码位置索引

| 主题 | 文件 | 行号 |
|-----|------|------|
| BUILD_TIME_ENV_PATH 常量 | [ApplicationDeploymentJob.php](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/app/Jobs/ApplicationDeploymentJob.php) | L46 |
| 生成构建时 env 内容 | [ApplicationDeploymentJob.php](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/app/Jobs/ApplicationDeploymentJob.php) | L1593-L1810 |
| 保存构建时 env 文件 | [ApplicationDeploymentJob.php](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/app/Jobs/ApplicationDeploymentJob.php) | L1812-L1848 |
| 保存运行时 env 文件 | [ApplicationDeploymentJob.php](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/app/Jobs/ApplicationDeploymentJob.php) | L1484-L1591 |
| 生成运行时 env 内容 | [ApplicationDeploymentJob.php](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/app/Jobs/ApplicationDeploymentJob.php) | L1309-L1473 |
| 构建命令 --env-file | [ApplicationDeploymentJob.php](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/app/Jobs/ApplicationDeploymentJob.php) | L754-L776 |
| 启动命令 --env-file（Raw Compose） | [ApplicationDeploymentJob.php](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/app/Jobs/ApplicationDeploymentJob.php) | L830-L836 |
| 启动命令 --env-file（普通模式） | [ApplicationDeploymentJob.php](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/app/Jobs/ApplicationDeploymentJob.php) | L860-L878 |
| set_coolify_variables (Shell 前缀) | [ApplicationDeploymentJob.php](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/app/Jobs/ApplicationDeploymentJob.php) | L2227-L2256 |
| generate_coolify_env_variables (.env & environment) | [ApplicationDeploymentJob.php](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/app/Jobs/ApplicationDeploymentJob.php) | L2942-L3037 |
| Compose 自动追加 env_file | [parsers.php](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/bootstrap/helpers/parsers.php) | L1422-L1428 |
| Raw Compose 分支逻辑 | [ApplicationDeploymentJob.php](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/app/Jobs/ApplicationDeploymentJob.php) | L668-L700 |
| 自动注入 -f 和 --env-file | [docker.php](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/bootstrap/helpers/docker.php) | L1414-L1433 |
| 注入构建 args | [docker.php](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/bootstrap/helpers/docker.php) | L1465-L1492 |
| 构建时 ARG 注入 Dockerfile | [ApplicationDeploymentJob.php](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/app/Jobs/ApplicationDeploymentJob.php) | L4130-L4232 |
| Preserve Repository 切换工作目录 | [ApplicationDeploymentJob.php](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/app/Jobs/ApplicationDeploymentJob.php) | L841-L878 |
| 写入部署配置文件 | [ApplicationDeploymentJob.php](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/app/Jobs/ApplicationDeploymentJob.php) | L1018-L1074 |
