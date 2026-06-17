# Coolify 环境变量处理流程深度分析

本文档深入分析 Coolify 中环境变量从定义到注入容器的完整生命周期，包括来源解析、占位符插值、密钥引用和容器注入四个阶段，以及各阶段的优先级与覆盖关系。

---

## 整体架构概览

```
┌─────────────────────────────────────────────────────────────────┐
│                    环境变量完整生命周期                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────┐    ┌──────────────┐    ┌──────────────────┐   │
│  │  来源解析    │───▶│  占位符插值  │───▶│   密钥/引用解析  │   │
│  └─────────────┘    └──────────────┘    └──────────────────┘   │
│                                 │                                │
│                                 ▼                                │
│                       ┌──────────────────┐                       │
│                       │   容器注入阶段   │                       │
│                       └──────────────────┘                       │
│                                 │                                │
│                                 ▼                                │
│                       ┌──────────────────┐                       │
│                       │   Docker 容器     │                       │
│                       └──────────────────┘                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 第一阶段：来源解析 (Source Parsing)

### 1.1 环境变量的六大来源

| 来源类型 | 优先级 | 描述 | 核心代码位置 |
|---------|-------|------|-------------|
| **用户自定义变量** | 最高 | 用户在 UI 中手动创建的变量，存储在数据库 | [EnvironmentVariable.php](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/app/Models/EnvironmentVariable.php) |
| **Coolify 自动生成变量** | 高 | 系统自动生成的 `COOLIFY_*` 和 `SERVICE_*` 变量 | [ApplicationDeploymentJob.php](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/app/Jobs/ApplicationDeploymentJob.php#L2942-L3037) |
| **Docker Compose 模板变量** | 中 | 从 `docker-compose.yml` 的 `environment` 字段解析 | [parsers.php](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/bootstrap/helpers/parsers.php#L361-L1502) |
| **共享环境变量** | 中 | 通过 `{{team.VAR}}` / `{{project.VAR}}` / `{{environment.VAR}}` 引用 | [shared.php](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/bootstrap/helpers/shared.php#L4138-L4176) |
| **Nixpacks 计划变量** | 低 | Nixpacks 构建器自动检测的变量 | [ApplicationDeploymentJob.php](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/app/Jobs/ApplicationDeploymentJob.php#L1604-L1630) |
| **模板默认值** | 最低 | `${VAR:-default}` 语法中的默认值 | [services.php](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/bootstrap/helpers/services.php#L71-L97) |

### 1.2 数据库存储层

核心模型：`EnvironmentVariable` ([EnvironmentVariable.php](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/app/Models/EnvironmentVariable.php))

**关键字段：**
- `key` - 变量名，自动规范化和验证
- `value` - 变量值，使用 Laravel `encrypted` cast 加密存储
- `is_buildtime` - 是否在构建阶段可用（默认 true）
- `is_runtime` - 是否在运行阶段可用（默认 true）
- `is_preview` - 是否为预览部署专用
- `is_literal` - 是否为字面量（禁止 `$` 扩展）
- `is_multiline` - 是否为多行值
- `is_shared` - 是否为共享变量引用
- `is_required` - 是否为必需变量
- `is_shown_once` - 是否只显示一次

**加密机制：**
```php
// 数据库层面自动加密/解密
protected $casts = [
    'value' => 'encrypted',  // Laravel 内置加密 cast
];
```

### 1.3 共享环境变量 (Shared Variables)

模型：`SharedEnvironmentVariable` ([SharedEnvironmentVariable.php](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/app/Models/SharedEnvironmentVariable.php))

支持四种作用域：
- `{{team.VAR}}` - 团队级别
- `{{project.VAR}}` - 项目级别
- `{{environment.VAR}}` - 环境级别
- `{{server.VAR}}` - 服务器级别

**解析逻辑：** [resolveSharedEnvironmentVariables()](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/bootstrap/helpers/shared.php#L4138-L4176)

```php
// 解析 {{type.variable}} 语法
$sharedEnvsFound = str($value)->matchAll('/{{(.*?)}}/');
// 根据 type 查找对应的记录
$found = SharedEnvironmentVariable::where('type', $type)
    ->where('key', $variable)
    ->where('team_id', $resource->team()->id)
    ->where("{$type}_id", $id)
    ->first();
