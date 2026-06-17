# Docker Compose CLI 层环境变量插值优先级

本文聚焦 **CLI 层**的环境变量来源和优先级——也就是 Docker Compose 解析 YAML 文件中 `${VAR}` 引用时，变量值从哪里来、谁覆盖谁。

注意与「容器层」环境变量（`services.*.environment` / `services.*.env_file`）区分开，后者决定容器内部能读到什么变量，二者是完全不同层面的概念。

---

## 一、先分清两个层面

| 层面 | 作用对象 | 决定什么 | 典型来源 |
|-----|---------|---------|---------|
| **CLI 层** | Docker Compose 程序本身 | YAML 中 `${VAR}` 引用被替换成什么值 | Shell 环境、`--env-file` 参数、项目目录 `.env` |
| **容器层** | 你的应用容器 | 容器内 `env` 命令能看到什么 | `services.*.environment`、`services.*.env_file`、Dockerfile `ENV` |

本文只讲 **CLI 层**。容器层优先级详见 `env-injection-followup.md`。

---

## 二、Docker Compose 原生优先级（CLI 层）

Docker Compose 在解析 YAML 中的 `${VAR}` 时，按以下优先级查找变量值（高 → 低）：

1. **Shell 环境变量** — 执行 `docker compose` 命令时所在 shell 的环境变量
2. **`--env-file` 指定的文件** — 按命令行上出现的顺序加载，后出现的文件覆盖先出现的
3. **项目目录 `.env`** — 如果没有指定任何 `--env-file`，默认从 `--project-directory`（或当前目录）自动加载 `.env`

关键规则：
- 一旦显式指定了 `--env-file`，**就不会自动加载**项目目录下的 `.env`
- 可以指定多个 `--env-file`，按顺序加载，同名变量后者覆盖前者

---

## 三、Coolify 的 CLI 层 env 来源（共 5 个）

在 Coolify 的部署流程中，CLI 层的 env 来源有 5 个。按优先级从高到低排列：

### 第 1 位：Shell 环境变量（执行环境自带）

执行 `docker compose` 命令的 shell 环境本身就有的变量。比如 helper 容器内预置的环境变量、SSH 登录后的环境变量等。

- **优先级**：最高，能覆盖所有其他来源
- **是否可控**：通常不可控，由服务器/容器环境决定

### 第 2 位：coolify_variables Shell 前缀

Coolify 主动拼接到 `docker compose` 命令前面的 Shell 变量，形如：

```bash
COOLIFY_URL=https://app.example.com COOLIFY_FQDN=app.example.com COOLIFY_BRANCH=main COOLIFY_RESOURCE_UUID=abc123 docker compose ...
```

包含的变量：
- `COOLIFY_URL` / `COOLIFY_FQDN` — 应用的 URL 和域名
- `COOLIFY_BRANCH` — Git 分支名
- `SOURCE_COMMIT` — 提交哈希（仅构建设置开启时）
- `COOLIFY_RESOURCE_UUID` — 应用 UUID

