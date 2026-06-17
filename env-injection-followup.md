# Coolify 环境变量优先级与共享变量解析 — 修正分析

本文档聚焦在前一份分析中容易混淆的两个核心问题：
1. **Compose `environment:` 内联值 vs `.env` (env_file) vs 系统环境变量** 的真实生效时机和优先级
2. **`{{team.var}}` / `{{project.var}}` / `{{environment.var}}` / `{{server.var}}`** 四类共享变量分别在哪个代码路径被解析、为什么要在那个时机解析

---

## 一、Docker Compose 原生优先级（必须先理解）

在分析 Coolify 的代码之前，先明确 Docker Compose 自己的优先级规则，这是理解 Coolify 所有设计决策的基础。

### 1.1 Compose 对单个变量的解析优先级（从高到低）

```
1. docker-compose.yml 中 environment: 字段内联写入的「具体值」
       ↓
2. docker compose up 时主机（Shell）上已有的同名环境变量
       ↓
3. env_file: 指定文件（如 .env）中的值
       ↓
4. Dockerfile 中 ENV 指令定义的值
       ↓
5. 容器内 OS 默认值（通常为空）
```

> **关键认知**：`environment:` 字段里如果写的是 `KEY=${VAR}` 这种 **引用形式**，Docker Compose 会按优先级 2→3→4→5 去查找 `VAR` 的值再替换；但如果写的是 `KEY=hardcoded-value` 这种 **硬编码值**，则优先级 1 直接生效，`.env` 里同名变量完全不会起作用。

### 1.2 一个具体例子说明优先级 1 与 3 的冲突

```yaml
# docker-compose.yml
services:
  app:
    environment:
      DB_PASSWORD: mysecret-123    # 内联硬编码值
    env_file:
      - .env                      # 里写了 DB_PASSWORD=other-value
```

最终容器内 `DB_PASSWORD = mysecret-123`，**`.env` 被完全忽略**。

Coolify 的代码在两个地方都要对抗这个规则，后面会详细说明。

---

## 二、Coolify 双轨注入机制的真实运作

Coolify 对每个服务同时使用两条路径注入环境变量：

```
          路径 A: environment: 内联（写在 compose YAML 里）
                        │
                        ▼
┌──────────────────────────────────────────┐
│            Docker Compose                │
│                                          │
│  services.app:                           │
│    environment:                          │
│      COOLIFY_FQDN: "app.example.com" ←──┼── 路径 A 内联具体值
│      DATABASE_URL: ${DATABASE_URL}   ←──┼── 路径 A 写的是引用（不是具体值）
│    env_file:                             │
│      - .env            ←────────────────┼── 路径 B 由 Coolify 生成的 .env
└──────────────────────────────────────────┘
                        │
                        ▼
          路径 B: .env 文件（写入服务器工作目录）
```

### 2.1 路径 A：写入 `environment:` 字段的代码