```

### 1.4 魔术变量 (Magic Variables)

**SERVICE_* 系列变量：**
- `SERVICE_FQDN_<NAME>` - 服务域名（无协议）
- `SERVICE_URL_<NAME>` - 服务 URL（含协议）
- `SERVICE_NAME_<NAME>` - 服务容器名
- `SERVICE_USER_<NAME>` - 服务用户名
- `SERVICE_PASSWORD_<NAME>` - 服务密码
- `SERVICE_HOST_<NAME>` - 服务主机名
- `SERVICE_PORT_<NAME>` - 服务端口

**COOLIFY_* 系列变量：**
- `COOLIFY_FQDN` - 应用域名
- `COOLIFY_URL` - 应用 URL
- `COOLIFY_BRANCH` - Git 分支
- `COOLIFY_RESOURCE_UUID` - 资源 UUID
- `COOLIFY_CONTAINER_NAME` - 容器名
- `SOURCE_COMMIT` - Git 提交哈希

**自动生成逻辑：** [generate_coolify_env_variables()](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/app/Jobs/ApplicationDeploymentJob.php#L2942-L3037)

> **注意**：`COOLIFY_CONTAINER_NAME` 和 `SOURCE_COMMIT` 默认不包含在构建阶段，避免破坏 Docker 缓存。

---

## 第二阶段：占位符插值 (Placeholder Interpolation)

### 2.1 支持的语法格式

Coolify 支持四种 Docker Compose 风格的变量语法：

| 语法 | 含义 | 行为 |
|-----|------|------|
| `${VAR}` | 简单引用 | 如果变量不存在则为空 |
| `${VAR:-default}` | 带默认值 | 变量不存在或为空时使用 default |
| `${VAR-default}` | 仅未设置时默认 | 仅变量不存在时使用 default |
| `${VAR:?error}` | 必需变量 | 变量不存在时报错 error |
| `${VAR?error}` | 严格必需 | 变量不存在或为空时报错 error |

### 2.2 核心解析函数

**平衡括号提取：** [extractBalancedBraceContent()](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/bootstrap/helpers/services.php#L28-L63)

```php
// 使用深度计数正确处理嵌套括号
$depth = 1;
while ($pos < $len && $depth > 0) {
    if ($str[$pos] === '{') $depth++;
    elseif ($str[$pos] === '}') $depth--;
    $pos++;
}
```

**操作符分割：** [splitOnOperatorOutsideNested()](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/bootstrap/helpers/services.php#L71-L97)

```php
// 仅在 depth=0 时匹配操作符，避免匹配嵌套括号内的 :-
foreach ($operators as $op) {
    if (substr($content, $i, strlen($op)) === $op && $depth === 0) {
        return [
            'variable' => substr($content, 0, $i),
            'operator' => $op,
            'default' => substr($content, $i + strlen($op)),
        ];
    }
}
```

**变量替换：** [replaceVariables()](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/bootstrap/helpers/services.php#L99-L137)

### 2.3 嵌套变量支持

Coolify 支持任意深度的嵌套变量语法：

```yaml
# 单层嵌套
DATABASE_URL: ${DATABASE_URL:-${SERVICE_URL_POSTGRES}/mydb}

# 多层嵌套
API_URL: ${API_URL:-${BACKEND_URL:-${SERVICE_URL_API}/v1}}

# 默认值内多个变量
CONFIG: ${CONFIG:-${SERVICE_URL_HOST}:${SERVICE_PORT}/config}
```

**测试验证：** [NestedEnvironmentVariableParsingTest.php](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/tests/Unit/NestedEnvironmentVariableParsingTest.php)

### 2.4 Compose 解析流程

解析入口：[applicationParser()](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/bootstrap/helpers/parsers.php#L361-L1502)

**处理步骤：**
1. 解析 YAML 获取 `services.<name>.environment`
2. 转换为键值对集合 [convertToKeyValueCollection()](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/bootstrap/helpers/shared.php#L3480-L3520)
3. 检测 `SERVICE_*` 魔术变量引用，自动生成对应变量
4. 遍历所有环境变量，识别 `${...}` 语法
5. 对带默认值的变量，调用 `firstOrCreate()` 存入数据库
6. 递归处理默认值中的嵌套变量

---

## 第三阶段：密钥/引用处理 (Secret/Reference Handling)

### 3.1 数据库加密

所有环境变量值在数据库中自动加密存储：

```php
// EnvironmentVariable.php
protected $casts = [
    'value' => 'encrypted',  // Laravel 的 encrypted cast
];

