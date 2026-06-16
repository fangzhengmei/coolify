# Docker Compose 编排服务到平台资源模型映射规则

## 一、整体架构与三段处理协作流程

### 1.1 核心入口

Coolify 对 docker-compose 编排清单的处理分为三个核心阶段，它们通过数据流转形成完整的处理链路：

```
用户输入 docker-compose.yml
       ↓
[阶段一] 编排清单解析（YAML 解析 + 安全验证）
       ↓ 解析后的 services 数组
[阶段二] 环境变量注入（魔法变量识别 + 变量替换 + 存储）
       ↓ 注入环境变量后的 service 配置
[阶段三] 多容器资源生成（ServiceApplication/ServiceDatabase 模型创建 + 卷/网络/标签处理）
       ↓
生成最终可部署的 docker-compose.yml + 平台资源模型
```

### 1.2 核心处理函数

| 资源类型 | 解析入口函数 | 所在文件 |
|---------|-------------|---------|
| Application（单应用多容器） | `applicationParser()` | [parsers.php](file:///d:/fz/0601-1/solo-dogfeeding/code/92-coolify/bootstrap/helpers/parsers.php#L361-L1502) |
| Service（服务模板多容器） | `serviceParser()` | [parsers.php](file:///d:/fz/0601-1/solo-dogfeeding/code/92-coolify/bootstrap/helpers/parsers.php#L1504-L2100) |
| 安全验证 | `validateDockerComposeForInjection()` | [parsers.php](file:///d:/fz/0601-1/solo-dogfeeding/code/92-coolify/bootstrap/helpers/parsers.php#L27-L101) |

### 1.3 调用链示例（以 Service 为例）

1. 用户提交表单 → [DockerCompose.php:submit()](file:///d:/fz/0601-1/solo-dogfeeding/code/92-coolify/app/Livewire/Project/New/DockerCompose.php#L30-L88)
2. 创建 Service 模型并保存 `docker_compose_raw`
3. 调用 `$service->parse(isNew: true)` → [Service.php:parse()](file:///d:/fz/0601-1/solo-dogfeeding/code/92-coolify/app/Models/Service.php#L1606-L1615)
4. 根据 `compose_parsing_version` 路由到 `serviceParser()` 或旧版 `parseDockerComposeFile()`

---

## 二、阶段一：容器编排清单解析

### 2.1 安全验证（前置步骤）

在保存到数据库之前，必须先通过 `validateDockerComposeForInjection()` 进行命令注入防护检查：

**验证内容：**
- YAML 格式合法性
- 必须包含 `services` 节
- 服务名称不能包含 shell 元字符
- Volume 定义（字符串和数组两种格式）的 source/target 路径安全检查
- 环境变量引用（`${VAR}`）的特殊放行规则

**关键代码：** [parsers.php:L27-L101](file:///d:/fz/0601-1/solo-dogfeeding/code/92-coolify/bootstrap/helpers/parsers.php#L27-L101)

### 2.2 YAML 解析与结构提取

```php
// parsers.php:L376-L387
try {
    $yaml = Yaml::parse($compose);
} catch (Exception) {
    return collect([]);  // 降级：解析失败返回空集合
}
$services = data_get($yaml, 'services', collect([]));
$topLevel = collect([
    'volumes' => collect(data_get($yaml, 'volumes', [])),
    'networks' => collect(data_get($yaml, 'networks', [])),
    'configs' => collect(data_get($yaml, 'configs', [])),
    'secrets' => collect(data_get($yaml, 'secrets', [])),
]);
```

### 2.3 Volume 字符串解析

`parseDockerVolumeString()` 函数处理复杂的 volume 定义格式：

**支持的格式：**
- `myvolume` - 命名卷
- `gitea:/data` - 简单映射
- `${VAR:-default}:/data` - 带默认值的环境变量
- `gitea:/data:ro` - 带模式
- `C:\path\on\windows:/data` - Windows 路径
- `${VAR}/path:/data` - 环境变量+路径拼接

**关键处理逻辑：** [parsers.php:L116-L359](file:///d:/fz/0601-1/solo-dogfeeding/code/92-coolify/bootstrap/helpers/parsers.php#L116-L359)

1. 特殊处理 `${VAR:-default}` 中的冒号（避免误判为分隔符）
2. 识别 Windows 驱动器号（`C:`）避免误分割
3. 提取 volume mode（ro, rw, cached 等）
4. 解析环境变量默认值并应用
5. 安全验证 source/target 路径

---

## 三、阶段二：环境变量注入

### 3.1 环境变量格式统一

`convertToKeyValueCollection()` 将不同格式的环境变量统一为 key-value 集合：

**支持的输入格式：**
```yaml
# 数组格式（列表）
environment:
  - FOO=bar
  - BAZ=qux

# 映射格式（对象）
environment:
  FOO: bar
  BAZ: qux
```

**关键代码：** [shared.php:L3480-L3520](file:///d:/fz/0601-1/solo-dogfeeding/code/92-coolify/bootstrap/helpers/shared.php#L3480-L3520)

### 3.2 魔法变量（Magic Variables）处理

Coolify 支持特殊的 `SERVICE_*` 前缀魔法变量，用于自动生成服务间通信地址。

#### 3.2.1 魔法变量识别

```php
// parsers.php:L439-L455
$regex = '/\$(\{?([a-zA-Z_\x80-\xff][a-zA-Z0-9_\x80-\xff]*)\}?)/';
preg_match_all($regex, $value, $valueMatches);
if (count($valueMatches[2]) > 0) {
    foreach ($valueMatches[2] as $match) {
        $match = str($match);
        if ($match->startsWith('SERVICE_')) {
            $magicEnvironments->put($match->value(), '');
        }
    }
}
```

#### 3.2.2 FQDN/URL 魔法变量

**支持的变量格式：**
- `SERVICE_FQDN_APP` - 生成服务的域名（无协议）
- `SERVICE_URL_APP` - 生成服务的 URL（含协议）
- `SERVICE_FQDN_APP_3000` - 指定端口的域名
- `SERVICE_URL_APP_3000` - 指定端口的 URL

**处理逻辑：** [parsers.php:L458-L640](file:///d:/fz/0601-1/solo-dogfeeding/code/92-coolify/bootstrap/helpers/parsers.php#L458-L640)

1. 解析变量名提取服务名和端口
2. 调用 `generateFqdn()` / `generateUrl()` 生成地址
3. 同时创建 FQDN 和 URL 配对变量（无论模板中写了哪个）
4. 保存到 `environment_variables` 表
5. 更新 `docker_compose_domains` JSON 字段

### 3.3 普通环境变量处理

#### 3.3.1 变量引用语法支持

```yaml
environment:
  # 简单引用 - 运行时由 Docker Compose 从 .env 解析
  DATABASE_URL: ${DATABASE_URL}
  
  # 带默认值
  DB_HOST: ${DB_HOST:-localhost}
  
  # 必选变量（无默认值，缺失时报错）
  API_KEY: ${API_KEY:?API_KEY is required}
  
  # 嵌套变量
  COMPOSED: ${PREFIX:-app}-${SUFFIX:-local}
```

#### 3.3.2 处理流程

**关键代码：** [parsers.php:L967-L1142](file:///d:/fz/0601-1/solo-dogfeeding/code/92-coolify/bootstrap/helpers/parsers.php#L967-L1142)

1. **简单变量引用**（`${VAR}`）：
   - 使用 `firstOrCreate()` 创建环境变量记录
   - `value` 字段留空（由用户后续填写）
   - compose 文件中保留 `${VAR}` 引用，部署时从 `.env` 解析

2. **带默认值**（`${VAR:-default}`）：
   - 提取变量名和默认值
   - 创建环境变量时设置默认值
   - 标记 `is_required` 为 `false`
   - 递归处理默认值中的嵌套变量

3. **必选变量**（`${VAR:?error message}`）：
   - 提取变量名和错误消息
   - 创建环境变量时标记 `is_required = true`
   - 值留空，需用户填写

### 3.4 Coolify 系统环境变量注入

自动注入的系统变量：
- `COOLIFY_BRANCH` - Git 分支名
- `COOLIFY_RESOURCE_UUID` - 资源 UUID
- `COOLIFY_CONTAINER_NAME` - 容器名
- `COOLIFY_URL` / `COOLIFY_FQDN` - 应用访问地址
- `SERVICE_URL_*` / `SERVICE_FQDN_*` - 服务间通信地址

### 3.5 变量替换辅助函数

`replaceVariables()` 处理各种变量格式：
- `${VAR}` → 提取 `VAR`
- `$VAR` → 提取 `VAR`
- 嵌套变量支持

**关键代码：** [services.php:L99-L137](file:///d:/fz/0601-1/solo-dogfeeding/code/92-coolify/bootstrap/helpers/services.php#L99-L137)

---

## 四、阶段三：多容器资源生成

### 4.1 服务类型判定与资源创建

#### 4.1.1 数据库识别

`isDatabaseImage()` 根据镜像名和服务配置识别数据库服务：

**关键代码：** [docker.php:L795-L846](file:///d:/fz/0601-1/solo-dogfeeding/code/92-coolify/bootstrap/helpers/docker.php#L795-L846)

**识别规则：**
- 镜像名匹配已知数据库模式（mysql, postgres, mongo, redis 等）
- 排除假阳性（如 `supertokens/supertokens-mysql` 不是数据库）
- 检查服务配置中的环境变量（如 `MYSQL_ROOT_PASSWORD`）

#### 4.1.2 资源模型创建

```php
// parsers.php:L1594-L1616
if ($isDatabase) {
    $savedService = ServiceDatabase::firstOrCreate([
        'name' => $serviceName,
        'service_id' => $resource->id,
    ]);
} else {
    $savedService = ServiceApplication::firstOrCreate([
        'name' => $serviceName,
        'service_id' => $resource->id,
    ], [
        'is_gzip_enabled' => true,
    ]);
}
```

**模型关系：**
- `Service` 1 → N `ServiceApplication`
- `Service` 1 → N `ServiceDatabase`
- `ServiceApplication` / `ServiceDatabase` 各有独立的配置（fqdn, 环境变量, 卷等）

### 4.2 容器名称生成规则

```php
// 通用格式
$containerName = "$serviceName-$baseName";

// Application（支持 PR 部署）
$baseName = generateApplicationContainerName($resource, $pull_request_id);
// 如 PR #123：serviceName-app-uuid-123

// Service
$containerName = "$serviceName-{$resource->uuid}";
```

### 4.3 Volume 处理

#### 4.3.1 Bind Mount（本地绑定）

```php
// parsers.php:L774-L824
if ($type->value() === 'bind') {
    $mainDirectory = str(base_configuration_dir().'/applications/'.$uuid);
    $source = replaceLocalSource($source, $mainDirectory);
    // PR 部署时添加后缀
    if ($isPullRequest) {
        $source = addPreviewDeploymentSuffix($source, $pull_request_id);
    }
    LocalFileVolume::updateOrCreate([...]);
}
```

#### 4.3.2 Named Volume（命名卷）

```php
// parsers.php:L825-L869
$name = "{$uuid}_{$slugWithoutUuid}";
if ($isPullRequest) {
    $name = addPreviewDeploymentSuffix($name, $pull_request_id);
}
LocalPersistentVolume::updateOrCreate([...]);
$topLevel->get('volumes')->put($name, ['name' => $name]);
```

### 4.4 网络处理

```php
// parsers.php:L891-L965
if (! $use_network_mode) {
    // 添加应用专属网络
    $baseNetwork = collect([$uuid]);
    if ($isPullRequest) {
        $baseNetwork = collect(["{$uuid}-{$pullRequestId}"]);
    }
    // 自动连接到 Coolify 代理网络
    if (data_get($resource, 'settings.connect_to_docker_network')) {
        $network = $resource->destination->network;
        $networks_temp->put($network, null);
        $topLevel->get('networks')->put($network, [
            'name' => $network,
            'external' => true,
        ]);
    }
}
```

### 4.5 代理标签生成

根据服务器配置的代理类型（Traefik 或 Caddy）生成对应的路由标签：

```php
// parsers.php:L1330-L1383
switch ($server->proxyType()) {
    case ProxyTypes::TRAEFIK->value:
        $serviceLabels = $serviceLabels->merge(fqdnLabelsForTraefik(...));
        break;
    case ProxyTypes::CADDY->value:
        $serviceLabels = $serviceLabels->merge(fqdnLabelsForCaddy(...));
        break;
}
```

### 4.6 最终 Compose 生成

```php
// parsers.php:L1449-L1471
$topLevel->put('services', $parsedServices);
$customOrder = ['services', 'volumes', 'networks', 'configs', 'secrets'];
$topLevel = $topLevel->sortBy(function ($value, $key) use ($customOrder) {
    return array_search($key, $customOrder);
});
// 过滤空节
$topLevel = $topLevel->filter(function ($value, $key) {
    if ($key === 'services') return true;
    return $value instanceof Collection ? $value->isNotEmpty() : ! empty($value);
});
$cleanedCompose = Yaml::dump(convertToArray($topLevel), 10, 2);
$resource->docker_compose = $cleanedCompose;
```

---

## 五、完整映射规则汇总表

| Compose 字段 | 平台处理逻辑 | 目标模型/字段 |
|-------------|-------------|--------------|
| `services.<name>` | 根据镜像判定类型，创建子资源 | `ServiceApplication` / `ServiceDatabase` |
| `services.<name>.image` | 更新到子资源 | `ServiceApplication.image` |
| `services.<name>.environment` | 解析变量，存储到环境变量表 | `EnvironmentVariable` |
| `services.<name>.volumes` | 解析为 LocalFileVolume 或 LocalPersistentVolume | `LocalFileVolume` / `LocalPersistentVolume` |
| `services.<name>.ports` | 保留到最终 compose，生成端口映射 | compose `ports` 节 |
| `services.<name>.depends_on` | PR 部署时添加后缀，保留依赖关系 | compose `depends_on` 节 |
| `services.<name>.networks` | 合并基础网络和自定义网络 | compose `networks` 节 + 顶层 `networks` |
| `services.<name>.labels` | 合并 Coolify 默认标签 + 代理标签 | compose `labels` 节 |
| `services.<name>.build` | 无镜像时自动注入 commit 标签 | compose `image` 字段 |
| `volumes.<name>` | 命名卷重命名（添加 uuid 前缀） | 顶层 `volumes` 节 |
| `networks.<name>` | 除 default 外保留并标记 external | 顶层 `networks` 节 |

---

## 六、非法清单或缺失变量时的降级策略

### 6.1 YAML 解析失败

**降级策略：** 返回空集合，跳过后续处理

```php
// parsers.php:L376-L380
try {
    $yaml = Yaml::parse($compose);
} catch (Exception) {
    return collect([]);
}
```

**影响：**
- `docker_compose` 字段不会更新
- 不会创建任何子资源（ServiceApplication/ServiceDatabase）
- 已有资源不受影响

### 6.2 安全验证失败（命令注入检测）

**降级策略：** 抛出异常，终止处理流程

```php
// parsers.php:L27-L33
try {
    $parsed = Yaml::parse($composeYaml);
} catch (Exception $e) {
    throw new Exception('Invalid YAML format: '.$e->getMessage(), 0, $e);
}
```

**触发场景：**
- 服务名称包含 shell 元字符
- Volume 路径包含危险字符
- 非法的环境变量格式

### 6.3 缺少 `services` 节

**降级策略：** 抛出异常

```php
// parsers.php:L35-L37
if (! is_array($parsed) || ! isset($parsed['services']) || ! is_array($parsed['services'])) {
    throw new Exception('Docker Compose file must contain a "services" section');
}
```

### 6.4 环境变量缺失

#### 6.4.1 可选变量（无默认值）

**降级策略：** 
- 创建环境变量记录，`value` 留空
- `is_required = false`
- compose 中保留 `${VAR}` 引用
- 部署时由 Docker Compose 从 `.env` 解析，若未设置则为空字符串

#### 6.4.2 必选变量（`${VAR:?}` 语法）

**降级策略：**
- 创建环境变量记录，标记 `is_required = true`
- 通过 `is_really_required` 访问器判断是否真正缺失

```php
// EnvironmentVariable.php:L212-L217
protected function isReallyRequired(): Attribute
{
    return Attribute::make(
        get: fn () => $this->is_required && str($this->real_value)->isEmpty(),
    );
}
```

- `isDeployable()` 检查会阻止部署

```php
// Service.php:L1622-L1636
protected function isDeployable(): Attribute
{
    return Attribute::make(
        get: function () {
            $envs = $this->environment_variables()->where('is_required', true)->get();
            foreach ($envs as $env) {
                if ($env->is_really_required) {
                    return false;  // 有必填变量未填写，不可部署
                }
            }
            return true;
        }
    );
}
```

**UI 表现：**
- 环境变量输入框边框显示红色（`border-error`）
- 部署按钮禁用或显示警告

### 6.5 Volume 解析失败

**降级策略：** 抛出异常，包含具体错误信息

```php
// parsers.php:L326-L335
try {
    validateShellSafePath($sourceStr, 'volume source');
} catch (Exception $e) {
    throw new Exception(
        'Invalid Docker volume definition: '.$e->getMessage().
        ' Please use safe path names without shell metacharacters.'
    );
}
```

### 6.6 镜像识别失败（数据库检测）

**降级策略：** 默认为应用类型（`ServiceApplication`）

```php
// docker.php:L795-L846
function isDatabaseImage(?string $image = null, ?array $serviceConfig = null)
{
    if (is_null($image)) {
        return false;  // 无镜像时默认不是数据库
    }
    // ... 匹配逻辑 ...
    return $isKnownDatabase;
}
```

### 6.7 魔法变量引用不存在的服务

**降级策略：**
- 环境变量仍然创建，但值可能为生成的占位域名
- `docker_compose_domains` 中不会添加该服务的域名记录
- 部署时容器间通信可能失败，需用户手动配置

```php
// parsers.php:L602-L628
$serviceExists = false;
foreach ($services as $serviceNameKey => $service) {
    $transformedServiceName = str($serviceNameKey)->replace('-', '_')->replace('.', '_')->value();
    if ($transformedServiceName === $serviceName) {
        $serviceExists = true;
        break;
    }
}
if ($serviceExists) {
    // 添加域名记录
}
// 否则跳过
```

### 6.8 嵌套变量解析失败

**降级策略：** 回退到旧的简单字符串分割逻辑

```php
// parsers.php:L1094-L1140
} else {
    // Fallback to old behavior for malformed input
    if ($value->contains(':-')) {
        $value = replaceVariables($value);
        $key = $value->before(':');
        $value = $value->after(':-');
    }
    // ... 其他回退逻辑 ...
}
```

### 6.9 旧版解析器兼容性

**降级策略：** 根据 `compose_parsing_version` 选择解析器

```php
// Service.php:L1606-L1615
public function parse(bool $isNew = false): Collection
{
    if ((int) $this->compose_parsing_version >= 3) {
        return serviceParser($this);
    } elseif ($this->docker_compose_raw) {
        return parseDockerComposeFile($this, $isNew);
    } else {
        return collect([]);
    }
}
```

---

## 七、关键数据流转图示

```
docker-compose.yml 输入
     │
     ▼
┌─────────────────────┐
│ 安全验证            │ validateDockerComposeForInjection()
│  - YAML 格式        │
│  - 服务名称安全     │
│  - Volume 路径安全  │
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│ YAML 解析           │ Yaml::parse()
│ 提取 services       │
│ 提取顶层 volumes/   │
│ networks/configs/   │
│ secrets             │
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│ 第一轮：魔法变量识别 │
│ 遍历所有服务        │
│ 收集 SERVICE_* 变量 │
│ 生成 FQDN/URL       │
│ 保存到 env 表       │
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│ 第二轮：服务预创建   │
│ 镜像识别            │ isDatabaseImage()
│ 创建 ServiceApp/DB  │ firstOrCreate()
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│ 第三轮：完整解析     │
│ 环境变量处理        │ convertToKeyValueCollection()
│  - 变量引用解析     │ extractBalancedBraceContent()
│  - 默认值提取       │ splitOnOperatorOutsideNested()
│  - 必填标记         │ is_required
│ Volume 解析         │ parseDockerVolumeString()
│ 网络处理            │
│ 标签生成            │ fqdnLabelsForTraefik/Caddy()
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│ 生成最终 Compose     │
│ 排序节顺序          │ services → volumes → networks
│ 过滤空节            │
│ 保存 docker_compose │
│ 保存 docker_compose_raw（清理后） │
└─────────────────────┘
```