代码位置：[parsers.php L1411-L1413](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/bootstrap/helpers/parsers.php#L1411-L1413)

```php
if ($environment->count() > 0 || $coolifyEnvironments->count() > 0) {
    $payload['environment'] = $environment
        ->merge($coolifyEnvironments)
        ->merge($serviceNameEnvironments);
}
```

三个集合按此顺序合并（后者覆盖前者，但实际实现中 key 不重叠）：

| 集合 | 内容 | 来源 |
|-----|------|------|
| `$environment` | Compose 模板原始 `environment:` + Coolify 解析后补充的变量 | `docker-compose.yml` 模板 |
| `$coolifyEnvironments` | `COOLIFY_FQDN`、`COOLIFY_URL`、`COOLIFY_BRANCH`、`COOLIFY_RESOURCE_UUID`、`COOLIFY_CONTAINER_NAME` | 运行时自动生成 |
| `$serviceNameEnvironments` | `SERVICE_NAME_*`（服务容器名） | 运行时自动生成 |

### 2.2 路径 B：写入 `.env` 文件的代码

代码位置：[ApplicationDeploymentJob.php L1484-L1591](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/app/Jobs/ApplicationDeploymentJob.php#L1484-L1591)

最终 `.env` 的条目顺序（**后出现的条目在 Docker Compose 解析时覆盖前面的同名 key**，因为 `.env` 是逐行读取、后者胜出）：

```
① COOLIFY_* 自动生成变量                    (最低)
② SERVICE_* 魔术变量 (SERVICE_URL_*, SERVICE_FQDN_*, SERVICE_NAME_*)
③ 不引用 ${SERVICE_*} 的用户自定义变量
④ 引用 ${SERVICE_*} 的用户自定义变量          (最高，因为排到后面)
⑤ (预览部署) 预览变量 → 生产变量回退
⑥ PORT / HOST 兜底（仅当用户未定义时）
```

> 注意：`.env` 是 shell 风格的 `KEY=VALUE` 格式，**同一 key 多次出现时以最后一行为准**。这就是 Coolify 把引用 `SERVICE_*` 的变量排到后面的根本原因——让它们在 `.env` 内就能读到第②步已写入的 `SERVICE_*` 值。

---

## 三、`environment:` 与 `.env` 的微妙分工

### 3.1 自引用变量：故意保留 `${VAR}` 而不立即替换

代码位置：[parsers.php L991-L1003](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/bootstrap/helpers/parsers.php#L991-L1003)

```php
if ($key->value() === $parsedValue->value()) {
    // Simple variable reference (e.g. DATABASE_URL: ${DATABASE_URL})
    // Ensure the variable exists in DB for .env generation and UI display
    $resource->environment_variables()->firstOrCreate([
        'key' => $key,
        ...
    ], [...]);
    // Keep the ${VAR} reference in compose — Docker Compose resolves from .env at deploy time.
    // Do NOT replace with DB value: if user updates env var without re-parsing compose,
    // a stale resolved value in environment: would override the correct .env value.
}
```

**设计意图（注释写得非常清楚）：**

当 Compose 模板里出现 `DATABASE_URL: ${DATABASE_URL}`（变量 key 等于引用名）这种写法时：

1. ✅ 在数据库里 `firstOrCreate` 一条记录（确保 UI 能看到、`.env` 生成时能写出）
2. ❌ **绝对不能**把数据库里的真实值写回到 `environment:` 里
3. ✅ 在最终的 compose YAML 中保留 `DATABASE_URL: ${DATABASE_URL}` 这行引用

**为什么不能直接写值？** 因为一旦把真实值写进 `environment:`，它就落入了上一节所说的优先级 1（内联具体值）。之后用户在 UI 里改了这个变量值并重新部署，Coolify 会更新 `.env`，但 **不会重新解析 Compose**，结果 `environment:` 里写的还是旧值，旧值（优先级 1）反而覆盖了 `.env` 里的新值（优先级 3），导致用户的修改不生效。

> 这是 GitHub issue #8885 和 #9136 的根因。回归测试见 [ServiceParserEnvVarPreservationTest.php](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/tests/Unit/ServiceParserEnvVarPreservationTest.php)。

### 3.2 带默认值的变量：数据库里落默认值、Compose 里替换成引用

代码位置：[parsers.php L1005-L1093](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/bootstrap/helpers/parsers.php#L1005-L1093)

```yaml
# 模板写法
DATABASE_URL: ${DATABASE_URL:-postgres://postgres:postgres@postgres:5432/postgres}
```

解析时发生的事：
1. 提取变量名 `DATABASE_URL`、默认值 `postgres://postgres:...`
2. `firstOrCreate` 写入数据库（value = 默认值）
3. 对默认值中嵌套的 `${SERVICE_URL_POSTGRES}` 等 **递归** 同样处理
4. `$environment[$varName] = $envVar->value;` ← **关键**：把数据库值写回 `$environment` 集合

这里没有像自引用那样「保留引用」，是因为模板写法是 `KEY=${VAR:-default}`，`KEY` 不一定等于 `VAR`，替换后不存在「用户改 UI 但 Compose 不重解析」的一致性问题。

### 3.3 null vs 空字符串的区别

代码位置：[parsers.php L1284-L1299](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/bootstrap/helpers/parsers.php#L1284-L1299)

```php
if ($value === null) {
    // User explicitly wants variable unset - respect that
    // NEVER override from database - null means "inherit from environment"
} elseif ($value === '') {
    // Empty string - allow database override for backward compatibility
    $dbEnv = $resource->environment_variables()->where('key', $key)->first();
    if ($dbEnv && str($dbEnv->value)->isNotEmpty()) {
        $value = $dbEnv->value;
    }
}
```

| Compose 中的写法 | 含义 | Coolify 行为 |
|----------------|------|-------------|
| `KEY:` (YAML null) | 用户显式要求「不要设这个变量，让它从主机继承」 | 保留 null，**绝不**从数据库取 |
| `KEY: ""` (空字符串) | 用户设了一个空值 | 先看数据库有没有非空值，有就覆盖；没有就保留空串 |

---

## 四、共享变量 `{{team.var}}` 等的解析时机与原因

### 4.1 四类共享变量的定义模型

模型：[SharedEnvironmentVariable.php](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/app/Models/SharedEnvironmentVariable.php)

| 引用语法 | type 字段 | 查找 id 的来源 |
|---------|----------|--------------|
| `{{team.VAR}}` | `team` | `$resource->team()->id` |
| `{{project.VAR}}` | `project` | `$resource->environment->project->id` |
| `{{environment.VAR}}` | `environment` | `$resource->environment->id` |
| `{{server.VAR}}` | `server` | `$serverOverride` → `$resource->server` → `$resource->destination->server` |

数据库查询：
```php
SharedEnvironmentVariable::where('type', $type)
    ->where('key', $variable)
    ->where('team_id', $resource->team()->id)
    ->where("{$type}_id", $id)
    ->first();
```

### 4.2 共享变量被解析的 **两处** 时机

#### 时机 1：解析 Compose 时（parsers.php）

代码位置：[parsers.php L1301-L1306](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/bootstrap/helpers/parsers.php#L1301-L1306)

```php
// Resolve shared variable patterns like {{environment.VAR}}, {{project.VAR}}, {{team.VAR}}
// Without this, literal {{...}} strings end up in the compose environment: section,
// which takes precedence over the resolved values in the .env file (env_file:)
if (is_string($value) && str_contains($value, '{{')) {
    $value = resolveSharedEnvironmentVariables($value, $resource);
}
```

**为什么必须在这时解析？**

回想第一节的优先级：`environment:` 内联值（优先级 1）> `env_file`（优先级 3）。

如果 Coolify **不**在这里解析，那么最终 compose YAML 里会是：

```yaml
environment:
  API_TOKEN: "{{team.MASTER_TOKEN}}"    # 字面字符串，优先级 1
env_file:
  - .env                                # 里写 API_TOKEN=实际值，优先级 3
```

容器内拿到的是字面字符串 `{{team.MASTER_TOKEN}}`，而不是解析后的真实值——因为优先级 1 赢了。

**所以必须在 compose 生成阶段就把 `{{...}}` 替换成真实值**，写入 `environment:` 字段。

#### 时机 2：生成 `.env` 文件时（EnvironmentVariable 模型）

代码位置：[EnvironmentVariable.php L253-L293](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/app/Models/EnvironmentVariable.php#L253-L293)

```php
public function getResolvedValueWithServer($server = null)
{
    // ... 加载 resource、environment、destination.server 等关系 ...
    $real_value = $this->get_real_environment_variables_internal($this->value, $resource, $server);

    if ($this->is_literal || $this->is_multiline) {
        $real_value = '\''.$real_value.'\'';
    } else {
        $real_value = escapeEnvVariables($real_value);
    }

    return $real_value;
}
```

以及 `realValue` Attribute ([EnvironmentVariable.php L171-L210](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/app/Models/EnvironmentVariable.php#L171-L210))。

**为什么这里也要解析一次？**

因为用户在 UI 里直接创建的环境变量，其 `value` 字段存的就是原始的 `{{team.VAR}}` 字符串（数据库里存的是字面量）。这些变量不会经过 `parsers.php`（它们不是 Compose 模板里的），只能在最终写出 `.env` 之前由 `getResolvedValueWithServer()` 解析。

调用链：
```
save_runtime_environment_variables()
  → generate_runtime_environment_variables()
    → $env->getResolvedValueWithServer($this->mainServer)   ← 解析 {{...}}
      → get_real_environment_variables_internal()
        → resolveSharedEnvironmentVariables()
```

### 4.3 两处解析的覆盖场景对比

| 场景 | 经 parsers.php 解析？ | 经 getResolvedValueWithServer 解析？ |
|-----|----------------------|-------------------------------------|
| Compose 模板 `environment:` 中含 `{{team.VAR}}` | ✅ 是 | ❌ 否（这个值写在 `environment:` 里，不走 `.env` 路径） |
| 用户在 UI 直接创建的变量值为 `{{team.VAR}}` | ❌ 否 | ✅ 是（写 `.env` 时解析） |
| Compose 模板中值是 `${VAR}`，数据库里 VAR 的值是 `{{team.X}}` | ❌ 否（Compose 里保留了 `${VAR}`） | ✅ 是（`.env` 里 VAR=xxx 这行会被解析） |
| `is_shadow` / 系统自动生成的变量 | ❌ 否 | ✅ 是（只要最终进 `.env`） |

---

## 五、完整的最终优先级链（修正版）

综合 Docker Compose 原生规则 + Coolify 的两层注入，最终容器内一个变量的值按以下链决定：

```
              ┌─────────────────────────────────────────────────────────┐
              │          最终容器内实际拿到的环境变量值                    │
              └─────────────────────────────────────────────────────────┘
                                    │
                    ┌───────────────┴───────────────┐
                    │                               │
          ┌─────────▼─────────┐          ┌──────────▼──────────┐
          │  路径 A 是否在      │          │  路径 B：.env 文件    │
          │  environment: 里    │   NO     │  中是否存在该 key     │
          │  写了「具体值」？    │─────────▶│                      │
          └─────────┬─────────┘          └──────────┬──────────┘
                    │ YES                            │
                    ▼                                ▼
     ┌──────────────────────────────┐    ┌─────────────────────────────┐
     │ 使用 environment: 中的值      │    │ 取 .env 中该 key 的最后一行  │
     │  (Docker Compose 优先级 1)    │    │  (Docker Compose 优先级 3)  │
     └──────────────────────────────┘    └─────────────────────────────┘
```

在 `.env` 内部，条目出现顺序决定优先级（后者覆盖前者），Coolify 精心安排了这个顺序：

```
.env 中条目从上到下（后出现的覆盖先出现的）：
┌────────────────────────────────────────────────────────────┐
│ COOLIFY_FQDN=...                                            │
│ COOLIFY_URL=...                                             │  ← 先写：容易被覆盖
│ COOLIFY_BRANCH=...                                          │
│ COOLIFY_RESOURCE_UUID=...                                   │
│ COOLIFY_CONTAINER_NAME=...                                  │
│ ────────────────────────────────────────────────            │
│ SERVICE_URL_POSTGRES=postgres://...                         │
│ SERVICE_FQDN_POSTGRES=db.example.com                        │
│ SERVICE_NAME_POSTGRES=postgres-1                            │
│ ────────────────────────────────────────────────            │
│ API_KEY=user-defined-value                                  │
│ DEBUG=false                                                 │  ← 不引用 SERVICE_* 的用户变量
│ ────────────────────────────────────────────────            │
│ DATABASE_URL=${SERVICE_URL_POSTGRES}/mydb                   │  ← 引用 SERVICE_* 的变量
│ REDIS_URL=${SERVICE_URL_REDIS}/0                            │     故意排到最后
│ ────────────────────────────────────────────────            │
│ PORT=3000                                                   │  ← 兜底（仅当用户未定义）
│ HOST=0.0.0.0                                                │
└────────────────────────────────────────────────────────────┘
```

### 5.1 用户变量 vs 系统自动生成变量的覆盖关系

| 变量来源 | 写入 `.env` 的位置 | 能否被其他来源覆盖 |
|---------|-------------------|------------------|
| Nixpacks 自动检测（构建时） | 最先写 | 被一切覆盖 |
| `COOLIFY_*` 自动生成 | 较早 | 被用户同 key 覆盖 |
| `SERVICE_*` 魔术变量 | 中等 | 被用户同 key 覆盖（UI 中设置了更高优先级的排序，但实际逻辑是用户变量在 SERVICE_* 之后写） |
| 用户自定义变量（不引用 SERVICE_*） | 较晚 | 只被引用 SERVICE_* 的用户变量覆盖 |
| 用户自定义变量（引用 SERVICE_*） | 最晚 | **不会被其他用户变量覆盖** |
| PORT / HOST 兜底 | 最后，但仅当 `where('key', 'PORT')->isEmpty()` 时才写 | 只有在完全没定义时才出现 |

> **特别注意**：`is_env_sorting_enabled` 开启时，用户变量之间按 `key` 字母序排序；关闭时按 `id`（创建顺序）排序。但无论是否开启，「引用 SERVICE_* 的用户变量」都会被一个单独的 `sortBy` 排到所有普通用户变量之后。

---

## 六、`{{server.var}}` 的特殊之处

四类共享变量中，`server` 类型有一个额外的查找链：

代码位置：[EnvironmentVariable.php L323-L330](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/app/Models/EnvironmentVariable.php#L323-L330)

```php
} elseif ($type->value() === 'server') {
    if ($serverOverride) {
        $id = $serverOverride->id;
    } elseif (isset($resource->server) && $resource->server) {
        $id = $resource->server->id;
    } elseif (isset($resource->destination) && $resource->destination
           && isset($resource->destination->server)) {
        $id = $resource->destination->server->id;
    }
}
```

三级回退：
1. **显式传入的 server**（`getResolvedValueWithServer($this->mainServer)`）
2. 资源自身的 `server` 关系（Service 类资源常用）
3. 资源的 `destination.server`（Application 类资源常用）

这是因为：
- Application 通过 `destination` 间接关联到 Server
- Service 直接有 `server_id` 字段
- 构建服务器和部署服务器可能不是同一台（`use_build_server` 场景），此时部署 Job 会明确传入 `$this->mainServer` 确保用对服务器的共享变量

---

## 七、修正总结（与前一份分析的关键差异）

| 主题 | 之前的理解 | 修正后的正确理解 |
|-----|-----------|----------------|
| `environment:` vs `.env` | 说「双轨注入」但没解释优先级冲突 | `environment:` 中写入具体值会 **完全屏蔽** `.env` 中的同名变量；因此 Coolify 对自引用变量故意在 `environment:` 中保留 `${VAR}` 而不写值 |
| 共享变量解析时机 | 说「两级解析」但没解释原因 | parsers.php 里解析是为了不让字面 `{{...}}` 落到优先级 1；模型里解析是为了 UI 创建的变量最终也能被替换 |
| `.env` 内优先级 | 说「用户定义 > 自动生成」 | 更精确：`.env` 内是 **后者覆盖前者**，Coolify 故意按 COOLIFY_* → SERVICE_* → 普通用户变量 → 引用 SERVICE_* 的用户变量 排序，以及引用变量靠 `sortBy` 额外下沉 |
| 预览回退 | 笼统说「预览覆盖生产」 | 仅当预览变量集合非空时才回退；且预览变量先写入 `.env`，生产回退值后写（所以预览值优先） |
| null vs 空串 | 未区分 | Compose 中 `KEY:` (null) 表示「从主机继承，绝不查 DB」；`KEY: ""` 表示「空值，但允许 DB 覆盖」 |

---

## 八、关键代码位置索引（聚焦本文件主题）

| 主题 | 文件 | 行号 |
|-----|------|------|
| `environment:` 合并写入 | [parsers.php](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/bootstrap/helpers/parsers.php) | L1411-L1413 |
| 自引用变量保留 `${VAR}` | [parsers.php](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/bootstrap/helpers/parsers.php) | L991-L1003 |
| null vs 空串处理 | [parsers.php](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/bootstrap/helpers/parsers.php) | L1284-L1299 |
| 共享变量在 compose 阶段解析 | [parsers.php](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/bootstrap/helpers/parsers.php) | L1301-L1306 |
| `env_file` 注入 | [parsers.php](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/bootstrap/helpers/parsers.php) | L1422-L1428 |
| 生成运行时 `.env` 条目顺序 | [ApplicationDeploymentJob.php](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/app/Jobs/ApplicationDeploymentJob.php) | L1309-L1473 |
| 写入服务器 `.env` 文件 | [ApplicationDeploymentJob.php](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/app/Jobs/ApplicationDeploymentJob.php) | L1484-L1591 |
| SERVICE_* 引用变量下沉排序 | [ApplicationDeploymentJob.php](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/app/Jobs/ApplicationDeploymentJob.php) | L1365-L1372 |
| 预览 → 生产回退 | [ApplicationDeploymentJob.php](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/app/Jobs/ApplicationDeploymentJob.php) | L1449-L1457 |
| 共享变量在模型中解析 | [EnvironmentVariable.php](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/app/Models/EnvironmentVariable.php) | L258-L293 |
| server 类型三级回退 | [EnvironmentVariable.php](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/app/Models/EnvironmentVariable.php) | L323-L330 |
| 回归测试（自引用变量不被覆盖） | [ServiceParserEnvVarPreservationTest.php](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/tests/Unit/ServiceParserEnvVarPreservationTest.php) | 全文 |
