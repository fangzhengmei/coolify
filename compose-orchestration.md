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

## 七、单应用镜像与构建分流

### 7.1 构建与镜像的分流判定逻辑

Coolify 在解析时会根据服务配置中是否存在 `build` 和 `image` 指令来决定镜像的来源：

**分流规则：**

| build 存在 | image 存在 | 行为 |
|-----------|-----------|------|
| 否 | 是 | 直接使用指定镜像 |
| 是 | 是 | 使用用户指定的 image，build 配置保留用于构建 |
| 是 | 否 | 自动注入 commit hash 作为镜像标签，用于回滚支持 |

**关键代码：** [parsers.php:L1430-L1441](file:///d:/fz/0601-1/solo-dogfeeding/code/92-coolify/bootstrap/helpers/parsers.php#L1430-L1441)

```php
// 注入 commit-based 镜像标签（仅当服务有 build 但无显式 image 时）
$hasBuild = data_get($service, 'build') !== null;
$hasImage = data_get($service, 'image') !== null;
if ($hasBuild && ! $hasImage && $commit) {
    $imageTag = str($commit)->substr(0, 128)->value();
    if ($isPullRequest) {
        $imageTag = "pr-{$pullRequestId}";
    }
    $imageRepo = "{$uuid}_{$serviceName}";
    $payload['image'] = "{$imageRepo}:{$imageTag}";
}
```

### 7.2 自动注入镜像的命名规则

**仓库名：** `{应用uuid}_{服务名}`
**标签：**
- 普通部署：`{commit hash 前128字符}`
- PR 预览部署：`pr-{pull_request_id}`

### 7.3 从 Dockerfile 自动检测端口

如果服务使用 `build` 指令，Coolify 会尝试从 Dockerfile 中解析 `EXPOSE` 指令来获取预设端口：

**关键代码：** [docker.php:L200-L216](file:///d:/fz/0601-1/solo-dogfeeding/code/92-coolify/bootstrap/helpers/docker.php#L200-L216)

```php
function get_port_from_dockerfile($dockerfile): ?int
{
    $dockerfile_array = explode("\n", $dockerfile);
    $found_exposed_port = null;
    foreach ($dockerfile_array as $line) {
        $line_str = str($line)->trim();
        if ($line_str->startsWith('EXPOSE')) {
            $found_exposed_port = $line_str->replace('EXPOSE', '')->trim();
            break;
        }
    }
    if ($found_exposed_port) {
        return (int) $found_exposed_port->value();
    }
    return null;
}
```

---

## 八、容器命名规则详解

### 8.1 基础命名公式

容器名由两部分组成：服务名 + 基础名

```
{serviceName}-{baseName}
```

**关键代码：** [parsers.php:L686-L690](file:///d:/fz/0601-1/solo-dogfeeding/code/92-coolify/bootstrap/helpers/parsers.php#L686-L690)

```php
$baseName = generateApplicationContainerName(
    application: $resource,
    pull_request_id: $pullRequestId
);
$containerName = "$serviceName-$baseName";
```

### 8.2 基础名生成策略

`generateApplicationContainerName()` 根据部署类型和配置生成不同的基础名：

**关键代码：** [docker.php:L184-L199](file:///d:/fz/0601-1/solo-dogfeeding/code/92-coolify/bootstrap/helpers/docker.php#L184-L199)

```php
function generateApplicationContainerName(Application $application, $pull_request_id = 0)
{
    $consistent_container_name = $application->settings->is_consistent_container_name_enabled;
    $now = now()->format('Hisu');
    if ($pull_request_id !== 0 && $pull_request_id !== null) {
        return $application->uuid.'-pr-'.$pull_request_id;
    } else {
        if ($consistent_container_name) {
            return $application->uuid;
        }
        return $application->uuid.'-'.$now;
    }
}
```

### 8.3 三种容器命名模式

| 部署类型 | 一致容器名 | 格式 | 示例 |
|---------|-----------|------|------|
| 普通部署 | 关闭 | `{service}-{uuid}-{timestamp}` | `web-app-abc123-12345678` |
| 普通部署 | 开启 | `{service}-{uuid}` | `web-app-abc123` |
| PR 预览部署 | - | `{service}-{uuid}-pr-{id}` | `web-app-abc123-pr-42` |

**说明：**
- 时间戳格式：`Hisu`（时分秒微秒），确保每次部署容器名唯一
- 一致容器名模式下，部署时会先停止旧容器再启动新容器
- PR 部署始终使用固定格式，不受一致容器名设置影响

### 8.4 SERVICE_NAME 环境变量

Coolify 会为每个 compose 服务生成对应的 `SERVICE_NAME_*` 环境变量，用于服务间通信时引用容器名：

**关键代码：** [shared.php:L3783-L3791](file:///d:/fz/0601-1/solo-dogfeeding/code/92-coolify/bootstrap/helpers/shared.php#L3783-L3791)

```php
function generateDockerComposeServiceName(mixed $services, int $pullRequestId = 0): Collection
{
    $collection = collect([]);
    foreach ($services as $serviceName => $_) {
        $collection->put(
            'SERVICE_NAME_'.str($serviceName)->replace('-', '_')->replace('.', '_')->upper(),
            addPreviewDeploymentSuffix($serviceName, $pullRequestId)
        );
    }
    return $collection;
}
```

**变量名转换规则：**
- 服务名中的 `-` 和 `.` 替换为 `_`
- 转换为大写
- 添加 `SERVICE_NAME_` 前缀

**示例：**
- 服务名 `my-app` → 变量名 `SERVICE_NAME_MY_APP`
- 服务名 `api.v2` → 变量名 `SERVICE_NAME_API_V2`

---

## 九、预览部署后缀的资源生成

### 9.1 后缀生成核心函数

`addPreviewDeploymentSuffix()` 是预览部署命名的基础工具函数：

**关键代码：** [shared.php:L3778-L3781](file:///d:/fz/0601-1/solo-dogfeeding/code/92-coolify/bootstrap/helpers/shared.php#L3778-L3781)

```php
function addPreviewDeploymentSuffix(string $name, int $pull_request_id = 0): string
{
    return ($pull_request_id === 0) ? $name : $name.'-pr-'.$pull_request_id;
}
```

### 9.2 应用后缀的资源类型

预览部署后缀 `-pr-{id}` 会应用到以下资源：

| 资源类型 | 普通部署 | 预览部署 | 可配置开关 |
|---------|---------|---------|-----------|
| 容器名 | `{service}-{uuid}` | `{service}-{uuid}-pr-{id}` | 否 |
| 服务名（compose） | `{service}` | `{service}-pr-{id}` | 否 |
| 网络名 | `{uuid}` | `{uuid}-{id}` | 否 |
| 命名卷 | `{uuid}_{volume}` | `{uuid}_{volume}-pr-{id}` | 否 |
| 绑定卷（source） | `{path}` | `{path}-pr-{id}` | 是（默认开启） |
| 镜像标签 | `{commit}` | `pr-{id}` | 否 |
| 代理标签 UUID | `{uuid}` | `{uuid}-{id}` | 否 |
| 代理网络名 | `{network}` | `{network}-{id}` | 否 |

### 9.3 绑定卷后缀的可配置性

对于本地绑定卷（bind mount），可以通过配置控制是否添加预览后缀：

**关键代码：** [parsers.php:L792-L797](file:///d:/fz/0601-1/solo-dogfeeding/code/92-coolify/bootstrap/helpers/parsers.php#L792-L797)

```php
$isPreviewSuffixEnabled = $foundConfig
    ? (bool) data_get($foundConfig, 'is_preview_suffix_enabled', true)
    : true;
if ($isPullRequest && $isPreviewSuffixEnabled) {
    $source = addPreviewDeploymentSuffix($source, $pull_request_id);
}
```

**默认行为：** 开启（`is_preview_suffix_enabled = true`）

### 9.4 预览部署域名生成

预览部署的域名通过 `preview_url_template` 模板生成，支持以下占位符：

- `{{random}}` - 随机 CUID2 字符串
- `{{domain}}` - 主应用域名的 host 部分
- `{{pr_id}}` - Pull Request ID

**关键代码：** [ApplicationPreview.php:L168-L181](file:///d:/fz/0601-1/solo-dogfeeding/code/92-coolify/app/Models/ApplicationPreview.php#L168-L181)

```php
$url = Url::fromString($domain);
$template = $this->application->preview_url_template;
$host = $url->getHost();
$schema = $url->getScheme();
$portInt = $url->getPort();
$port = $portInt !== null ? ':'.$portInt : '';
$urlPath = $url->getPath();
$path = ($urlPath !== '' && $urlPath !== '/') ? $urlPath : '';
$random = new Cuid2;
$preview_fqdn = str_replace('{{random}}', $random, $template);
$preview_fqdn = str_replace('{{domain}}', $host, $preview_fqdn);
$preview_fqdn = str_replace('{{pr_id}}', $this->pull_request_id, $preview_fqdn);
$preview_fqdn = "$schema://$preview_fqdn{$port}{$path}";
```

**生成规则：**
- 保留原始协议（http/https）
- 保留端口和路径
- 域名部分通过模板替换生成

### 9.5 预览部署域名继承机制

创建预览部署时，会从主应用继承域名配置：

**关键代码：** [Previews.php:L203](file:///d:/fz/0601-1/solo-dogfeeding/code/92-coolify/app/Livewire/Project/Application/Previews.php#L203)

```php
'docker_compose_domains' => $this->application->docker_compose_domains,
```

然后调用 `generate_preview_fqdn_compose()` 为每个服务生成预览域名。

---

## 十、SERVICE_FQDN_* 与 SERVICE_URL_* 魔法变量深层解析

### 10.1 变量名解析规则

`parseServiceEnvironmentVariable()` 函数负责从变量名中提取服务名和端口：

**关键代码：** [services.php:L423-L456](file:///d:/fz/0601-1/solo-dogfeeding/code/92-coolify/bootstrap/helpers/services.php#L423-L456)

```php
function parseServiceEnvironmentVariable(string $key): array
{
    $strKey = str($key);
    $lastSegment = $strKey->afterLast('_')->value();
    $hasPort = is_numeric($lastSegment) && ctype_digit($lastSegment);

    if ($hasPort) {
        // 端口特定变量（如 SERVICE_URL_APP_3000）
        if ($strKey->startsWith('SERVICE_URL_')) {
            $serviceName = $strKey->after('SERVICE_URL_')->beforeLast('_')->lower()->value();
        } elseif ($strKey->startsWith('SERVICE_FQDN_')) {
            $serviceName = $strKey->after('SERVICE_FQDN_')->beforeLast('_')->lower()->value();
        }
        $port = $lastSegment;
    } else {
        // 基础变量（如 SERVICE_URL_APP）
        if ($strKey->startsWith('SERVICE_URL_')) {
            $serviceName = $strKey->after('SERVICE_URL_')->lower()->value();
        } elseif ($strKey->startsWith('SERVICE_FQDN_')) {
            $serviceName = $strKey->after('SERVICE_FQDN_')->lower()->value();
        }
        $port = null;
    }

    return [
        'service_name' => $serviceName,
        'port' => $port,
        'has_port' => $hasPort,
    ];
}
```

### 10.2 端口识别逻辑

**端口判定条件：** `is_numeric($lastSegment) && ctype_digit($lastSegment)`

这意味着：
- 必须是纯数字（无小数点、无科学计数法）
- `0` 被认为是有效端口
- `3.14` 或 `1e5` 不被认为是端口

**示例：**

| 变量名 | 服务名 | 端口 | has_port |
|-------|-------|------|----------|
| `SERVICE_URL_APP` | `app` | null | false |
| `SERVICE_URL_APP_3000` | `app` | `3000` | true |
| `SERVICE_FQDN_MY_API_8080` | `my_api` | `8080` | true |
| `SERVICE_URL_REDIS_CACHE_6379` | `redis_cache` | `6379` | true |
| `SERVICE_URL_APP_0` | `app` | `0` | true |
| `SERVICE_URL_APP_3.14` | `app_3.14` | null | false |
| `SERVICE_URL_APP_1e5` | `app_1e5` | null | false |

### 10.3 协议判定逻辑

`generateUrl()` 和 `generateFqdn()` 从服务器的 wildcard_domain 配置中提取协议：

**关键代码：** [shared.php:L1001-L1037](file:///d:/fz/0601-1/solo-dogfeeding/code/92-coolify/bootstrap/helpers/shared.php#L1001-L1037)

**协议来源优先级：**

1. **自定义 wildcard_domain**（配置了的话）
2. **sslip 自动域名**（默认回退）
   - 开发环境：`http://127.0.0.1.sslip.io`
   - IPv4：`http://{ip}.sslip.io`
   - IPv6：`http://{ipv6}.sslip.io`（冒号替换为连字符）

**forceHttps 参数：** 当设置为 `true` 时，强制使用 `https` 协议。

### 10.4 FQDN 与 URL 的区别

| 类型 | 格式 | 示例 |
|-----|------|------|
| URL | `{scheme}://{host}{path}` | `https://app.example.com/api` |
| FQDN (v5+) | `{host}{path}` | `app.example.com/api` |
| FQDN (旧版) | `{scheme}://{host}{path}` | `https://app.example.com/api` |

**版本判定：** [shared.php:L1032-L1034](file:///d:/fz/0601-1/solo-dogfeeding/code/92-coolify/bootstrap/helpers/shared.php#L1032-L1034)

```php
if ($parserVersion >= 5 && version_compare(config('constants.coolify.version'), '4.0.0-beta.420.7', '>=')) {
    return "{$random}.$host$path";
}
```

### 10.5 配对生成机制

无论 compose 模板中写的是 `SERVICE_URL_*` 还是 `SERVICE_FQDN_*`，Coolify 都会**同时创建对应的两个变量**：

**关键代码：** [parsers.php:L577-L601](file:///d:/fz/0601-1/solo-dogfeeding/code/92-coolify/bootstrap/helpers/parsers.php#L577-L601)

```php
// 同时创建 FQDN 和 URL 变量
$resource->environment_variables()->updateOrCreate([
    'key' => "SERVICE_FQDN_{$serviceNamePreserved}_{$port}",
    'resourceable_type' => get_class($resource),
    'resourceable_id' => $resource->id,
], [
    'value' => $fqdnWithPort,
    'is_build_time' => false,
    'is_preview' => $isPullRequest,
]);

$resource->environment_variables()->updateOrCreate([
    'key' => "SERVICE_URL_{$serviceNamePreserved}_{$port}",
    // ...
], [
    'value' => $urlWithPort,
    // ...
]);
```

**设计意图：** 确保其他服务无论引用 URL 还是 FQDN 形式的变量，都能获得正确的值。

### 10.6 端口特定变量的生成

当检测到端口特定变量时，除了生成基础变量（无端口），还会生成带端口的版本：

**值的格式：**
- URL 带端口：`{scheme}://{host}:{port}{path}`
- FQDN 带端口：`{host}:{port}{path}`

---

## 十一、自定义域名优先级与域名记录一致性

### 11.1 docker_compose_domains 数据结构

`docker_compose_domains` 是存储在 Application/ApplicationPreview 模型上的 JSON 字符串，记录每个服务的域名配置：

**结构示例：**
```json
{
  "web": {
    "domain": "https://web.example.com,https://app.example.com"
  },
  "api": {
    "domain": "https://api.example.com:8080"
  }
}
```

**模型字段位置：**
- 主应用：[Application.php:L94](file:///d:/fz/0601-1/solo-dogfeeding/code/92-coolify/app/Models/Application.php#L94)
- 预览部署：[ApplicationPreview.php:L23](file:///d:/fz/0601-1/solo-dogfeeding/code/92-coolify/app/Models/ApplicationPreview.php#L23)

### 11.2 域名优先级

域名来源按以下优先级从高到低：

| 优先级 | 来源 | 说明 |
|-------|------|------|
| 1 | 用户自定义域名 | 用户在界面上手动设置的域名（最高优先级） |
| 2 | 自动生成的通配符域名 | 通过服务器 wildcard_domain 自动生成的域名 |
| 3 | sslip 自动域名 | 服务器 IP 的 sslip.io 域名（最低回退） |

### 11.3 updateCompose：自定义域名同步机制

当用户更新服务的 `fqdn` 时，`updateCompose()` 函数负责将自定义域名同步到 `SERVICE_*` 环境变量中：

**关键代码：** [services.php:L211-L398](file:///d:/fz/0601-1/solo-dogfeeding/code/92-coolify/bootstrap/helpers/services.php#L211-L398)

#### 11.3.1 处理流程

```
用户更新服务 fqdn
       ↓
updateCompose() 被调用
       ↓
1. 从 compose 模板中提取当前服务直接声明的 SERVICE_* 变量名
       ↓
2. 解析变量名，提取服务名和端口
       ↓
3. 删除所有旧的 SERVICE_URL_* 和 SERVICE_FQDN_* 变量
       ↓
4. 根据用户设置的 fqdn，创建新的变量值
       ↓
5. 同时创建 URL 和 FQDN 配对变量（基础版 + 端口特定版）
```

#### 11.3.2 只更新直接声明的变量

`updateCompose()` 只会更新**当前服务自己声明**的 `SERVICE_*` 变量，不会更新其他服务引用的变量：

**关键代码：** [services.php:L241-L261](file:///d:/fz/0601-1/solo-dogfeeding/code/92-coolify/bootstrap/helpers/services.php#L241-L261)

```php
// DO NOT extract variables that are only referenced with ${VAR_NAME} syntax
// Those belong to other services and will be updated when THOSE services are updated
```

**判断标准：**
- 直接声明：`SERVICE_URL_APP` 或 `SERVICE_URL_APP=value`（变量名作为 key）
- 引用：`NEXT_PUBLIC_URL=${SERVICE_URL_APP}`（变量在 value 中以 `${}` 引用）

#### 11.3.3 删除旧变量再重建

为确保一致性，先删除所有旧变量，再创建新的：

**关键代码：** [services.php:L313-L325](file:///d:/fz/0601-1/solo-dogfeeding/code/92-coolify/bootstrap/helpers/services.php#L313-L325)

```php
// Delete base variables
$resource->service->environment_variables()->where('key', "SERVICE_URL_{$serviceName}")->delete();
$resource->service->environment_variables()->where('key', "SERVICE_FQDN_{$serviceName}")->delete();

// Delete port-specific variables
foreach ($serviceInfo['ports'] as $port) {
    $resource->service->environment_variables()->where('key', "SERVICE_URL_{$serviceName}_{$port}")->delete();
    $resource->service->environment_variables()->where('key', "SERVICE_FQDN_{$serviceName}_{$port}")->delete();
}
```

#### 11.3.4 域名值的解析与重组

从用户设置的 fqdn 中解析出协议、主机、端口、路径，然后重组为 URL 和 FQDN：

**关键代码：** [services.php:L327-L343](file:///d:/fz/0601-1/solo-dogfeeding/code/92-coolify/bootstrap/helpers/services.php#L327-L343)

```php
$resourceFqdns = str($resource->fqdn)->explode(',');
$resourceFqdns = $resourceFqdns->first();  // 只使用第一个域名
$url = Url::fromString($resourceFqdns);
$port = $url->getPort();
$path = $url->getPath();

// URL 值（含协议和主机）
$urlValue = $url->getScheme().'://'.$url->getHost();
$urlValue = ($path === '/') ? $urlValue : $urlValue.$path;

// FQDN 值（仅主机，无协议）
$fqdnHost = $url->getHost();
$fqdnValue = str($fqdnHost)->after('://');
if ($path !== '/') {
    $fqdnValue = $fqdnValue.$path;
}
```

**注意：** 当有多个域名时（逗号分隔），只使用第一个域名来生成 `SERVICE_*` 变量的值。

### 11.4 域名记录一致性保证

#### 11.4.1 parse 时的域名同步

在 `applicationParser()` 解析过程中，会检查 `docker_compose_domains` 中是否已存在该服务的域名，如果不存在则添加：

**关键代码：** [parsers.php:L614-L626](file:///d:/fz/0601-1/solo-dogfeeding/code/92-coolify/bootstrap/helpers/parsers.php#L614-L626)

```php
$domains = collect(json_decode(data_get($resource, 'docker_compose_domains'))) ?? collect([]);
$domainExists = data_get($domains->get($serviceName), 'domain');

// Update domain using URL with port if applicable
$domainValue = $port ? $urlWithPort : $url;

if (is_null($domainExists)) {
    $domains->put($serviceName, [
        'domain' => $domainValue,
    ]);
    $resource->docker_compose_domains = $domains->toJson();
    $resource->save();
}
```

**规则：** 只在域名不存在时添加，不覆盖已有的用户自定义域名。

#### 11.4.2 服务删除时的域名清理

当 compose 文件中删除了某个服务时，`docker_compose_domains` 中对应的条目也会被清理：

**关键代码：** [Application.php:L2008-L2035](file:///d:/fz/0601-1/solo-dogfeeding/code/92-coolify/app/Models/Application.php#L2008-L2035)

```php
$json = collect(is_array($decoded) ? $decoded : []);
$normalized = collect();
foreach ($json as $key => $value) {
    // 规范化服务名
}
// 找出已删除的服务并移除
$diff = $normalized->keys()->diff($parsedServices->keys());
$json = $json->filter(function ($value, $key) use ($diff) {
    return ! in_array($key, $diff);
});
```

#### 11.4.3 用户手动修改域名

用户可以通过 UI 手动修改每个服务的域名，修改后：
1. 更新 `docker_compose_domains` 中的对应条目
2. 触发 `updateCompose()` 重新生成 `SERVICE_*` 环境变量

**关键代码：** [PreviewsCompose.php:L38-L43](file:///d:/fz/0601-1/solo-dogfeeding/code/92-coolify/app/Livewire/Project/Application/PreviewsCompose.php#L38-L43)

### 11.5 COOLIFY_URL 与 COOLIFY_FQDN 系统变量

除了 `SERVICE_*` 变量，Coolify 还会为每个服务注入自身的访问地址：

**关键代码：** [parsers.php:L1268-L1277](file:///d:/fz/0601-1/solo-dogfeeding/code/92-coolify/bootstrap/helpers/parsers.php#L1268-L1277)

```php
if (! $isDatabase && $fqdns instanceof Collection && $fqdns->count() > 0) {
    $fqdnsWithoutPort = $fqdns->map(function ($fqdn) {
        return str($fqdn)->after('://')->before(':')->prepend(str($fqdn)->before('://')->append('://'));
    });
    $coolifyEnvironments->put('COOLIFY_URL', $fqdnsWithoutPort->implode(','));

    $urls = $fqdns->map(function ($fqdn) {
        return str($fqdn)->replace('http://', '')->replace('https://', '')->before(':');
    });
    $coolifyEnvironments->put('COOLIFY_FQDN', $urls->implode(','));
}
```

**说明：**
- `COOLIFY_URL`：服务的完整访问 URL（含协议，不含端口）
- `COOLIFY_FQDN`：服务的域名（不含协议和端口）
- 多域名时用逗号分隔
- 数据库服务不注入这些变量

---

## 十二、关键数据流转图示

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
