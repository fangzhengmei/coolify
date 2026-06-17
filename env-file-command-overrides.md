# Compose 插值来源纠正：命令前缀覆盖与自定义 env-file 的变量丢失

本文纠正前一份文档（`env-file-precedence.md`）中对 CLI 层插值来源的两个理解偏差：

1. **命令前缀不是"并列"于 shell 环境的另一层**——它是对 shell 环境的临时覆盖，与 shell 环境同属一个"进程环境"层
2. **自定义命令带 `--env-file` 时，丢失的不仅是"优先级"，而是整个 Coolify 自动 env 文件不参与解析**——所有应用变量从 CLI 层插值中消失

---

## 一、命令前缀怎样覆盖和继承 shell 环境

### 1.1 Shell 临时赋值语义

Coolify 在默认命令路径下生成的命令形如：

```bash
COOLIFY_URL=https://app.example.com COOLIFY_FQDN=app.example.com COOLIFY_BRANCH=main COOLIFY_RESOURCE_UUID=abc123 docker compose --env-file /artifacts/build-time.env ...
```

这是 POSIX shell 的 **临时赋值前缀语法**（`VAR=value command`）。它的语义是：

- 为 `command`（这里是 `docker compose`）及其子进程创建一个**临时环境**
- 该临时环境**继承**父 shell 的所有变量（`PATH`、`HOME`、`USER` 等）
- 同时**覆盖**前缀中显式列出的变量（`COOLIFY_URL` 等）
- 赋值**不持久化**到父 shell——命令结束后这些变量就消失了

### 1.2 前缀与 shell 环境的关系

关键纠正：前缀和 shell 环境不是两个独立的"层"，而是**同一个进程环境**。

```
进程环境（docker compose 收到的环境）
├── 继承自父 shell：PATH, HOME, USER, ...
└── 前缀覆盖/新增：COOLIFY_URL, COOLIFY_FQDN, COOLIFY_BRANCH, COOLIFY_RESOURCE_UUID, SOURCE_COMMIT
```

前缀中出现的变量，优先级**高于**从父 shell 继承的同名变量。如果父 shell 已有 `COOLIFY_BRANCH=develop`，前缀里的 `COOLIFY_BRANCH=main` 会覆盖它。

### 1.3 前缀包含的变量（精确清单）

