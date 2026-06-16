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
    $this->disableSshMux();  // 先禁用 SSH 多路复用

    if ($this->skipServer()) {
        return ['uptime' => false, 'error' => 'Server skipped.'];
    }
    try {
        instant_remote_process(['ls /'], $this);  // 执行 ls / 测试连接
        // 成功: 更新 is_reachable = true
        return ['uptime' => true, 'error' => null];
    } catch (\Throwable $e) {
        // 失败: 更新 is_reachable = false
        return ['uptime' => false, 'error' => $e->getMessage()];
    }
}
```

**校验结果 → 状态字段** 对应关系：

| 校验结果 | `settings.is_reachable` | 广播事件 |
|---------|------------------------|----------|
| 连接成功 | `true` | `ServerReachabilityChanged` |
| 连接失败 | `false` | `ServerReachabilityChanged` |

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
| **numberOfTries 从哪来？** | 构造函数传入，首次调用为 0 |
| **重试次数计数包含首次吗？** | 包含。numberOfTries=0 是第1次尝试，达到 3 时终止。实际可执行：检查→安装→重试(1)→检查→安装→重试(2)→检查→安装→重试(3)→终止。共 3 次安装机会。 |
| **安装成功后还会重试吗？** | 不会。安装后重新 dispatch 新 Job，新 Job 会从第一步重新开始校验。如果此时前置依赖/Docker 已安装，校验会通过，流程继续向下。 |
| **重试时会重新校验连接吗？** | 会。每次新 Job 都会从 `validateConnection()` 开始，完整重新执行所有校验步骤。 |
| **安装失败会怎样？** | `installPrerequisites()` 和 `installDocker()` 内部通过 `remote_process()` 执行命令。如果命令执行失败会抛出异常，被 Job 的 `catch` 捕获，`is_validating` 设为 `false`，流程终止。 |
| **30秒延迟的作用？** | 给安装命令留出执行时间，避免立即重试时安装还未完成。 |

### 4.4 重试流程时序图

```
首次调用: numberOfTries=0
    ↓
validateConnection() ✓
validateOS() ✓
validatePrerequisites() ✗ (missing: git)
    ↓
numberOfTries (0) < maxTries (3)
    ↓
installPrerequisites() 执行安装
    ↓
dispatch 新 Job, numberOfTries=1, delay 30s
    ↓
当前 Job return → 结束
    ↓
30秒后 → 新 Job 启动: numberOfTries=1
    ↓
validateConnection() ✓
validateOS() ✓
validatePrerequisites() ✓ (git 已安装)
    ↓
继续向下执行 Docker 检查...
```

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
        return false;
    }
    // 成功
    $this->settings->is_reachable = true;
    $this->settings->is_usable = true;
    ServerReachabilityChanged::dispatch($this);
    return true;
}
```

**校验结果 → 状态字段** 对应关系：

| 校验结果 | `settings.is_reachable` | `settings.is_usable` | 广播事件 |
|---------|------------------------|---------------------|----------|
| 版本不满足 | 未修改 | `false` | - |
| 版本满足 | `true` | `true` | `ServerReachabilityChanged` |

---

## 六、接入完成后的状态持久化

**代码位置**: [ValidateAndInstallServerJob](file:///d:/fz/0601-1/solo-dogfeeding/code/93-coolify/app/Jobs/ValidateAndInstallServerJob.php#L163-L192)

全部校验通过后：

```php
// 1. 标记验证完成
$this->server->update(['is_validating' => false]);

// 2. 收集服务器元数据（OS、CPU、内存等）
$this->server->gatherServerMetadata();

// 3. 启动代理（非构建服务器）
if (! $this->server->isBuildServer()) {
    $proxyShouldRun = CheckProxy::run($this->server, true);
    if ($proxyShouldRun) {
        instant_remote_process(ensureProxyNetworksExist(...), ...);
        StartProxy::dispatch($this->server);
    }
}

// 4. 广播事件通知 UI 更新
ServerValidated::dispatch($this->server->team_id, $this->server->uuid);
ServerReachabilityChanged::dispatch($this->server);
```

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