// 读取时自动解密
private function get_environment_variables(?string $environment_variable = null): ?string
{
    if (! $environment_variable) return null;
    return trim(decrypt($environment_variable));
}

// 写入时自动加密
private function set_environment_variables(?string $environment_variable = null): ?string
{
    $environment_variable = trim($environment_variable);
    return encrypt($environment_variable);
}
```

### 3.2 共享变量解析时机

共享变量 `{{team.VAR}}` 在两个阶段被解析：

**阶段 1 - 写入 Compose 时：**
```php
// parsers.php L1301-L1306
if (is_string($value) && str_contains($value, '{{')) {
    $value = resolveSharedEnvironmentVariables($value, $resource);
}
```

**阶段 2 - 生成 .env 文件时：**
```php
// EnvironmentVariable.php L171-L210
public function realValue(): Attribute
{
    return Attribute::make(
        get: function () {
            $real_value = $this->get_real_environment_variables($this->value, $resource);
            // 根据 is_literal/is_multiline 决定转义方式
            return $this->is_literal || $this->is_multiline
                ? "'".$real_value."'"
                : escapeEnvVariables($real_value);
        }
    );
}
```

### 3.3 值转义策略

根据 `is_literal` 和 `is_multiline` 标志采用不同转义：

| 类型 | 转义方式 | 适用场景 |
|-----|---------|---------|
| 普通变量 | `escapeEnvVariables()` + 双引号 | 允许 `$VAR` 扩展 |
| 字面量 (is_literal) | 单引号包裹 | 禁止 `$` 扩展，保留原始值 |
| 多行 (is_multiline) | 单引号包裹 | 保留换行符 |

**转义函数：**

- [escapeEnvVariables()](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/bootstrap/helpers/docker.php#L1267-L1273) - 转义 `\`、`\r`、`\t`、`\0`、`"`、`'`
- [escapeDollarSign()](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/bootstrap/helpers/docker.php#L1274-L1279) - `$` → `$$`（用于 Docker 标签）
- [escapeBashEnvValue()](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/bootstrap/helpers/docker.php#L1289-L1302) - 单引号包裹，内部 `'` → `'\''`
- [escapeBashDoubleQuoted()](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/bootstrap/helpers/docker.php#L1313-L1356) - 双引号包裹，保留 `$VAR` 扩展

> **重要设计决策**：普通变量使用双引号允许容器内 `$VAR` 扩展，而字面量使用单引号完全禁止扩展。

---

## 第四阶段：容器注入 (Container Injection)

### 4.1 双轨注入机制

Coolify 使用 **双轨注入** 确保环境变量正确到达容器：

```
                          ┌─────────────────────┐
                          │  docker-compose.yml  │
                          │                     │
┌─────────────┐         │  services:          │
│   .env 文件  │────────▶│    env_file:        │
│ (工作目录)   │         │      - .env         │
└─────────────┘         │                     │
                          │    environment:     │◀─── 内联变量
                          │      KEY=VALUE     │
                          └─────────────────────┘
```

**注入点 1 - env_file:**
```php
// parsers.php L1420-L1428
$existingEnvFiles = data_get($service, 'env_file');
$envFiles = collect(is_null($existingEnvFiles) ? [] : 
    (is_array($existingEnvFiles) ? $existingEnvFiles : [$existingEnvFiles]))
    ->push('.env')
    ->unique()
    ->values();
$payload['env_file'] = $envFiles;
```

**注入点 2 - environment:**
```php
// parsers.php L1411-L1413
if ($environment->count() > 0 || $coolifyEnvironments->count() > 0) {
    $payload['environment'] = $environment
        ->merge($coolifyEnvironments)
        ->merge($serviceNameEnvironments);
}
```

### 4.2 构建时 vs 运行时分离

Coolify 严格区分构建时和运行时环境变量：

| 特性 | 构建时 (Build-time) | 运行时 (Runtime) |
|-----|---------------------|-----------------|
| 文件位置 | `/artifacts/build-time.env` | `<workdir>/.env` |
| 过滤条件 | `is_buildtime = true` | `is_runtime = true` |
| 生成函数 | [save_buildtime_environment_variables()](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/app/Jobs/ApplicationDeploymentJob.php#L1812-L1848) | [save_runtime_environment_variables()](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/app/Jobs/ApplicationDeploymentJob.php#L1484-L1591) |
| Docker 缓存影响 | 变量变化会导致缓存失效 | 不影响构建缓存 |
| 包含变量 | Nixpacks 检测 + COOLIFY_* + 用户构建变量 | COOLIFY_* + SERVICE_* + 用户运行变量 |

**执行时序：**
```
1. 部署开始
   ↓
2. save_buildtime_environment_variables()  ← 构建前执行
   ↓
3. Docker 构建 (使用 build-time.env)
   ↓
4. save_runtime_environment_variables()   ← 构建后执行，覆盖 .env
   ↓
5. 容器启动 (使用 .env)
```

### 4.3 构建时变量优先级 (Build-time)

[generate_buildtime_environment_variables()](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/app/Jobs/ApplicationDeploymentJob.php#L1593-L1810)

```php
// 优先级从低到高：
$envs_dict = [];

// 1. Nixpacks 计划变量 (最低优先级)
foreach ($planVariables as $key => $value) {
    $envs_dict[$key] = escapeBashEnvValue($value);
}

// 2. COOLIFY_* 系统变量
foreach ($coolify_envs as $key => $item) {
    $envs_dict[$key] = escapeBashEnvValue($item);
}

// 3. SERVICE_* 变量 (Docker Compose)
foreach ($services as $serviceName => $_) {
    $envs_dict['SERVICE_NAME_...'] = escapeBashEnvValue($serviceName);
}

// 4. 用户自定义变量 (最高优先级)
foreach ($sorted_environment_variables as $env) {
    $envs_dict[$env->key] = $escapedValue;  // 覆盖之前的
}
```

### 4.4 运行时变量优先级 (Runtime)

[generate_runtime_environment_variables()](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/app/Jobs/ApplicationDeploymentJob.php#L1309-L1473)

```php
$envs = collect([]);

// 1. COOLIFY_* 系统变量
$coolify_envs->each(function ($item, $key) use ($envs) {
    $envs->push($key.'='.$item);
});

// 2. SERVICE_* 变量 (Docker Compose)
foreach ($domains as $forServiceName => $domain) {
    $envs->push('SERVICE_URL_...='.$coolifyUrl);
    $envs->push('SERVICE_FQDN_...='.$coolifyFqdn);
}

// 3. 引用 SERVICE_* 的用户变量放到后面
$runtime_environment_variables = $runtime_environment_variables->sortBy(function ($env) {
    if (str($env->value)->contains('${SERVICE_')) {
        return 2;  // 后面处理，确保 SERVICE_* 已定义
    }
    return 1;
});

// 4. 用户自定义变量
foreach ($runtime_environment_variables as $env) {
    $envs->push($env->key.'='.$env->getResolvedValueWithServer(...));
}

// 5. 预览部署的生产变量回退
if ($runtime_environment_variables_preview->isNotEmpty()) {
    // 预览变量未覆盖的 key，使用生产环境值
}
```

---

## 优先级与覆盖关系总结

### 5.1 综合优先级表

| 层级 | 来源 | 优先级 | 覆盖说明 |
|-----|------|-------|---------|
| 1 | 用户自定义变量（UI 中设置） | 🔴 最高 | 覆盖所有其他来源 |
| 2 | Coolify 自动生成 (COOLIFY_*, SERVICE_*) | 🟠 高 | 覆盖模板和默认值，但不覆盖用户设置 |
| 3 | Docker Compose environment 字段 | 🟡 中 | 从模板解析，优先级低于数据库中的用户变量 |
| 4 | 共享环境变量 ({{team.VAR}}) | 🟡 中 | 引用解析为实际值，不单独排序 |
| 5 | Nixpacks 自动检测变量 | 🟢 低 | 仅在没有用户定义时生效 |
| 6 | ${VAR:-default} 默认值 | ⚪ 最低 | 仅在变量完全未定义时使用 |

### 5.2 同级别排序

- **默认**：按 `id` 升序（创建顺序）
- **启用排序后**：按 `key` 字母顺序
- **特殊规则**：引用 `$SERVICE_*` 的变量自动排到后面

### 5.3 预览部署特殊规则

1. 预览变量优先于生产变量
2. 如果预览变量存在但缺少某些 key，自动回退到生产变量值
3. `SERVICE_*` 变量针对预览重新生成（含 PR ID）

### 5.4 Docker Compose 优先级

在容器内，Docker Compose 按以下优先级确定最终值：

```
1. docker-compose.yml 的 environment: 字段  ← 最高
2. .env 文件 (env_file: 指定)
3. 主机环境变量
4. Dockerfile 中的 ENV 指令  ← 最低
```

> **关键设计**：Coolify 将 `${VAR}` 形式的引用保留在 `environment:` 字段中（不解析），让 Docker Compose 从 `.env` 文件解析。这样用户更新变量后无需重新解析 Compose 文件即可生效。

---

## 关键代码位置索引

| 功能 | 文件 | 行号 |
|-----|------|------|
| 平衡括号提取 | [services.php](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/bootstrap/helpers/services.php) | L28-L63 |
| 操作符分割 | [services.php](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/bootstrap/helpers/services.php) | L71-L97 |
| 变量替换 | [services.php](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/bootstrap/helpers/services.php) | L99-L137 |
| 应用解析器 | [parsers.php](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/bootstrap/helpers/parsers.php) | L361-L1502 |
| 服务解析器 | [parsers.php](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/bootstrap/helpers/parsers.php) | L1504+ |
| 共享变量解析 | [shared.php](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/bootstrap/helpers/shared.php) | L4138-L4176 |
| 环境变量模型 | [EnvironmentVariable.php](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/app/Models/EnvironmentVariable.php) | 全文 |
| 共享变量模型 | [SharedEnvironmentVariable.php](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/app/Models/SharedEnvironmentVariable.php) | 全文 |
| 生成运行时变量 | [ApplicationDeploymentJob.php](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/app/Jobs/ApplicationDeploymentJob.php) | L1309-L1473 |
| 生成构建时变量 | [ApplicationDeploymentJob.php](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/app/Jobs/ApplicationDeploymentJob.php) | L1593-L1810 |
| 保存运行时 .env | [ApplicationDeploymentJob.php](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/app/Jobs/ApplicationDeploymentJob.php) | L1484-L1591 |
| 保存构建时 .env | [ApplicationDeploymentJob.php](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/app/Jobs/ApplicationDeploymentJob.php) | L1812-L1848 |
| 生成 COOLIFY_* 变量 | [ApplicationDeploymentJob.php](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/app/Jobs/ApplicationDeploymentJob.php) | L2942-L3037 |
| Bash 转义函数 | [docker.php](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/bootstrap/helpers/docker.php) | L1267-L1356 |

---

## 测试验证位置

| 测试内容 | 文件 |
|---------|------|
| 嵌套变量解析 | [NestedEnvironmentVariableParsingTest.php](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/tests/Unit/NestedEnvironmentVariableParsingTest.php) |
| 嵌套变量整体流程 | [NestedEnvironmentVariableTest.php](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/tests/Unit/NestedEnvironmentVariableTest.php) |
| Bash 转义测试 | [BashEnvEscapingTest.php](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/tests/Unit/BashEnvEscapingTest.php) |
| 环境变量边界情况 | [EnvironmentVariableParsingEdgeCasesTest.php](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/tests/Unit/EnvironmentVariableParsingEdgeCasesTest.php) |
| 魔术变量覆盖 | [ServiceMagicVariableOverwriteTest.php](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/tests/Feature/ServiceMagicVariableOverwriteTest.php) |
| 多行环境变量 | [MultilineEnvironmentVariableTest.php](file:///d:/fz/0601-2/solo-dogfeeding/code/20-coolify/tests/Feature/MultilineEnvironmentVariableTest.php) |