代码位置：[ApplicationDeploymentJob.php:set_coolify_variables()](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/app/Jobs/ApplicationDeploymentJob.php#L2227-L2256)

| 变量 | 来源 | 是否受用户自定义影响 |
|-----|------|-------------------|
| `SOURCE_COMMIT` | `$this->commit` | 仅当 `include_source_commit_in_build` 开启时才出现 |
| `COOLIFY_URL` | 从 `application->fqdn` 解析 | ❌ 不检查用户是否自定义，始终覆盖 |
| `COOLIFY_FQDN` | 从 `application->fqdn` 解析 | ❌ 不检查用户是否自定义，始终覆盖 |
| `COOLIFY_BRANCH` | `$this->application->git_branch` | ❌ **始终用真实 git 分支**，即使用户自定义了 COOLIFY_BRANCH |
| `COOLIFY_RESOURCE_UUID` | `$this->application->uuid` | ❌ 不检查用户是否自定义，始终覆盖 |

**关键差异**：前缀里的 `COOLIFY_BRANCH` 永远是真实 git 分支名，不看用户有没有自定义这个变量。这与 env 文件中的行为不同——env 文件中如果用户自定义了 `COOLIFY_BRANCH`，则不写入自动值。

### 1.4 前缀不包含的变量

前缀**只有**上表中的 5 个变量。以下变量不在前缀中，只能从 env 文件获得：

- `COOLIFY_CONTAINER_NAME` — 每次部署变化，破坏 Docker 缓存，不放前缀
- `SERVICE_NAME_*`、`SERVICE_URL_*`、`SERVICE_FQDN_*` — Docker Compose 服务变量
- 所有用户自定义环境变量
- 所有共享环境变量（`{{team.*}}`、`{{project.*}}` 等）

---

## 二、Coolify 自动 env 文件里有什么

要理解自定义命令带 `--env-file` 后丢失了什么，必须先搞清 Coolify 自动生成的 env 文件里有哪些变量。

### 2.1 build-time.env（`/artifacts/build-time.env`）

代码位置：[generate_buildtime_environment_variables()](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/app/Jobs/ApplicationDeploymentJob.php#L1593-L1810)

写入顺序（同一文件内后写覆盖先写）：

```
1. Nixpacks plan 变量（最低优先级）
2. COOLIFY_* 变量（COOLIFY_URL, COOLIFY_FQDN, COOLIFY_BRANCH, COOLIFY_RESOURCE_UUID, SOURCE_COMMIT）
3. SERVICE_NAME_*, SERVICE_URL_*, SERVICE_FQDN_*（dockercompose 构建）
4. 用户标记为 is_buildtime=true 的环境变量（最高优先级）
```

### 2.2 runtime .env（`<workdir>/.env`）

代码位置：[generate_runtime_environment_variables()](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/app/Jobs/ApplicationDeploymentJob.php#L1309-L1473)

写入顺序：

```
1. COOLIFY_* 变量（含 COOLIFY_CONTAINER_NAME，不含 SOURCE_COMMIT 除非开启）
2. SERVICE_NAME_*, SERVICE_URL_*, SERVICE_FQDN_*（dockercompose 运行）
3. 用户 runtime 环境变量（引用 SERVICE_* 的排后面）
4. PORT（非 dockercompose 且用户未定义时自动添加）
5. HOST=0.0.0.0（用户未定义时自动添加）
```

### 2.3 env 文件与前缀的变量重叠

| 变量 | Shell 前缀 | env 文件 | 冲突时谁赢 |
|-----|-----------|---------|-----------|
| COOLIFY_URL | ✅ | ✅ | 前缀赢（进程环境优先于 --env-file） |
| COOLIFY_FQDN | ✅ | ✅ | 前缀赢 |
| COOLIFY_BRANCH | ✅ 真实分支 | ✅ 用户可覆盖 | 前缀赢（始终真实分支） |
| COOLIFY_RESOURCE_UUID | ✅ | ✅ | 前缀赢 |
| SOURCE_COMMIT | ✅（可选） | ✅（可选） | 前缀赢 |
| COOLIFY_CONTAINER_NAME | ❌ | ✅ 仅 runtime | env 文件 |
| SERVICE_* | ❌ | ✅ | env 文件 |
| 用户变量 | ❌ | ✅ | env 文件 |

---

## 三、自定义命令带 env-file 后，哪些变量不再参与解析

这是最关键的纠正点。前一份文档把这当作"优先级"问题，但实际上是**变量是否参与**的问题。

### 3.1 触发条件

当用户在自定义构建/启动命令中**自己写了 `--env-file`**：

```bash
# 用户的自定义命令
docker compose --env-file /my/custom.env build
```

`injectDockerComposeFlags` 检测到 `--env-file` 已存在，**不会注入** Coolify 的 `--env-file /artifacts/build-time.env`（或 `<workdir>/.env`）。

代码位置：[docker.php:injectDockerComposeFlags()](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/bootstrap/helpers/docker.php#L1425-L1429)

```php
if (! preg_match('/(?:^|\s)--env-file(?:=|\s)/', $command)) {
    $dockerComposeReplacement .= " --env-file {$envFilePath}";
}
```

### 3.2 Docker Compose 的 --env-file 排他性

Docker Compose 的 `--env-file` 是**排他**的：

- 一旦指定了 `--env-file`，**不会自动加载**项目目录的 `.env`
- 可以指定多个 `--env-file`，按顺序加载，后者覆盖前者
- 但 Coolify 的 `injectDockerComposeFlags` 只在用户没写时注入一个

### 3.3 自定义命令路径下，Shell 前缀也不存在

代码位置对比：

- **默认构建命令**：[ApplicationDeploymentJob.php#L754](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/app/Jobs/ApplicationDeploymentJob.php#L754)
  ```php
  $command = "{$this->coolify_variables} docker compose";  // ← 有前缀
  ```

- **自定义构建命令**：[ApplicationDeploymentJob.php#L717](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/app/Jobs/ApplicationDeploymentJob.php#L715-L752)
  ```php
  $build_command = injectDockerComposeFlags(...);  // ← 无前缀
  ```

- **默认启动命令**：[ApplicationDeploymentJob.php#L830](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/app/Jobs/ApplicationDeploymentJob.php#L830) 和 [L860](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/app/Jobs/ApplicationDeploymentJob.php#L860)
  ```php
  $command = "{$this->coolify_variables} docker compose";  // ← 有前缀
  ```

- **自定义启动命令**：[ApplicationDeploymentJob.php#L807](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/app/Jobs/ApplicationDeploymentJob.php#L805-L825) 和 [L843](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/app/Jobs/ApplicationDeploymentJob.php#L839-L858)
  ```php
  $start_command = injectDockerComposeFlags(...);  // ← 无前缀
  ```

### 3.4 变量丢失矩阵

**场景：自定义命令 + 用户自带 `--env-file`**

```
CLI 层插值来源：
  1. 进程环境（继承 shell）     ← 只有 PATH/HOME 等，无 COOLIFY_*
  2. 用户的 --env-file           ← 只有用户文件里的变量
  3. Coolify 自动 env 文件       ← ❌ 完全不加载
  4. 项目目录 .env 自动加载      ← ❌ 不触发（因为已有 --env-file）
```

以下变量**从 CLI 层 YAML 插值中消失**：

| 变量类别 | 是否参与 CLI 层插值 | 原因 |
|---------|-------------------|------|
| COOLIFY_URL | ❌ | 前缀不存在 + env 文件不加载 |
| COOLIFY_FQDN | ❌ | 前缀不存在 + env 文件不加载 |
| COOLIFY_BRANCH | ❌ | 前缀不存在 + env 文件不加载 |
| COOLIFY_RESOURCE_UUID | ❌ | 前缀不存在 + env 文件不加载 |
| COOLIFY_CONTAINER_NAME | ❌ | env 文件不加载 |
| SOURCE_COMMIT | ❌ | 前缀不存在 + env 文件不加载 |
| SERVICE_NAME_* | ❌ | env 文件不加载 |
| SERVICE_URL_* | ❌ | env 文件不加载 |
| SERVICE_FQDN_* | ❌ | env 文件不加载 |
| 用户应用环境变量 | ❌ | env 文件不加载 |

**结果**：YAML 中所有 `${COOLIFY_*}`、`${SERVICE_*}`、`${USER_VAR}` 引用都会变成空字符串（或 Docker Compose 的默认空值），除非用户在自己的 `--env-file` 中定义了同名变量。

### 3.5 对比：自定义命令不带 `--env-file`

```
CLI 层插值来源：
  1. 进程环境（继承 shell）     ← 只有 PATH/HOME 等，无 COOLIFY_*
  2. Coolify 自动注入 --env-file ← ✅ 加载 build-time.env 或 .env
  3. 项目目录 .env 自动加载      ← ❌ 不触发
```

此时 COOLIFY_*、SERVICE_*、用户变量都从 env 文件参与 CLI 层插值——但 `COOLIFY_BRANCH` 的值可能和默认命令路径下不同（因为 env 文件里如果用户自定义了 COOLIFY_BRANCH，写入的是用户值；而前缀里永远是真实分支）。

---

## 四、容器层的补偿机制

CLI 层变量丢失不等于容器里也丢失。容器层有独立的变量来源。

### 4.1 非 Raw Compose 模式

parser 自动给每个服务追加 `env_file: [.env]`：

代码位置：[parsers.php](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/bootstrap/helpers/parsers.php#L1420-L1428) 和 [ApplicationDeploymentJob.php#L679-L686](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/app/Jobs/ApplicationDeploymentJob.php#L679-L686)

```php
$service['env_file'] = ['.env'];
```

所以即使自定义命令的 `--env-file` 导致 CLI 层不加载 `.env`，**容器层**仍然会通过 `services.*.env_file` 加载 `.env` 文件（因为它已生成在磁盘上）。

但有个前提：这个 `.env` 文件确实已写入磁盘。在启动阶段，`save_runtime_environment_variables()` 在构建之后执行，写入 `<workdir>/.env`。

### 4.2 Raw Compose 模式

Raw Compose **跳过 parser**，不自动追加 `env_file: [.env]`。

代码位置：[ApplicationDeploymentJob.php#L668-L676](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/app/Jobs/ApplicationDeploymentJob.php#L668-L676)

所以 Raw Compose + 自定义命令 + 用户 `--env-file`：

| 层面 | 变量来源 | 结果 |
|-----|---------|------|
| CLI 层（YAML 插值） | 用户 `--env-file` only | COOLIFY_*/SERVICE_*/用户变量全部缺失 |
| 容器层 | 用户 compose 文件原样 | 只有用户在 compose 里显式写的 environment/env_file |

这是变量丢失最严重的场景。

---

## 五、四种场景的完整对比

| 场景 | Shell 前缀 | Coolify --env-file | CLI 层有 COOLIFY_*? | CLI 层有 SERVICE_*/用户变量? | 容器层有 .env? |
|-----|-----------|-------------------|--------------------|-------------------------|---------------|
| 默认命令 | ✅ | ✅ 自动注入 | ✅（前缀，真实分支） | ✅（env 文件） | ✅（parser 追加 env_file） |
| 自定义命令，无自带 --env-file | ❌ | ✅ 自动注入 | ✅（env 文件，可能用户值） | ✅（env 文件） | ✅（parser 追加 env_file） |
| 自定义命令 + 自带 --env-file | ❌ | ❌ 不注入 | ❌ 全部缺失 | ❌ 全部缺失 | ✅（parser 追加 env_file，非 Raw） |
| Raw Compose + 自定义命令 + 自带 --env-file | ❌ | ❌ 不注入 | ❌ 全部缺失 | ❌ 全部缺失 | ❌（parser 被跳过） |

---

## 六、关键纠正总结

### 纠正 1：前缀不是独立层，是对 shell 环境的覆盖

前一份文档把 "Shell 环境变量" 和 "coolify_variables 前缀" 列为两个独立的优先级层。实际上它们是**同一个进程环境**——前缀只是在该环境中覆盖了特定变量。

正确理解：
- 进程环境 = 继承的 shell 变量 + 前缀覆盖的变量
- 前缀变量**覆盖**同名继承变量
- 前缀变量优先级**高于** `--env-file`（因为进程环境优先于文件加载）

### 纠正 2：丢失的不是"优先级"，是"参与与否"

前一份文档把自定义命令带 `--env-file` 描述为"用户自带 `--env-file` 排在 Shell 前缀之后"。

实际上：
- 自定义命令路径下**根本没有 Shell 前缀**
- 用户自带 `--env-file` 导致 Coolify 的 env 文件**完全不加载**
- 这不是"优先级降低"，而是"变量从 CLI 层插值中完全消失"

### 纠正 3：COOLIFY_BRANCH 的双值行为

前一份文档没有区分 `COOLIFY_BRANCH` 在前缀和 env 文件中的不同取值逻辑：

- **前缀**：始终是 `$this->application->git_branch`（真实分支），不看用户自定义
- **env 文件**：如果用户自定义了 `COOLIFY_BRANCH`，则不写入自动值（用户值生效）

代码位置：
- 前缀：[set_coolify_variables() L2252-L2254](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/app/Jobs/ApplicationDeploymentJob.php#L2252-L2254) — 无条件写入
- env 文件：[generate_coolify_env_variables() L3017-L3020](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/app/Jobs/ApplicationDeploymentJob.php#L3017-L3020) — 检查 `isEmpty()` 后才写入

---

## 七、代码位置索引

| 功能 | 文件 | 行号 |
|-----|------|------|
| Shell 前缀设置 | [ApplicationDeploymentJob.php](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/app/Jobs/ApplicationDeploymentJob.php) | L2227-L2256 |
| COOLIFY_* env 文件生成 | [ApplicationDeploymentJob.php](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/app/Jobs/ApplicationDeploymentJob.php) | L2942-L3037 |
| build-time env 生成 | [ApplicationDeploymentJob.php](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/app/Jobs/ApplicationDeploymentJob.php) | L1593-L1810 |
| runtime env 生成 | [ApplicationDeploymentJob.php](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/app/Jobs/ApplicationDeploymentJob.php) | L1309-L1473 |
| 默认构建命令（有前缀） | [ApplicationDeploymentJob.php](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/app/Jobs/ApplicationDeploymentJob.php) | L753-L776 |
| 自定义构建命令（无前缀） | [ApplicationDeploymentJob.php](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/app/Jobs/ApplicationDeploymentJob.php) | L715-L752 |
| 默认启动命令（有前缀） | [ApplicationDeploymentJob.php](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/app/Jobs/ApplicationDeploymentJob.php) | L827-L879 |
| 自定义启动命令（无前缀） | [ApplicationDeploymentJob.php](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/app/Jobs/ApplicationDeploymentJob.php) | L805-L858 |
| injectDockerComposeFlags | [docker.php](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/bootstrap/helpers/docker.php) | L1414-L1433 |
| parser 自动追加 env_file | [parsers.php](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/bootstrap/helpers/parsers.php) | L1420-L1428 |
| 非 parser 路径追加 env_file | [ApplicationDeploymentJob.php](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/app/Jobs/ApplicationDeploymentJob.php) | L679-L686 |
| Raw Compose 跳过 parser | [ApplicationDeploymentJob.php](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/app/Jobs/ApplicationDeploymentJob.php) | L668-L676 |