**代码位置**：[ApplicationDeploymentJob.php:set_coolify_variables()](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/app/Jobs/ApplicationDeploymentJob.php#L2227-L2256)

⚠️ **重要限制**：只在**非自定义命令**路径下生效。如果用户写了自定义构建/启动命令，这层前缀**不会被添加**。

### 第 3 位：用户自带的 `--env-file`

如果用户在自定义构建命令或自定义启动命令里自己写了 `--env-file somefile.env`，Coolify 会保留用户的定义，**不会重复注入**。

检测逻辑在 [docker.php:injectDockerComposeFlags()](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/bootstrap/helpers/docker.php#L1414-L1433)：

```php
if (! preg_match('/(?:^|\s)--env-file(?:=|\s)/', $command)) {
    $dockerComposeReplacement .= " --env-file {$envFilePath}";
}
```

用户自带的 `--env-file` 排在 Shell 前缀之后，但在 Coolify 自动注入的 `--env-file` 之前——因为用户写的位置和自动注入的位置是互斥的，**只有一个会生效**（要么用户的，要么 Coolify 自动注入的）。

### 第 4 位：Coolify 自动注入的 `--env-file`

如果命令里没有 `--env-file`，`injectDockerComposeFlags()` 会自动注入一个。不同阶段指向不同文件：

| 阶段 | 注入的 env 文件 |
|-----|---------------|
| 构建阶段 | `/artifacts/build-time.env` |
| 启动阶段 | `<workdir>/.env` 或 `<server_workdir>/.env`（取决于 Preserve Repository） |

**代码位置**：
- 构建（非自定义）：[ApplicationDeploymentJob.php#L754-L765](file:///d:/fz/0601-2\solo-dogfeeding\code\20-coolify\app\Jobs\ApplicationDeploymentJob.php#L754-L765)
- 启动（非自定义非 Raw）：[ApplicationDeploymentJob.php#L860-L877](file:///d:/fz/0601-2\solo-dogfeeding\code\20-coolify\app\Jobs\ApplicationDeploymentJob.php#L860-L877)
- 启动（非自定义 Raw）：[ApplicationDeploymentJob.php#L830-L836](file:///d:/fz/0601-2\solo-dogfeeding\code\20-coolify\app\Jobs\ApplicationDeploymentJob.php#L830-L836)

### 第 5 位：项目目录 `.env`（自动加载）

如果完全没有指定 `--env-file`，Docker Compose 会从 `--project-directory` 指定的目录自动加载 `.env` 文件。

但在 Coolify 的部署流程中，**几乎总是显式指定了 `--env-file`**，所以这个默认加载机制通常不生效。唯一的例外是用户在自定义命令里既没写 `--env-file`，而 `injectDockerComposeFlags()` 因为某些原因也没注入（正常情况不会发生）。

---

## 四、自定义命令如何改变取值来源

这是最容易产生理解偏差的地方。自定义命令**不是简单地换一条命令**，它会改变整个 env 来源的构成。

### 4.1 自定义构建命令

路径代码：[ApplicationDeploymentJob.php#L715-L752](file:///d:/fz/0601-2\solo-dogfeeding\code\20-coolify\app\Jobs\ApplicationDeploymentJob.php#L715-L752)

自定义构建命令下的 CLI 层 env 来源：

| 来源 | 是否生效 | 说明 |
|-----|---------|------|
| Shell 环境变量 | ✅ | 始终生效 |
| coolify_variables 前缀 | ❌ | **不添加** |
| 用户自带 `--env-file` | 看情况 | 用户写了就有，没写就没有 |
| 自动注入 `--env-file` | 看情况 | 用户没写才会注入 `/artifacts/build-time.env` |
| 项目目录 `.env` | ❌ | 因为总是有 `--env-file`（要么用户的，要么自动注入的） |

**关键差异**：自定义构建命令**没有** `coolify_variables` Shell 前缀。也就是说，YAML 里如果写了 `${COOLIFY_BRANCH}` 或 `${COOLIFY_FQDN}`，在自定义构建命令场景下，不会从 Shell 前缀拿到值。

### 4.2 自定义启动命令

路径代码：
- Raw Compose + 自定义启动：[ApplicationDeploymentJob.php#L805-L825](file:///d:/fz/0601-2\solo-dogfeeding\code\20-coolify\app\Jobs\ApplicationDeploymentJob.php#L805-L825)
- 非 Raw + 自定义启动：[ApplicationDeploymentJob.php#L839-L858](file:///d:/fz/0601-2\solo-dogfeeding\code\20-coolify\app\Jobs\ApplicationDeploymentJob.php#L839-L858)

与自定义构建命令类似，自定义启动命令下：

| 来源 | 是否生效 | 说明 |
|-----|---------|------|
| Shell 环境变量 | ✅ | 始终生效 |
| coolify_variables 前缀 | ❌ | **不添加** |
| 用户自带 `--env-file` | 看情况 | 用户写了就有 |
| 自动注入 `--env-file` | 看情况 | 用户没写才会注入 `<workdir>/.env` |
| 项目目录 `.env` | ❌ | 因为总是有 `--env-file` |

### 4.3 为什么自定义命令不加 coolify_variables？

这是设计选择：自定义命令意味着用户想完全掌控执行流程，Coolify 只做最少的自动注入（补全 `-f` 和 `--env-file`），不主动添加 Shell 前缀变量。

如果用户在自定义命令场景下需要 `COOLIFY_BRANCH` 等变量，可以：
1. 自己在命令里写 `--env-file` 指向包含这些变量的文件
2. 或者自己在命令前拼接变量

---

## 五、Raw Compose 模式的影响

Raw Compose 模式**不影响 CLI 层**的 env 来源优先级。它影响的是 compose 文件解析层面（parser 跳过时的行为）。

换句话说：
- ✅ Shell 前缀（非自定义命令路径下仍然有）
- ✅ `--env-file`（仍然自动注入）
- ✅ `.env` 文件生成（仍然会生成）
- ❌ parser 解析 YAML 中的变量引用、自动追加 `env_file` 字段等

Raw Compose 模式下，YAML 里的 `${VAR}` 仍然会被 Docker Compose CLI 层解析——因为那是 Docker Compose 的原生行为，不是 Coolify parser 做的。

---

## 六、完整优先级链总结

### 默认命令（非自定义）场景

```
优先级从高到低：

  1. Shell 环境变量（执行环境自带）
        ↑
  2. coolify_variables Shell 前缀（COOLIFY_* 等）
        ↑
  3. 自动注入的 --env-file
        （构建：/artifacts/build-time.env
         启动：<workdir>/.env）
        ↑
  4. （项目目录 .env 自动加载 — 通常不生效，因为有 --env-file）
```

### 自定义命令场景

```
优先级从高到低：

  1. Shell 环境变量（执行环境自带）
        ↑
  2. （coolify_variables 前缀 — 不生效！）
        ↑
  3. 用户自带的 --env-file（如果写了）
        或 自动注入的 --env-file（如果没写）
        ↑
  4. （项目目录 .env 自动加载 — 通常不生效）
```

---

## 七、关键代码位置索引

| 功能 | 文件 | 行号 |
|-----|------|------|
| coolify_variables 设置 | [ApplicationDeploymentJob.php](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/app/Jobs/ApplicationDeploymentJob.php) | L2227-L2256 |
| 构建命令组装（非自定义） | [ApplicationDeploymentJob.php](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/app/Jobs/ApplicationDeploymentJob.php) | L753-L776 |
| 自定义构建命令处理 | [ApplicationDeploymentJob.php](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/app/Jobs/ApplicationDeploymentJob.php) | L715-L752 |
| 启动命令组装（Raw 非自定义） | [ApplicationDeploymentJob.php](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/app/Jobs/ApplicationDeploymentJob.php) | L827-L837 |
| 启动命令组装（非 Raw 非自定义） | [ApplicationDeploymentJob.php](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/app/Jobs/ApplicationDeploymentJob.php) | L859-L879 |
| 自定义启动命令处理（Raw） | [ApplicationDeploymentJob.php](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/app/Jobs/ApplicationDeploymentJob.php) | L805-L825 |
| 自定义启动命令处理（非 Raw） | [ApplicationDeploymentJob.php](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/app/Jobs/ApplicationDeploymentJob.php) | L839-L858 |
| injectDockerComposeFlags 函数 | [docker.php](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/bootstrap/helpers/docker.php) | L1414-L1433 |
| injectDockerComposeBuildArgs 函数 | [docker.php](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/bootstrap/helpers/docker.php) | L1465-L1492 |
| build-time env 生成 | [ApplicationDeploymentJob.php](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/app/Jobs/ApplicationDeploymentJob.php) | L1593-L1848 |
| runtime env 生成 | [ApplicationDeploymentJob.php](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/app/Jobs/ApplicationDeploymentJob.php) | L1309-L1591 |
