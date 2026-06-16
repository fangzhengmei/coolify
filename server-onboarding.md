# 服务器接入流程代码分析

## 概述

服务器接入流程由 [ValidateAndInstallServerJob](file:///d:/fz/0601-1/solo-dogfeeding/code/93-coolify/app/Jobs/ValidateAndInstallServerJob.php) 驱动，包含三大核心阶段：

1. **远程登录连通校验** - 验证 SSH 连接及系统基本信息
2. **容器运行时安装作业** - 前置依赖安装、Docker 引擎安装
3. **状态持久化** - 校验结果与服务器状态字段的对应关系

---

## 一、按代码顺序的完整流程

### 总入口
[ValidateAndInstallServerJob::handle()](file:///d:/fz/0601-1/solo-dogfeeding/code/93-coolify/app/Jobs/ValidateAndInstallServerJob.php#L33-L205)

```
┌─────────────────────────────────────────────────────────────┐
│ 0. 标记验证开始                                             │
│    $server->update(['is_validating' => true])               │
└───────────────────────────┬─────────────────────────────────┘
                            │
┌───────────────────────────▼─────────────────────────────────┐
│ 1. 远程登录连通校验                                         │
│    validateConnection() → validateOS()                      │
└───────────────────────────┬─────────────────────────────────┘
                            │
┌───────────────────────────▼─────────────────────────────────┐
│ 2. 前置依赖检查与安装                                       │
│    validatePrerequisites() → installPrerequisites()         │
│    (失败则重试，最多3次)                                    │
└───────────────────────────┬─────────────────────────────────┘
                            │
┌───────────────────────────▼─────────────────────────────────┐
│ 3. Docker 检查与安装                                        │
│    validateDockerEngine() + validateDockerCompose()         │
│    → installDocker()                                        │
│    (失败则重试，最多3次)                                    │
└───────────────────────────┬─────────────────────────────────┘
                            │
┌───────────────────────────▼─────────────────────────────────┐
│ 4. Docker 版本验证                                          │
│    validateDockerEngineVersion()                            │
└───────────────────────────┬─────────────────────────────────┘
                            │
┌───────────────────────────▼─────────────────────────────────┐
│ 5. 接入完成                                                 │
│    启动代理 → 收集元数据 → 广播事件                          │
└─────────────────────────────────────────────────────────────┘
```

---

## 二、远程登录连通校验（阶段1）

### 2.1 SSH 连通性校验

**代码位置**: [Server::validateConnection()](file:///d:/fz/0601-1/solo-dogfeeding/code/93-coolify/app/Models/Server.php#L1270-L1298)

```php
public function validateConnection(bool $justCheckingNewKey = false)
{
    $this->disableSshMux();

    if ($this->skipServer()) {
        return ['uptime' => false, 'error' => 'Server skipped.'];
    }
    try {
        instant_remote_process(['ls /'], $this);
        if ($this->settings->is_reachable === false) {
            $this->settings->is_reachable = true;
            $this->settings->save();
            ServerReachabilityChanged::dispatch($this);
        }
        return ['uptime' => true, 'error' => null];
    } catch (\Throwable $e) {
        if ($justCheckingNewKey) {
            return ['uptime' => false, 'error' => 'This key is not valid for this server.'];
        }
        if ($this->settings->is_reachable === true) {
            $this->settings->is_reachable = false;
            $this->settings->save();
            ServerReachabilityChanged::dispatch($this);
        }
        return ['uptime' => false, 'error' => $e->getMessage()];
    }
}
```

#### 2.1.1 完整分支与状态字段行为

方法有 4 条退出路径，每条路径对状态字段的处理不同：

**路径 A：skipServer 命中**（[L1274-L1276](file:///d:/fz/0601-1/solo-dogfeeding/code/93-coolify/app/Models/Server.php#L1274-L1276)）

```php
if ($this->skipServer()) {
    return ['uptime' => false, 'error' => 'Server skipped.'];
}
```

触发条件：`ip === '1.2.3.4'` 或 `settings.force_disabled === true`（见 [skipServer()](file:///d:/fz/0601-1/solo-dogfeeding/code/93-coolify/app/Models/Server.php#L1071-L1081)）

| 字段 | 行为 |
|-----|------|
| `settings.is_reachable` | **不修改**，保持原值 |
| `settings.save()` | **不调用** |
| `ServerReachabilityChanged` | **不广播** |

**路径 B：连接成功**（[L1278-L1285](file:///d:/fz/0601-1/solo-dogfeeding/code/93-coolify/app/Models/Server.php#L1278-L1285)）

```php
if ($this->settings->is_reachable === false) {  // 仅在值从 false→true 变化时
    $this->settings->is_reachable = true;
    $this->settings->save();
    ServerReachabilityChanged::dispatch($this);
}
```

| 条件 | `settings.is_reachable` | `save()` | 广播 |
|-----|------------------------|----------|------|
| 之前 `is_reachable === false` | 改为 `true` | 调用 | 触发 |
| 之前 `is_reachable === true` | **不修改** | **不调用** | **不触发** |

**关键**：`save()` 和广播**不是无条件执行**的，只在 `is_reachable` 值真正从 `false` 变为 `true` 时才触发。如果服务器本来就可达，连接成功后不做任何写操作。

**路径 C：连接失败 + justCheckingNewKey**（[L1287-L1289](file:///d:/fz/0601-1/solo-dogfeeding/code/93-coolify/app/Models/Server.php#L1287-L1289)）

```php
if ($justCheckingNewKey) {
    return ['uptime' => false, 'error' => 'This key is not valid for this server.'];
}
```

触发条件：调用方传入 `justCheckingNewKey = true`，用于验证新 SSH 密钥是否可用

| 字段 | 行为 |
|-----|------|
| `settings.is_reachable` | **不修改**，保持原值 |
| `settings.save()` | **不调用** |
| `ServerReachabilityChanged` | **不广播** |

这个分支的设计意图：检查新密钥时如果连接失败，不影响服务器已有的可达性状态。错误信息是固定的 `"This key is not valid for this server."`，而非原始异常消息。

**路径 D：连接失败（正常路径）**（[L1290-L1296](file:///d:/fz/0601-1/solo-dogfeeding/code/93-coolify/app/Models/Server.php#L1290-L1296)）

```php
if ($this->settings->is_reachable === true) {  // 仅在值从 true→false 变化时
    $this->settings->is_reachable = false;
    $this->settings->save();
    ServerReachabilityChanged::dispatch($this);
}
```

| 条件 | `settings.is_reachable` | `save()` | 广播 |
|-----|------------------------|----------|------|
| 之前 `is_reachable === true` | 改为 `false` | 调用 | 触发 |
| 之前 `is_reachable === false` | **不修改** | **不调用** | **不触发** |

**关键**：与成功路径对称，`save()` 和广播只在 `is_reachable` 值真正从 `true` 变为 `false` 时才触发。如果服务器本来就不可达，再次连接失败不会重复写库或广播。

#### 2.1.2 四条路径汇总

| 路径 | 触发条件 | `is_reachable` | `save()` | 广播 | 返回 error |
|-----|---------|---------------|----------|------|-----------|
| A: skipServer | ip=1.2.3.4 或 force_disabled | 不修改 | 不调用 | 不触发 | `'Server skipped.'` |
| B: 连接成功 | SSH 可达 | 仅 false→true 时改 | 仅值变化时调用 | 仅值变化时触发 | `null` |
| C: justCheckingNewKey | 新密钥验证失败 | 不修改 | 不调用 | 不触发 | `'This key is not valid for this server.'` |
| D: 连接失败 | SSH 不可达 | 仅 true→false 时改 | 仅值变化时调用 | 仅值变化时触发 | 原始异常消息 |

### 2.2 操作系统类型校验

**代码位置**: [Server::validateOS()](file:///d:/fz/0601-1/solo-dogfeeding/code/93-coolify/app/Models/Server.php#L1099-L1121)

```php
public function validateOS(): bool|Stringable
{
    $os_release = instant_remote_process(['cat /etc/os-release'], $this);
    // 解析 ID 字段，与 SUPPORTED_OS 常量对比
    // 支持: debian / rhel / sles / arch 等
}
```

**返回值**:
- 成功: 返回 OS 类型字符串（如 `"debian"`）
- 失败: 返回 `false`

### 2.3 阶段1失败时的状态持久化

在 [ValidateAndInstallServerJob](file:///d:/fz/0601-1/solo-dogfeeding/code/93-coolify/app/Jobs/ValidateAndInstallServerJob.php#L46-L75) 中：

| 失败点 | `validation_logs` | `is_validating` | `settings.is_reachable` |
|-------|-------------------|-----------------|------------------------|
| 连接失败 (L47-60) | 错误信息（含文档链接） | `false` | 已由 validateConnection 更新 |
| OS 不支持 (L63-75) | 错误信息（提示手动安装 Docker） | `false` | 保持之前的值 |

---

## 三、容器运行时安装作业（阶段2）

### 3.1 前置依赖校验

**代码位置**: [ValidatePrerequisites](file:///d:/fz/0601-1/solo-dogfeeding/code/93-coolify/app/Actions/Server/ValidatePrerequisites.php)

检查命令: `git`, `curl`, `jq`

```php
return [
    'success' => empty($missing),  // 是否全部存在
    'missing' => $missing,         // 缺失的命令列表
    'found' => $found,             // 已存在的命令列表
];
```

### 3.2 前置依赖安装

**代码位置**: [InstallPrerequisites](file:///d:/fz/0601-1/solo-dogfeeding/code/93-coolify/app/Actions/Server/InstallPrerequisites.php)

根据 OS 类型使用不同包管理器安装缺失命令。

### 3.3 Docker 引擎校验

**代码位置**: [Server::validateDockerEngine()](file:///d:/fz/0601-1/solo-dogfeeding/code/93-coolify/app/Models/Server.php#L1320-L1348)

```php
public function validateDockerEngine($throwError = false)
{
    // 1. 检查 docker 二进制是否存在
    $dockerBinary = instant_remote_process(['command -v docker'], ...);
    if (is_null($dockerBinary)) {
        $this->settings->is_usable = false;  // 标记不可用
        return false;
    }
    // 2. 检查 Docker 是否正常运行
    try {
        instant_remote_process(['docker version'], $this);
    } catch (\Throwable $e) {
        $this->settings->is_usable = false;  // 标记不可用
        return false;
    }
    // 3. 成功: 标记可用，创建网络
    $this->settings->is_usable = true;
    $this->validateCoolifyNetwork(...);
    return true;
}
```

**校验结果 → 状态字段** 对应关系：

| 校验结果 | `settings.is_usable` | 副作用 |
|---------|---------------------|--------|
| Docker 二进制不存在 | `false` | - |
| Docker 未运行 | `false` | - |
| Docker 正常运行 | `true` | 创建 `coolify` 网络 |

### 3.4 Docker Compose 校验

**代码位置**: [Server::validateDockerCompose()](file:///d:/fz/0601-1/solo-dogfeeding/code/93-coolify/app/Models/Server.php#L1350-L1366)

执行 `docker compose version` 检查。

### 3.5 Docker 安装

**代码位置**: [InstallDocker](file:///d:/fz/0601-1/solo-dogfeeding/code/93-coolify/app/Actions/Server/InstallDocker.php)

主要步骤：
1. 验证 OS 类型是否支持
2. 生成 CA 证书（用于镜像仓库 TLS）
3. 根据 OS 类型执行对应安装命令
4. 配置 Docker daemon.json（日志驱动配置）
5. 重启 Docker 服务
6. 创建 `coolify` 或 `coolify-overlay` 网络

### 3.6 阶段2失败时的状态持久化

在 [ValidateAndInstallServerJob](file:///d:/fz/0601-1/solo-dogfeeding/code/93-coolify/app/Jobs/ValidateAndInstallServerJob.php#L78-L145) 中：

| 失败点 | `validation_logs` | `is_validating` | `settings.is_usable` |
|-------|-------------------|-----------------|---------------------|
| 前置依赖失败 (L78-111) | 3次重试后才设置错误 | 仅3次重试后才设为 `false` | 未修改 |
| Docker 缺失 (L117-145) | 3次重试后才设置错误 | 仅3次重试后才设为 `false` | 已由 validateDockerEngine 更新 |

---

## 四、安装成功后重试的边界判定

这是整个流程最需要注意的部分。

### 4.1 重试参数

```php
// ValidateAndInstallServerJob 类属性
public int $maxTries = 3;
public int $numberOfTries = 0;  // 构造函数传入
```

### 4.2 重试触发条件

**前置依赖安装重试** (L80):
```php
if (! $validationResult['success']) {
    if ($this->numberOfTries >= $this->maxTries) {
        // 终止：已达最大重试次数
    } else {
        // 安装 + 延迟30秒后重试
        $this->server->installPrerequisites();
        self::dispatch($this->server, $this->numberOfTries + 1)
            ->delay(now()->addSeconds(30));
        return;  // 提前退出，当前 Job 结束
    }
}
```

**Docker 安装重试** (L119):
```php
if (! $dockerInstalled || ! $dockerComposeInstalled) {
    if ($this->numberOfTries >= $this->maxTries) {
        // 终止：已达最大重试次数
    } else {
        // 安装 + 延迟30秒后重试
        $this->server->installDocker();
        self::dispatch($this->server, $this->numberOfTries + 1)
            ->delay(now()->addSeconds(30));
        return;  // 提前退出，当前 Job 结束
    }
}
```

### 4.3 重试边界的关键细节

| 问题 | 答案 |
|-----|------|
| **numberOfTries 从哪来？** | 构造函数传入，首次调用为 `numberOfTries = 0` |
| **最大安装机会有几次？** | 3 次。`numberOfTries` 取值为 `0`、`1`、`2` 时各执行 1 次安装 |
| **共有几次 Job 执行？** | 4 次。`numberOfTries` 取值为 `0`、`1`、`2`、`3`，共 4 次 Job 实例 |
| **终止条件是什么？** | `numberOfTries >= maxTries`（即 `numberOfTries = 3` 时），第 4 次 Job 不再安装，直接终止 |
| **安装成功后还会重试吗？** | 不会。安装后重新 dispatch 新 Job，新 Job 从第一步重新开始校验。如果前置依赖/Docker 已安装，校验通过，流程继续向下。 |
| **重试时会重新校验连接吗？** | 会。每次新 Job 都会从 `validateConnection()` 开始，完整重新执行所有校验步骤。 |
| **安装失败会怎样？** | `installPrerequisites()` 和 `installDocker()` 内部通过 `remote_process()` 执行命令。如果命令执行失败会抛出异常，被 Job 的 `catch` 捕获，`is_validating` 设为 `false`，流程终止。 |
| **30秒延迟的作用？** | 给安装命令留出执行时间，避免立即重试时安装还未完成。 |

### 4.4 按 numberOfTries 取值的详细推导

**核心判定代码**（前置依赖和 Docker 共用同一逻辑）：

```php
if (! $validationResult['success']) {
    if ($this->numberOfTries >= $this->maxTries) {
        // 终止：写入错误日志，is_validating = false，return
    } else {
        // 执行安装
        $this->server->installPrerequisites();
        // dispatch 新 Job，numberOfTries + 1，延迟 30 秒
        self::dispatch($this->server, $this->numberOfTries + 1)
            ->delay(now()->addSeconds(30));
        return;  // 当前 Job 结束
    }
}
```

`maxTries = 3`，逐个取值推导：

| 场景 | numberOfTries | 判定条件 | 动作 | 安装次数 | `validation_logs` | `is_validating` |
|-----|--------------|---------|------|---------|-------------------|-----------------|
| 首次检查（校验1） | 0 | `0 >= 3` → false | 执行安装 → dispatch(1) | 第 1 次 | 不设置 | 保持 `true` |
| 重试一（校验2） | 1 | `1 >= 3` → false | 执行安装 → dispatch(2) | 第 2 次 | 不设置 | 保持 `true` |
| 重试二（校验3） | 2 | `2 >= 3` → false | 执行安装 → dispatch(3) | 第 3 次 | 不设置 | 保持 `true` |
| 重试三（校验4） | 3 | `3 >= 3` → true | 终止流程 | — | 写入错误信息 | 设为 `false` |

**结论**：
- **3 次安装机会**：`numberOfTries = 0/1/2` 时各执行 1 次
- **4 次校验机会**：`numberOfTries = 0/1/2/3` 各有 1 次校验
- **第 4 次 Job 终止**：`numberOfTries = 3` 时触发终止条件，不再安装

### 4.5 重试流程时序图（前置依赖始终安装失败场景）

```
外部触发 → 首次调用: numberOfTries=0
    ↓
validateConnection() ✓
validateOS() ✓
validatePrerequisites() ✗ (missing: git)
    ↓
0 < 3 → 执行第1次安装
    ↓
dispatch 新 Job, numberOfTries=1, delay 30s
    ↓
当前 Job return → 结束 (is_validating 保持 true)
    ↓
30秒后 → 新 Job 启动: numberOfTries=1
    ↓
validateConnection() ✓
validateOS() ✓
validatePrerequisites() ✗ (git 仍缺失)
    ↓
1 < 3 → 执行第2次安装
    ↓
dispatch 新 Job, numberOfTries=2, delay 30s
    ↓
当前 Job return → 结束 (is_validating 保持 true)
    ↓
30秒后 → 新 Job 启动: numberOfTries=2
    ↓
validateConnection() ✓
validateOS() ✓
validatePrerequisites() ✗ (git 仍缺失)
    ↓
2 < 3 → 执行第3次安装
    ↓
dispatch 新 Job, numberOfTries=3, delay 30s
    ↓
当前 Job return → 结束 (is_validating 保持 true)
    ↓
30秒后 → 新 Job 启动: numberOfTries=3
    ↓
validateConnection() ✓
validateOS() ✓
validatePrerequisites() ✗ (git 仍缺失)
    ↓
3 >= 3 → 终止流程
    ↓
写入 validation_logs + is_validating = false
    ↓
return → 彻底结束
```

### 4.6 重试中状态字段的关键行为

**重试过程中（未达到 maxTries）**：
- `is_validating` 始终保持 `true`，因为没有代码修改它
- `validation_logs` 不会被写入，避免覆盖之前的错误信息
- 每次安装后 dispatch 新 Job，`numberOfTries` 递增 1
- 新 Job 延迟 30 秒启动，给安装命令留出执行时间

**达到 maxTries 时**：
- `validation_logs` 写入最终错误信息（包含重试次数）
- `is_validating` 设为 `false`
- `return` 终止整个接入流程

**注意**：`numberOfTries` 是 Job 实例的属性，**不是**数据库字段。每次 dispatch 新 Job 时传入新值，不会持久化到数据库。

---

## 五、Docker 版本验证（阶段3）

**代码位置**: [Server::validateDockerEngineVersion()](file:///d:/fz/0601-1/solo-dogfeeding/code/93-coolify/app/Models/Server.php#L1384-L1402)

```php
public function validateDockerEngineVersion()
{
    $dockerVersionRaw = instant_remote_process(['docker version --format json'], ...);
    $dockerVersion = data_get($dockerVersionJson, 'Server.Version', '0.0.0');
    $dockerVersion = checkMinimumDockerEngineVersion($dockerVersion);

    if (is_null($dockerVersion)) {
        $this->settings->is_usable = false;
        $this->settings->save();
        return false;
    }
    // 成功
    $this->settings->is_reachable = true;
    $this->settings->is_usable = true;
    $this->settings->save();
    ServerReachabilityChanged::dispatch($this);
    return true;
}
```

### 5.1 失败路径的状态字段全貌

Docker 版本验证失败时，状态字段分两处设置，按执行顺序：

**第一步：在 `validateDockerEngineVersion()` 内部**（[L1390-L1394](file:///d:/fz/0601-1/solo-dogfeeding/code/93-coolify/app/Models/Server.php#L1390-L1394)）
```php
if (is_null($dockerVersion)) {
    $this->settings->is_usable = false;  // ①
    $this->settings->save();             // ② 持久化到 server_settings 表
    return false;
}
```

**第二步：回到 Job 的失败分支**（[L149-L161](file:///d:/fz/0601-1/solo-dogfeeding/code/93-coolify/app/Jobs/ValidateAndInstallServerJob.php#L149-L161)）
```php
if (! $dockerVersion) {
    $errorMessage = 'Minimum Docker Engine version ...';
    $this->server->update([              // ③ 写入 servers 表
        'validation_logs' => $errorMessage,
        'is_validating' => false,
    ]);
    return;  // 终止流程
}
```

**完整执行顺序和字段变化**：

| 步骤 | 代码位置 | 修改字段 | 值 | 表 |
|-----|---------|---------|----|-----|
| ① | `validateDockerEngineVersion` L1391 | `settings.is_usable` | `false` | 内存 |
| ② | `validateDockerEngineVersion` L1392 | 持久化 | - | `server_settings` |
| ③ | Job L152-L155 | `validation_logs` | 错误信息（提示手动安装 Docker） | `servers` |
| ③ | Job L152-L155 | `is_validating` | `false` | `servers` |

**校验结果 → 状态字段** 完整对应关系：

| 校验结果 | `validation_logs` | `is_validating` | `settings.is_reachable` | `settings.is_usable` | 广播事件 |
|---------|-------------------|-----------------|------------------------|---------------------|----------|
| 版本不满足 | 写入错误信息（提示手动安装 Docker） | `false` | 未修改（保持之前的值） | `false`（在 validateDockerEngineVersion 内部设置） | - |
| 版本满足 | 未修改 | 未修改（后续代理启动后才置为 false） | `true` | `true` | `ServerReachabilityChanged` |

**补充说明**：
- 版本不满足时，**没有重试机制**，直接终止流程
- `settings.is_usable = false` 在模型方法内部先设置并保存
- 回到 Job 后再设置 `validation_logs` 和 `is_validating = false`，然后 `return`
- 注意 `settings.is_reachable` 在此处不修改，它的设置点是 `validateConnection()` 和版本满足时

---

## 六、接入完成后的状态持久化

**代码位置**: [ValidateAndInstallServerJob](file:///d:/fz/0601-1/solo-dogfeeding/code/93-coolify/app/Jobs/ValidateAndInstallServerJob.php#L163-L191)

按代码实际执行顺序（从上到下）：

```php
// 1. 启动代理（非构建服务器）
if (! $this->server->isBuildServer()) {
    $proxyShouldRun = CheckProxy::run($this->server, true);
    if ($proxyShouldRun) {
        // 先同步创建网络，避免异步代理启动时的竞态
        instant_remote_process(ensureProxyNetworksExist($this->server)->toArray(), $this->server, false);
        StartProxy::dispatch($this->server);
    }
}

// 2. 标记验证完成
$this->server->update(['is_validating' => false]);

// 3. 收集服务器元数据（OS、CPU、内存等）
$this->server->gatherServerMetadata();

// 4. 刷新模型，获取最新状态
$this->server->refresh();

// 5. 广播事件通知 UI 更新
ServerValidated::dispatch($this->server->team_id, $this->server->uuid);
ServerReachabilityChanged::dispatch($this->server);
```

**顺序说明**：
- 代理启动放在最前面，因为 `StartProxy::dispatch` 是异步队列任务，先派发后继续执行后续步骤
- `is_validating = false` 在代理启动之后、元数据收集之前置位
- `gatherServerMetadata()` 内部调用 `$this->update()` 更新 `server_metadata` 字段
- `refresh()` 确保后续事件广播时使用的是最新数据

---

## 七、状态字段汇总表

| 字段 | 位置 | 阶段 | 设置时机 | 含义 |
|-----|------|------|---------|------|
| `is_validating` | `servers` 表 | 全流程 | 开始时 `true`，结束/失败时 `false` | 是否正在验证中 |
| `validation_logs` | `servers` 表 | 各失败点 | 校验失败时设置错误信息 | 验证日志/错误信息 |
| `settings.is_reachable` | `server_settings` 表 | 阶段1、3 | `validateConnection()` 和 `validateDockerEngineVersion()` 中设置 | SSH 是否可达 |
| `settings.is_usable` | `server_settings` 表 | 阶段2、3 | `validateDockerEngine()` 和 `validateDockerEngineVersion()` 中设置 | Docker 是否可用 |
| `server_metadata` | `servers` 表 | 接入完成 | `gatherServerMetadata()` 中设置 | OS、CPU、内存等硬件信息 |

---

## 八、异常处理

**代码位置**: [ValidateAndInstallServerJob](file:///d:/fz/0601-1/solo-dogfeeding/code/93-coolify/app/Jobs/ValidateAndInstallServerJob.php#L193-L204)

整个 `handle()` 方法包裹在 `try-catch` 中：

```php
catch (\Throwable $e) {
    Log::error(...);
    $this->server->update([
        'validation_logs' => 'An error occurred during validation: ' . $e->getMessage(),
        'is_validating' => false,
    ]);
}
```

**关键点**: 任何未被内部逻辑捕获的异常都会导致验证终止，`is_validating` 设为 `false`，错误信息写入 `validation_logs`。

---

## 九、ValidateServer 与 ValidateAndInstallServerJob 的区别

项目中同时存在两个验证入口：

| 维度 | [ValidateServer](file:///d:/fz/0601-1/solo-dogfeeding/code/93-coolify/app/Actions/Server/ValidateServer.php) | [ValidateAndInstallServerJob](file:///d:/fz/0601-1/solo-dogfeeding/code/93-coolify/app/Jobs/ValidateAndInstallServerJob.php) |
|-----|---------------|--------------------------|
| 用途 | 仅校验，不安装 | 校验 + 自动安装缺失组件 |
| 失败处理 | 抛出异常 | 自动尝试安装，最多重试3次 |
| 重试机制 | 无 | 有（前置依赖和Docker各3次） |
| 代理启动 | 无 | 有 |
| 元数据收集 | 无 | 有 |
| 异常处理 | 直接抛出 | 捕获并写入 validation_logs |

`ValidateServer` 主要用于 API 接口的纯校验场景，`ValidateAndInstallServerJob` 用于完整的服务器接入流程。
