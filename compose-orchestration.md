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

## 十二、代理层路由标签拼装与服务地址变量

### 12.1 标签生成调度入口

在 `applicationParser()` 第三轮解析中，根据服务器代理类型选择标签生成函数：

**关键代码：** [parsers.php:L1330-L1383](file:///d:/fz/0601-1/solo-dogfeeding/code/92-coolify/bootstrap/helpers/parsers.php#L1330-L1383)

```php
switch ($server->proxyType()) {
    case ProxyTypes::TRAEFIK->value:
        $serviceLabels = $serviceLabels->merge(fqdnLabelsForTraefik(...));
        break;
    case ProxyTypes::CADDY->value:
        $serviceLabels = $serviceLabels->merge(fqdnLabelsForCaddy(...));
        break;
}
```

**参数传递：**
- `uuid`：应用/服务 UUID（PR 部署时带后缀）
- `domains`：`Collection` 逗号分隔域名列表
- `is_force_https_enabled`：是否强制 HTTPS
- `onlyPort`：从 `ports_exposes_array` 提取的首个暴露端口
- `serviceLabels`：从 compose 文件中解析的已有 traefik/caddy 标签
- `is_gzip_enabled` / `is_stripprefix_enabled`：中间件开关
- `service_name`：服务名（用于区分同应用多服务路由）
- `redirect_direction`：www/非 www 重定向方向
- `is_http_basic_auth_enabled`：HTTP Basic Auth 开关

### 12.2 fqdnLabelsForTraefik：多域名多路由规则生成

**关键代码：** [docker.php:L414-L650](file:///d:/fz/0601-1/solo-dogfeeding/code/92-coolify/bootstrap/helpers/docker.php#L414-L650)

#### 12.2.1 全局中间件与 UUID 隔离

**redirect-to-https 全局定义：**

在函数开头，redirect-to-https 中间件被**全局定义一次**，供所有域名的 HTTP 路由共享引用：

**关键代码：** [docker.php:L418-L422](file:///d:/fz/0601-1/solo-dogfeeding/code/92-coolify/bootstrap/helpers/docker.php#L418-L422)

```php
function fqdnLabelsForTraefik(...)
{
    $labels = collect([]);
    $labels->push('traefik.enable=true');
    if ($is_gzip_enabled) {
        $labels->push('traefik.http.middlewares.gzip.compress=true');
    }
    $labels->push('traefik.http.middlewares.redirect-to-https.redirectscheme.scheme=https');
    // ... 其余处理
}
```

**说明：** `redirect-to-https` 和 `gzip` 中间件在所有路由处理之前定义，每个域名的 HTTP 路由只需通过 `middlewares=redirect-to-https` 引用即可，避免重复定义。

**unique_uuid 隔离机制：**

当 `generate_unique_uuid = true` 时，每个域名在循环内部会生成**独立的 Cuid2 UUID**，确保路由规则完全隔离：

**关键代码：** [docker.php:L459-L463](file:///d:/fz/0601-1/solo-dogfeeding/code/92-coolify/bootstrap/helpers/docker.php#L459-L463)

```php
foreach ($domains as $loop => $domain) {
    try {
        if ($generate_unique_uuid) {
            $uuid = new Cuid2;  // 每个域名重新生成 UUID
        }
        // ... 使用新的 UUID 生成路由标签
    }
}
```

**效果：**
- 当 `generate_unique_uuid = false`（默认）：所有域名共享同一个 UUID，路由标签如 `http-0-abc123`, `http-1-abc123`
- 当 `generate_unique_uuid = true`：每个域名使用独立 UUID，路由标签如 `http-0-xyz789`, `http-1-def456`
- 应用场景：需要完全隔离的多域名部署，避免一个域名的配置变更影响其他域名

#### 12.2.2 多域名遍历机制

Traefik 标签生成函数的核心结构是对每个域名**独立遍历**，为每个域名生成完整的路由规则集：

```php
foreach ($domains as $loop => $domain) {
    try {
        $url = Url::fromString($domain);
        $host = $url->getHost();
        $path = $url->getPath();
        $schema = $url->getScheme();
        $port = $url->getPort();
        // 为每个域名独立生成 router 和 service 标签
    } catch (Throwable) {
        continue;  // 单个域名解析失败不影响其他域名
    }
}
```

**关键点：**
- `$loop` 是域名索引（0, 1, 2...），用于标签名的唯一性
- 每个域名独立处理，一个域名解析失败不影响其他域名

#### 12.2.2 路由标签命名规则

**Router 命名公式：**

```
{http|https}-{loop_index}-{uuid}[-{service_name}]
```

| 组件 | 格式 | 示例 |
|-----|------|------|
| HTTP Router | `http-{loop}-{uuid}` | `http-0-abc123` |
| HTTPS Router | `https-{loop}-{uuid}` | `https-0-abc123` |
| 带 service_name | `http-{loop}-{uuid}-{name}` | `http-0-abc123-web` |
| Service (有端口) | `https-{loop}-{uuid}` | `https-0-abc123` |

**多域名场景示例（2 个域名）：**

```
域名1: https://app.example.com   → routers: http-0-abc123, https-0-abc123
域名2: https://www.example.com   → routers: http-1-abc123, https-1-abc123
```

#### 12.2.3 HTTPS 路由标签生成

当域名为 `https` 协议时，生成 HTTPS + HTTP 两种路由：

```php
if ($schema === 'https') {
    // HTTPS 路由
    $labels->push("traefik.http.routers.{$https_label}.rule=Host(`{$host}`) && PathPrefix(`{$path}`)");
    $labels->push("traefik.http.routers.{$https_label}.entryPoints=https");
    
    // 端口绑定（如果指定）
    if ($port) {
        $labels->push("traefik.http.routers.{$https_label}.service={$https_label}");
        $labels->push("traefik.http.services.{$https_label}.loadbalancer.server.port=$port");
    }
    
    // 中间件链（路径剥离、gzip、重定向、认证等）
    if ($path !== '/') {
        $middlewares = collect([]);
        if ($is_stripprefix_enabled) {
            $labels->push("traefik.http.middlewares.{$https_label}-stripprefix.stripprefix.prefixes={$path}");
            $middlewares->push("{$https_label}-stripprefix");
        }
        if ($is_gzip_enabled) $middlewares->push('gzip');
        // ... 其他中间件
        if ($middlewares->isNotEmpty()) {
            $labels->push("traefik.http.routers.{$https_label}.middlewares={$middlewares->join(',')}");
        }
    }
    
    // SSL 证书绑定
    $labels->push("traefik.http.routers.{$https_label}.tls=true");
    $labels->push("traefik.http.routers.{$https_label}.tls.certresolver=letsencrypt");
    
    // HTTP 路由（用于重定向到 HTTPS）
    $labels->push("traefik.http.routers.{$http_label}.rule=Host(`{$host}`) && PathPrefix(`{$path}`)");
    $labels->push("traefik.http.routers.{$http_label}.entryPoints=http");
    if ($is_force_https_enabled) {
        $labels->push("traefik.http.routers.{$http_label}.middlewares=redirect-to-https");
    }
}
```

#### 12.2.4 SSL 证书绑定

每个 HTTPS 路由自动绑定 Let's Encrypt 证书解析器：

```
traefik.http.routers.https-0-{uuid}.tls=true
traefik.http.routers.https-0-{uuid}.tls.certresolver=letsencrypt
```

**多域名证书：** 每个域名独立申请证书，Traefik 通过 router 规则的 `Host()` 匹配自动为每个域名获取对应证书。

#### 12.2.5 中间件链组装（四种场景完整对照表）

`fqdnLabelsForTraefik` 中有四个独立的中间件组装分支，分别对应：
1. HTTPS 协议 + 非根路径（`path !== '/'`）
2. HTTPS 协议 + 根路径（`path === '/'`）
3. HTTP 协议 + 非根路径（`path !== '/'`）
4. HTTPS 场景下的 HTTP 路由（仅 `redirect-to-https`）

**HTTPS + 非根路径（主分支）** — 中间件最多的场景：

| 顺序 | 中间件 | push 条件 | 标签名 | 代码位置 |
|-----|--------|----------|--------|---------|
| 1 | StripPrefix | `$is_stripprefix_enabled` **且** `!str($image)->contains('ghost')` | `{$https_label}-stripprefix` | [docker.php:L510-L513](file:///d:/fz/0601-1/solo-dogfeeding/code/92-coolify/bootstrap/helpers/docker.php#L510-L513) |
| 2 | Gzip | `$is_gzip_enabled` | `gzip` | [docker.php:L514-L516](file:///d:/fz/0601-1/solo-dogfeeding/code/92-coolify/bootstrap/helpers/docker.php#L514-L516) |
| 3 | Ghost redirect | `str($image)->contains('ghost')` | `redir-ghost-{$uuid}` | [docker.php:L517-L519](file:///d:/fz/0601-1/solo-dogfeeding/code/92-coolify/bootstrap/helpers/docker.php#L517-L519) |
| 4 | → non-www | `$redirect_direction === 'non-www'` 且 `str($host)->startsWith('www.')` | `{$loop}-{$uuid}-to-non-www` | [docker.php:L520-L523](file:///d:/fz/0601-1/solo-dogfeeding/code/92-coolify/bootstrap/helpers/docker.php#L520-L523) |
| 5 | → www | `$redirect_direction === 'www'` 且 `!str($host)->startsWith('www.')` | `{$loop}-{$uuid}-to-www` | [docker.php:L524-L527](file:///d:/fz/0601-1/solo-dogfeeding/code/92-coolify/bootstrap/helpers/docker.php#L524-L527) |
| 6 | Basic Auth | `$is_http_basic_auth_enabled` | `http-basic-auth-{$uuid}` | [docker.php:L528-L530](file:///d:/fz/0601-1/solo-dogfeeding/code/92-coolify/bootstrap/helpers/docker.php#L528-L530) |
| 7 | 自定义中间件 | `$middlewares_from_labels` 不为空 | 从 compose labels 提取 | [docker.php:L531-L533](file:///d:/fz/0601-1/solo-dogfeeding/code/92-coolify/bootstrap/helpers/docker.php#L531-L533) |

**HTTPS + 根路径（else 分支）** — 跳过路径剥离但保留其他中间件：

| 顺序 | 中间件 | push 条件 | 与非根路径的区别 |
|-----|--------|----------|----------------|
| 1 | Gzip | `$is_gzip_enabled` | 相同 |
| 2 | Ghost redirect | `str($image)->contains('ghost')` | 相同（ghost 重定向在根路径仍生效） |
| 3 | → non-www | `$redirect_direction === 'non-www'` 且 host 以 www. 开头 | 相同 |
| 4 | → www | `$redirect_direction === 'www'` 且 host 不以 www. 开头 | 相同 |
| 5 | Basic Auth | `$is_http_basic_auth_enabled` | 相同 |
| 6 | 自定义中间件 | `$middlewares_from_labels` 不为空 | 相同 |

**关键区别：** 根路径分支**没有 StripPrefix**（因为没有路径可剥离），但 **ghost redirect 仍然会 push**（因为 ghost 的路径重写逻辑在根路径也需要）。

**关键代码：** [docker.php:L538-L563](file:///d:/fz/0601-1/solo-dogfeeding/code/92-coolify/bootstrap/helpers/docker.php#L538-L563)

**HTTP 协议 + 非根路径** — 纯 HTTP 场景下的中间件：

与 HTTPS + 非根路径的中间件链**完全相同**，只是标签使用 `{$http_label}` 而不是 `{$https_label}`。

**关键代码：** [docker.php:L586-L615](file:///d:/fz/0601-1/solo-dogfeeding/code/92-coolify/bootstrap/helpers/docker.php#L586-L615)

**HTTPS 场景下的 HTTP 路由** — 最轻量的分支：

HTTPS 场景中，HTTP 路由**只有一个中间件**，只负责重定向到 HTTPS：

| 中间件 | push 条件 | 代码位置 |
|--------|----------|---------|
| `redirect-to-https` | `$is_force_https_enabled` | [docker.php:L575-L577](file:///d:/fz/0601-1/solo-dogfeeding/code/92-coolify/bootstrap/helpers/docker.php#L575-L577) |

```php
// HTTPS 场景的 HTTP 路由：只做 redirect-to-https
if ($is_force_https_enabled) {
    $labels->push("traefik.http.routers.{$http_label}.middlewares=redirect-to-https");
}
```

**注意：** 这个 HTTP 路由没有 gzip、没有 www 重定向、没有 Basic Auth、没有自定义中间件 —— 只有强制 HTTPS 重定向。所有业务中间件都在 HTTPS router 上。

**完整分支结构图示：**

```
fqdnLabelsForTraefik
  ├─ 全局定义: gzip, redirect-to-https (函数开头)
  └─ 遍历每个域名
      ├─ HTTPS 协议 (schema === 'https')
      │   ├─ HTTPS router
      │   │   ├─ 非根路径 (path !== '/'): 7 类中间件
      │   │   └─ 根路径 (path === '/'): 6 类中间件 (无 StripPrefix)
      │   ├─ TLS + certresolver (固定)
      │   └─ HTTP router (仅重定向用)
      │       └─ 仅 1 个中间件: redirect-to-https (需 is_force_https_enabled)
      └─ HTTP 协议 (schema !== 'https')
          └─ HTTP router
              ├─ 非根路径: 7 类中间件
              └─ 根路径: 6 类中间件 (无 StripPrefix)
```

**ghost 与 stripprefix 互斥逻辑详解：**

互斥条件出现在所有 `path !== '/'` 的分支中（HTTPS 和 HTTP 各一处）：

```php
// 条件：启用了 stripprefix 且 不是 ghost 镜像
if ($is_stripprefix_enabled && ! str($image)->contains('ghost')) {
    $labels->push("traefik.http.middlewares.{$https_label}-stripprefix.stripprefix.prefixes={$path}");
    $middlewares->push("{$https_label}-stripprefix");
}
```

**四种组合的行为：**

| is_stripprefix_enabled | 镜像含 ghost | StripPrefix 中间件 | Ghost redirect 中间件 | 说明 |
|----------------------|-------------|-------------------|---------------------|------|
| true | 否 | ✅ push | ❌ 不 push | 正常路径剥离场景 |
| true | 是 | ❌ 不 push | ✅ push | ghost 场景，改用正则重写 |
| false | 否 | ❌ 不 push | ❌ 不 push | 保留原始路径 |
| false | 是 | ❌ 不 push | ✅ push | ghost 场景，只做路径重写 |

**设计意图：** Ghost CMS 使用自己的路径系统，如果用 Traefik 的 StripPrefix 会破坏 Ghost 的内部路由。改用 `redir-ghost` 中间件（基于 `redirectregex`）将 `^{path}/(.*)` 重写为 `/$1`，这样 Ghost 能正确感知原始路径。

**ghost redirect 中间件的提前定义：**

`redir-ghost-{$uuid}` 中间件在**域名循环内部、schema 判断之前**就定义了，这样在后面的中间件链中直接引用即可：

**关键代码：** [docker.php:L479-L485](file:///d:/fz/0601-1/solo-dogfeeding/code/92-coolify/bootstrap/helpers/docker.php#L479-L485)

```php
if (str($image)->contains('ghost')) {
    // Traefik 版本
    $labels->push("traefik.http.middlewares.redir-ghost-{$uuid}.redirectregex.regex=^{$path}/(.*)");
    $labels->push("traefik.http.middlewares.redir-ghost-{$uuid}.redirectregex.replacement=/$1");
    // Caddy 版本（同时生成）
    $labels->push("caddy_{$loop}.handle_path.{$loop}_redir-ghost-{$uuid}.handler=rewrite");
    $labels->push("caddy_{$loop}.handle_path.{$loop}_redir-ghost-{$uuid}.rewrite.regexp=^{$path}/(.*)");
    $labels->push("caddy_{$loop}.handle_path.{$loop}_redir-ghost-{$uuid}.rewrite.replacement=/$1");
}
```

**注意：** 这段代码在域名循环的最开头，位于 `try` 块内、schema 判断之前。每个域名都会检查镜像是否含 ghost，如果是则同时生成 Traefik 和 Caddy 两种格式的中间件定义。

#### 12.2.6 用户自定义中间件提取

从 compose 文件的 labels 中提取用户自定义的 Traefik 中间件：

```php
$middlewares_from_labels = $serviceLabels->map(function ($item) {
    if (preg_match('/traefik\.http\.middlewares\.(.*?)(\.|$)/', $item, $matches)) {
        return $matches[1];
    }
    if (preg_match('/coolify\.traefik\.middlewares=(.*)/', $item, $matches)) {
        return explode(',', $matches[1]);
    }
    return null;
})->flatten()->filter()->unique();
```

**两种声明方式：**
1. 标准 Traefik 标签：`traefik.http.middlewares.my-middleware.xxx=yyy`
2. Coolify 快捷方式：`coolify.traefik.middleware=mw1,mw2`

### 12.3 fqdnLabelsForCaddy：多域名多站点生成

**关键代码：** [docker.php:L356-L412](file:///d:/fz/0601-1/solo-dogfeeding/code/92-coolify/bootstrap/helpers/docker.php#L356-L412)

#### 12.3.1 Caddy 标签结构

Caddy 标签结构与 Traefik 不同，使用数组索引式的路径结构：

```
caddy_{loop}={schema}://{host}
caddy_{loop}.header=-Server
caddy_{loop}.try_files={path} /index.html /index.php
caddy_{loop}.{handle}.{loop}_reverse_proxy={{upstreams [$port]}}
caddy_{loop}.{handle}={path}*
```

#### 12.3.2 多域名遍历

```php
foreach ($domains as $loop => $domain) {
    $url = Url::fromString($domain);
    $host = $url->getHost();
    $path = $url->getPath();
    $schema = $url->getScheme();
    $port = $url->getPort();
    
    $handle = 'handle_path';
    if (! $is_stripprefix_enabled) {
        $handle = 'handle';
    }
    
    $labels->push("caddy_{$loop}={$schema}://{$host}");
    $labels->push("caddy_{$loop}.header=-Server");
    $labels->push("caddy_{$loop}.try_files={path} /index.html /index.php");
    
    if ($port) {
        $labels->push("caddy_{$loop}.{$handle}.{$loop}_reverse_proxy={{upstreams $port}}");
    } else {
        $labels->push("caddy_{$loop}.{$handle}.{$loop}_reverse_proxy={{upstreams}}");
    }
    $labels->push("caddy_{$loop}.{$handle}={$path}*");
    
    if ($is_gzip_enabled) {
        $labels->push("caddy_{$loop}.encode=zstd gzip");
    }
    // 重定向和认证...
}
```

#### 12.3.3 Caddy vs Traefik 多域名处理差异

| 维度 | Traefik | Caddy |
|-----|---------|-------|
| 路由标识 | 命名 router（`http-{loop}-{uuid}`） | 数组索引（`caddy_{loop}`） |
| HTTPS 处理 | 显式 tls 标签 + certresolver | Caddy 自动 HTTPS（自动申请证书） |
| HTTP→HTTPS | 中间件 `redirect-to-https` | Caddy 自动重定向 |
| 路径处理 | StripPrefix 中间件 | `handle_path`（自动剥离）/ `handle`（保留路径） |
| 中间件链 | 逗号分隔列表 | 嵌套路径式 |
| 端口指定 | `loadbalancer.server.port` | `{{upstreams $port}}` |
| SSL 证书 | `tls.certresolver=letsencrypt` | Caddy 内置自动证书管理 |

#### 12.3.4 Caddy SSL 证书

Caddy 通过 `caddy_{loop}={schema}://{host}` 标签自动识别域名并申请证书。无需额外配置证书解析器，Caddy 内置了 ACME 证书管理。

### 12.4 服务地址变量与代理标签的协作

#### 12.4.1 代理标签注入到 compose 的时机

标签在 `applicationParser()` 第三轮解析中生成，注入到每个服务的 `labels` 字段：

```php
// parsers.php:L1310-L1320
$serviceLabels = $labels->merge($defaultLabels);
// 根据代理类型添加路由标签
$serviceLabels = $serviceLabels->merge(fqdnLabelsForTraefik/Caddy(...));
// 写入 compose payload
$payload['labels'] = $serviceLabels->toArray();
```

#### 12.4.2 COOLIFY_URL/FQDN 与代理标签的域源一致性

`COOLIFY_URL` / `COOLIFY_FQDN` 的值与代理标签中的域名来自**同一个 fqdn 集合**，确保服务内部获取的访问地址与外部代理路由指向的地址一致。

```php
// parsers.php:L1268-L1277
// 代理标签和 COOLIFY_URL/FQDN 使用相同的 $fqdns 集合
if (! $isDatabase && $fqdns instanceof Collection && $fqdns->count() > 0) {
    $coolifyEnvironments->put('COOLIFY_URL', $fqdnsWithoutPort->implode(','));
    $coolifyEnvironments->put('COOLIFY_FQDN', $urls->implode(','));
}
// 同一个 $fqdns 也传递给 fqdnLabelsForTraefik/Caddy
```

#### 12.4.3 预览部署标签的 UUID 隔离

预览部署时，代理标签中的 UUID 会添加 PR 后缀，确保与主部署的路由规则不冲突：

```php
// parsers.php:L1322-L1329
$labelUuid = $resource->uuid;
$labelNetwork = data_get($resource, 'destination.network');
if ($isPullRequest) {
    $labelUuid = "{$resource->uuid}-{$pullRequestId}";
}
if ($isPullRequest) {
    $labelNetwork = "{$resource->destination->network}-{$pullRequestId}";
}
```

---

## 十三、isDeployable 部署守卫机制

### 13.1 两种 isDeployable 的语义差异

Coolify 中存在两个 `isDeployable`，分别用于不同的资源类型和场景：

| 资源类型 | 方法 | 语义 | 代码位置 |
|---------|------|------|---------|
| **Application** | `isDeployable(): bool` | 检查自动部署开关（`is_auto_deploy_enabled`） | [Application.php:L1106-L1113](file:///d:/fz/0601-1/solo-dogfeeding/code/92-coolify/app/Models/Application.php#L1106-L1113) |
| **Service** | `isDeployable(): Attribute` | 检查必填环境变量是否已填写 | [Service.php:L1622-L1636](file:///d:/fz/0601-1/solo-dogfeeding/code/92-coolify/app/Models/Service.php#L1622-L1636) |

### 13.2 Application::isDeployable()

**语义：** 该应用是否允许通过 Webhook 自动触发部署

```php
// Application.php:L1106-L1113
public function isDeployable(): bool
{
    if ($this->settings->is_auto_deploy_enabled) {
        return true;
    }
    return false;
}
```

**使用场景：** Webhook 控制器中判断是否响应该 push 事件

```php
// Github.php:L133
if ($application->isDeployable()) {
    // 入队 ApplicationDeploymentJob
}
```

**影响范围：**
- `true` → Webhook push 事件会触发自动部署
- `false` → Webhook push 事件被忽略，但手动部署不受影响

**入队逻辑链路（以 GitHub Webhook 为例）：**

```
GitHub Push Event
    ↓
Github.php:133  if ($application->isDeployable())
    ↓ true
$application->parse();  // 解析 compose
    ↓
ApplicationDeploymentQueue::create([...]);  // 创建部署队列记录
    ↓
ApplicationDeploymentJob::dispatch(...);  // 入队实际部署 Job
```

### 13.3 Service::isDeployable()

**语义：** 该 Service 的所有必填环境变量是否已填写

```php
// Service.php:L1622-L1636
protected function isDeployable(): Attribute
{
    return Attribute::make(
        get: function () {
            $envs = $this->environment_variables()->where('is_required', true)->get();
            foreach ($envs as $env) {
                if ($env->is_really_required) {
                    return false;
                }
            }
            return true;
        }
    );
}
```

**判断逻辑：**

```
遍历所有 is_required=true 的环境变量
    ↓
对每个变量调用 is_really_required
    ↓
is_really_required = is_required && real_value 为空
    ↓
任一变量 is_really_required = true → isDeployable = false
```

### 13.4 is_really_required 计算逻辑

**关键代码：** [EnvironmentVariable.php:L212-L217](file:///d:/fz/0601-1/solo-dogfeeding/code/92-coolify/app/Models/EnvironmentVariable.php#L212-L217)

```php
protected function isReallyRequired(): Attribute
{
    return Attribute::make(
        get: fn () => $this->is_required && str($this->real_value)->isEmpty(),
    );
}
```

**判定规则：**
- `is_required = false` → 不是必填，`is_really_required = false`
- `is_required = true` 且 `real_value` 非空 → 已填写，`is_really_required = false`
- `is_required = true` 且 `real_value` 为空 → 未填写，`is_really_required = true`

**`real_value` 的计算：** [EnvironmentVariable.php:L195-L210](file:///d:/fz/0601-1/solo-dogfeeding/code/92-coolify/app/Models/EnvironmentVariable.php#L195-L210)

`real_value` 会按优先级解析：值本身 → 共享变量查找 → 构建包控制变量。

### 13.5 UI 部署按钮禁用状态

#### 13.5.1 Service 部署按钮

**关键代码：** [heading.blade.php:L32-L212](file:///d:/fz/0601-1/solo-dogfeeding/code/92-coolify/resources/views/livewire/project/service/heading.blade.php#L32-L212)

**isDeployable = true 时：** 显示正常的部署操作按钮

```blade
@if ($service->isDeployable)
    <!-- 显示 Deploy / Restart / Stop 等操作按钮 -->
    <div class="order-first flex flex-wrap items-center gap-2 sm:order-last">
        <!-- 根据服务状态显示不同操作 -->
    </div>
```

**isDeployable = false 时：** 替换为错误提示

```blade
@else
    <div class="flex flex-wrap order-first gap-2 items-center sm:order-last">
        <div class="text-error">
            Unable to deploy. <a class="underline font-bold cursor-pointer"
                href="{{ route('project.service.environment-variables', $parameters) }}">
                Required environment variables missing.</a>
        </div>
    </div>
@endif
```

**UI 效果：**
- 部署/重启/停止按钮**完全隐藏**
- 显示红色错误信息："Unable to deploy. Required environment variables missing."
- 提供指向环境变量配置页面的链接

#### 13.5.2 环境变量输入框的状态指示

在环境变量编辑界面中，必填且未填写的变量会显示视觉警告（红色边框 `border-error`），帮助用户快速定位缺失的变量。

### 13.6 部署守卫的完整链路

```
compose 解析
    ↓
环境变量创建（is_required 标记）
    ↓
用户填写 / 未填写变量
    ↓
is_really_required 计算
    ↓                    ↓
is_really_required=false  is_really_required=true
    ↓                      ↓
isDeployable=true         isDeployable=false
    ↓                      ↓
UI: 显示部署按钮          UI: 显示错误提示 + 隐藏按钮
    ↓                      ↓
手动部署 → 入队 Job       手动部署 → 不可用
Webhook → isDeployable()  Webhook → 请求被忽略
    ↓
ApplicationDeploymentJob 执行
```

### 13.7 Application::isDeployable() 与 Service::isDeployable() 的协作

对于使用 docker-compose 构建包的 Application：

- `Application::isDeployable()` 只控制 **Webhook 自动部署**，手动部署不受其影响
- Service 类型的 `isDeployable()` 控制 **UI 部署按钮的显示**和**手动部署的可用性**
- Application 类型没有 Service 那样的必填变量检查部署守卫，其 `isDeployable()` 语义完全不同

---

## 十四、watch_paths 闸口与 deployment_queue helper

### 14.1 Webhook 三层闸口

Webhook 触发部署前需要经过三层闸口检查：

```
Git Push Webhook 到达
       ↓
[闸口 1] Application::isDeployable()
       ↓ true
[闸口 2] isWatchPathsTriggered() 或 watch_paths 为空
       ↓ true
[闸口 3] 检查 commit 是否包含 [skip cd] / [skip ci]
       ↓ 通过
queue_application_deployment() 入队
```

**关键代码：** [Github.php:L133-L135](file:///d:/fz/0601-1/solo-dogfeeding/code/92-coolify/app/Http/Controllers/Webhook/Github.php#L133-L135)

```php
if ($application->isDeployable()) {
    $is_watch_path_triggered = $application->isWatchPathsTriggered($changed_files);
    if ($is_watch_path_triggered || blank($application->watch_paths)) {
        // ... 入队部署
    }
}
```

### 14.2 watch_paths 匹配算法

`matchPaths()` 实现了支持否定模式的路径匹配，采用**顺序匹配、最后匹配优先**的策略。

**关键代码：** [Application.php:L2142-L2177](file:///d:/fz/0601-1/solo-dogfeeding/code/92-coolify/app/Models/Application.php#L2142-L2177)

```php
public static function matchPaths(Collection $modified_files, ?Collection $watch_paths): Collection
{
    if (is_null($watch_paths) || $watch_paths->isEmpty()) {
        return collect([]);
    }

    return $modified_files->filter(function ($file) use ($watch_paths) {
        $shouldInclude = null; // null 表示还没有任何模式匹配

        foreach ($watch_paths as $pattern) {
            $pattern = trim($pattern);
            if (empty($pattern)) continue;

            $isExclusion = str_starts_with($pattern, '!');
            $matchPattern = $isExclusion ? substr($pattern, 1) : $pattern;

            if (self::globMatch($matchPattern, $file)) {
                $shouldInclude = ! $isExclusion;
            }
        }

        // 所有模式都是排除模式且无匹配时，默认包含
        if ($shouldInclude === null) {
            $hasInclusionPatterns = $watch_paths->contains(
                fn ($p) => ! str_starts_with(trim($p), '!')
            );
            return ! $hasInclusionPatterns;
        }

        return $shouldInclude;
    })->values();
}
```

**匹配规则表：**

| 模式类型 | 示例 | 说明 |
|---------|------|------|
| 普通模式 | `*.yml`, `src/**/*.php` | 匹配指定文件 |
| 否定模式 | `!*.md`, `!docs/**` | 排除指定文件 |
| 仅否定模式 | `!*.md`, `!*.txt` | 无包含模式时，默认包含所有未排除的文件 |
| 混合模式 | `src/**`, `!*.test.js` | 最后匹配的模式决定结果 |

**匹配优先级：**
- 模式按顺序处理，**后定义的模式优先级更高**
- 同一文件被多个模式匹配时，以最后一个匹配的模式为准

### 14.3 globToRegex 通配符转正则表达式

**关键代码：** [Application.php:L2193-L2270](file:///d:/fz/0601-1/solo-dogfeeding/code/92-coolify/app/Models/Application.php#L2193-L2270)

支持的通配符：

| 通配符 | 含义 | 正则 |
|-------|------|------|
| `*` | 匹配任意字符（不包含 `/`） | `[^/]*` |
| `**` | 匹配任意数量目录 | `.*` |
| `**/` | 匹配开头的任意目录 | `.*`（自动跳过后面的 `/`） |
| `?` | 匹配单个字符（不包含 `/`） | `[^/]` |
| `[abc]` | 字符类 | `[abc]` |
| `[!abc]` / `[^abc]` | 否定字符类 | `[^abc]` |

### 14.4 isWatchPathsTriggered 判定

**关键代码：** [Application.php:L2282-L2298](file:///d:/fz/0601-1/solo-dogfeeding/code/92-coolify/app/Models/Application.php#L2282-L2298)

```php
public function isWatchPathsTriggered(Collection $modified_files): bool
{
    if (is_null($this->watch_paths)) {
        return false;
    }

    $this->normalizeWatchPaths();  // 规范化，移除空行和前后空格

    $watch_paths = collect(explode("\n", $this->watch_paths));

    if ($watch_paths->isEmpty()) {
        return false;
    }

    $matches = $this->matchWatchPaths($modified_files, $watch_paths);

    return $matches->count() > 0;
}
```

### 14.5 watch_paths 存储格式

watch_paths 以换行分隔的字符串存储，支持多行输入：

**UI 输入示例：**
```
*.yml
src/**/*.php
!*.test.js
!docs/**
```

**normalizeWatchPaths 规范化处理：**
- 每行去除前后空格
- `!` 否定模式保留 `!`，去除后面的空格和前导 `/`
- 移除空行
- 保存为 JSON 字符串

**关键代码：** [Application.php:L2096-L2116](file:///d:/fz/0601-1/solo-dogfeeding/code/92-coolify/app/Models/Application.php#L2096-L2116)

### 14.6 deployment_queue() 队列路由 helper

**关键代码：** [shared.php:L612-L625](file:///d:/fz/0601-1/solo-dogfeeding/code/92-coolify/bootstrap/helpers/shared.php#L612-L625)

```php
function deployment_queue(): string
{
    return isCloud() ? 'deployments' : 'high';
}
```

**队列路由规则：**
- **Cloud 环境**：使用 `deployments` 专用队列，由独立的 Horizon worker 池处理
- **自托管环境**：使用 `high` 共享队列
- **配置依据**：`config('constants.coolify.self_hosted')`，而非 `HORIZON_QUEUES` 环境变量
- **注意**：Cloud 环境的 worker 必须在 `HORIZON_QUEUES` 中包含 `deployments`，否则 Job 永远不会被处理

### 14.7 queue_application_deployment() 入队流程

**关键代码：** [applications.php:L15-L109](file:///d:/fz/0601-1/solo-dogfeeding/code/92-coolify/bootstrap/helpers/applications.php#L15-L109)

完整的入队流程包含四道检查：

```
queue_application_deployment() 调用
       ↓
[检查 1] 服务器部署队列是否已满
       ↓ 未满（默认限制 25）
[检查 2] 是否已有相同 commit + PR 的部署在队列中
       ↓ 无重复 或 force_rebuild=true
创建 ApplicationDeploymentQueue 记录
       ↓
[检查 3] 是否 no_questions_asked（立即执行）
       ↓ 否
[检查 4] next_queuable() 是否允许启动
       ↓ 允许
更新状态为 IN_PROGRESS
       ↓
ApplicationDeploymentJob::dispatch()
       ↓
返回到调用方
```

#### 14.7.1 队列满检查

```php
$queue_limit = $serverForQueueCheck->settings->deployment_queue_limit ?? 25;
$queued_count = ApplicationDeploymentQueue::where('server_id', $server_id)
    ->where('status', ApplicationDeploymentStatus::QUEUED->value)
    ->count();

if ($queued_count >= $queue_limit) {
    return [
        'status' => 'queue_full',
        'message' => 'Deployment queue is full. Please wait for existing deployments to complete.',
    ];
}
```

#### 14.7.2 重复部署检查

```php
$existing_deployment = ApplicationDeploymentQueue::where('application_id', $application_id)
    ->where('commit', $commit)
    ->where('pull_request_id', $pull_request_id)
    ->whereIn('status', [IN_PROGRESS, QUEUED])
    ->first();

if ($existing_deployment && ! $force_rebuild && ! $rollback && ! $no_questions_asked) {
    return [
        'status' => 'skipped',
        'message' => 'Deployment already queued for this commit.',
        'deployment_uuid' => $existing_deployment->deployment_uuid,
    ];
}
```

#### 14.7.3 next_queuable 并发控制

**关键代码：** [applications.php:L142-L164](file:///d:/fz/0601-1/solo-dogfeeding/code/92-coolify/bootstrap/helpers/applications.php#L142-L164)

```php
function next_queuable(string $server_id, string $application_id, string $commit = 'HEAD', int $pull_request_id = 0): bool
{
    // 同一应用 + 同一 PR 不允许并发部署
    $in_progress = ApplicationDeploymentQueue::where('application_id', $application_id)
        ->where('pull_request_id', $pull_request_id)
        ->where('status', ApplicationDeploymentStatus::IN_PROGRESS->value)
        ->exists();

    if ($in_progress) {
        return false;
    }

    // 服务器并发构建限制
    $server = Server::find($server_id);
    $concurrent_builds = $server->settings->concurrent_builds;
    $active_deployments = ApplicationDeploymentQueue::where('server_id', $server_id)
        ->where('status', ApplicationDeploymentStatus::IN_PROGRESS->value)
        ->count();

    if ($active_deployments >= $concurrent_builds) {
        return false;
    }

    return true;
}
```

**并发规则：**
- 同一应用的普通部署（`pull_request_id = 0`）之间不允许并发
- 同一应用的 PR 部署（`pull_request_id > 0`）之间不允许并发
- 普通部署和 PR 部署**可以**并发运行（因为 `pull_request_id` 不同）
- 受服务器的 `concurrent_builds` 全局限制

#### 14.7.4 Deployment Job 分发

当检查通过时，Job 通过 `deployment_queue()` helper 路由到正确的队列：

```php
ApplicationDeploymentJob::dispatch(
    application_deployment_queue_id: $deployment->id,
)->onQueue(deployment_queue());
```

### 14.8 Webhook 到 Job 的完整链路

```
GitHub/Gitea/GitLab Push Event
       ↓
Webhook 控制器解析 changed_files
       ↓
┌─────────────────────────────────────────┐
│ 闸口 1: Application::isDeployable()     │ 检查自动部署开关
└─────────────────────────────────────────┘
       ↓ true
┌─────────────────────────────────────────┐
│ 闸口 2: isWatchPathsTriggered()         │ 匹配变更文件
│   - matchPaths() 支持否定模式           │
│   - 最后匹配优先                        │
│ 或 watch_paths 为空（全部触发）         │
└─────────────────────────────────────────┘
       ↓ true
┌─────────────────────────────────────────┐
│ 闸口 3: 检查 [skip cd]/[skip ci]        │ 检查 commit 消息
└─────────────────────────────────────────┘
       ↓ 通过
生成 deployment_uuid (Cuid2)
       ↓
queue_application_deployment()
       ↓
┌─────────────────────────────────────────┐
│ 检查 1: 队列是否已满（默认 25）         │
└─────────────────────────────────────────┘
       ↓
┌─────────────────────────────────────────┐
│ 检查 2: 是否重复部署                    │
└─────────────────────────────────────────┘
       ↓
创建 ApplicationDeploymentQueue 记录
       ↓
┌─────────────────────────────────────────┐
│ 检查 3: next_queuable()                 │
│   - 同一应用+PR 是否有进行中部署         │
│   - 服务器并发限制                       │
└─────────────────────────────────────────┘
       ↓ 允许
更新状态为 IN_PROGRESS
       ↓
ApplicationDeploymentJob::dispatch()
       ↓
onQueue(deployment_queue())
       ↓
Worker 执行 Job
```

---

## 十五、关键数据流转图示

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
