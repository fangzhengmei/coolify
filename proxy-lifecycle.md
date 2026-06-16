# Coolify 反向代理生命周期深度解析

本文档系统梳理 Coolify 中反向代理（Traefik/Caddy）从配置读取、文件写入、启动重启，到应用域名绑定的完整协同链路，并深入分析配置写入失败对实际流量的影响范围。

---

## 1. 核心代码文件索引

| 功能模块 | 文件路径 | 关键元素 |
|---------|---------|---------|
| 代理配置读取 | [GetProxyConfiguration.php](file:///d:/fz/0601-1/solo-dogfeeding/code/94-coolify/app/Actions/Proxy/GetProxyConfiguration.php) | `handle()`, `configMatchesProxyType()`, `backfillFromDisk()` |
| 代理配置写入 | [SaveProxyConfiguration.php](file:///d:/fz/0601-1/solo-dogfeeding/code/94-coolify/app/Actions/Proxy/SaveProxyConfiguration.php) | `handle()`, 备份策略, base64 传输 |
| 代理启动 | [StartProxy.php](file:///d:/fz/0601-1/solo-dogfeeding/code/94-coolify/app/Actions/Proxy/StartProxy.php) | `handle()`, 网络确保, compose up |
| 代理停止 | [StopProxy.php](file:///d:/fz/0601-1/solo-dogfeeding/code/94-coolify/app/Actions/Proxy/StopProxy.php) | `handle()`, 优雅停止超时 |
| 代理重启 | [RestartProxyJob.php](file:///d:/fz/0601-1/solo-dogfeeding/code/94-coolify/app/Jobs/RestartProxyJob.php) | `buildRestartCommands()`, 组合停止+启动 |
| 代理健康检查 | [CheckProxy.php](file:///d:/fz/0601-1/solo-dogfeeding/code/94-coolify/app/Actions/Proxy/CheckProxy.php) | `handle()`, 端口冲突检测, 并行检查 |
| 代理标签生成 | [docker.php](file:///d:/fz/0601-1/solo-dogfeeding/code/94-coolify/bootstrap/helpers/docker.php) | `generateLabelsApplication()`, `fqdnLabelsForTraefik()`, `fqdnLabelsForCaddy()` |
| 代理默认配置生成 | [proxy.php](file:///d:/fz/0601-1/solo-dogfeeding/code/94-coolify/bootstrap/helpers/proxy.php) | `generateDefaultProxyConfiguration()`, `connectProxyToNetworks()` |
| 服务器模型代理方法 | [Server.php](file:///d:/fz/0601-1/solo-dogfeeding/code/94-coolify/app/models/Server.php) | `proxyType()`, `proxyPath()`, `changeProxy()`, `setupDefaultRedirect()`, `setupDynamicProxyConfiguration()` |
| 定时调度入口 | [Kernel.php](file:///d:/fz/0601-1/solo-dogfeeding/code/94-coolify/app/Console/Kernel.php) | `schedule()`, ServerManagerJob 每分钟 |
| 服务器巡检 Job | [ServerManagerJob.php](file:///d:/fz/0601-1/solo-dogfeeding/code/94-coolify/app/Jobs/ServerManagerJob.php) | `processServerTasks()`, 调度 ServerCheckJob |
| 服务器状态检查 Job | [ServerCheckJob.php](file:///d:/fz/0601-1/solo-dogfeeding/code/94-coolify/app/Jobs/ServerCheckJob.php) | `handle()`, 代理自动恢复逻辑 |
| Sentinel 推送更新 Job | [PushServerUpdateJob.php](file:///d:/fz/0601-1/solo-dogfeeding/code/94-coolify/app/Jobs/PushServerUpdateJob.php) | `updateProxyStatus()`, 代理容器缺失检测 |
| 代理类型枚举 | [ProxyTypes.php](file:///d:/fz/0601-1/solo-dogfeeding/code/94-coolify/app/Enums/ProxyTypes.php) | `NONE`, `TRAEFIK`, `NGINX`, `CADDY` |
| 应用域名属性 | [Application.php](file:///d:/fz/0601-1/solo-dogfeeding/code/94-coolify/app/Models/Application.php) | `fqdns()`, `isForceHttpsEnabled()`, `isGzipEnabled()` |

---

## 2. 反向代理配置读取流程

### 2.1 三级读取降级机制

配置读取由 [GetProxyConfiguration::handle()](file:///d:/fz/0601-1/solo-dogfeeding/code/94-coolify/app/Actions/Proxy/GetProxyConfiguration.php#L16-L69) 执行，遵循严格的三级降级策略：

```
请求配置
  │
  ├─► 第一级：数据库读取
  │     └─ server.proxy.last_saved_proxy_configuration
  │           └─ 校验 configMatchesProxyType() — 确认服务名与代理类型一致
  │                 ├─ 匹配 → 使用数据库配置
  │                 └─ 不匹配 → 置空，进入下一级
  │
  ├─► 第二级：磁盘回读（仅数据库为空时触发）
  │     └─ backfillFromDisk()
  │           └─ SSH 读取 $proxy_path/docker-compose.yml
  │                 ├─ 成功 → 回写数据库，返回配置
  │                 └─ 失败 → 进入下一级
  │
  └─► 第三级：默认配置生成（兜底）
        └─ generateDefaultProxyConfiguration()
              ├─ 提取已有配置中的自定义命令
              └─ 按代理类型（TRAEFIK/CADDY）生成全新默认 docker-compose.yml
```

### 2.2 类型一致性校验

[configMatchesProxyType()](file:///d:/fz/0601-1/solo-dogfeeding/code/94-coolify/app/Actions/Proxy/GetProxyConfiguration.php#L76-L92) 通过解析 YAML 中的 `services` 键名防止配置类型错位：

- TRAEFIK → 必须存在 `services.traefik`
- CADDY → 必须存在 `services.caddy`
- NGINX → 必须存在 `services.nginx`

若 YAML 解析失败（格式损坏），该校验返回 `true`（放行），避免因格式问题导致配置完全不可用。

### 2.3 磁盘回写机制

[backfillFromDisk()](file:///d:/fz/0601-1/solo-dogfeeding/code/94-coolify/app/Actions/Proxy/GetProxyConfiguration.php#L98-L118) 用于兼容历史服务器（数据库中无配置记录），回读成功后立即写入数据库，后续读取直接走数据库路径，避免重复 SSH。

---

## 3. 代理配置文件写入机制

### 3.1 写入与备份策略

[SaveProxyConfiguration::handle()](file:///d:/fz/0601-1/solo-dogfeeding/code/94-coolify/app/Actions/Proxy/SaveProxyConfiguration.php#L14-L48) 执行写入，核心逻辑：

```
计算新配置 MD5 哈希 (base64 → md5)
  │
  ├─ 与旧哈希 last_saved_settings 对比
  │     ├─ 相同 → 跳过备份，仅写入文件
  │     └─ 不同 → 执行备份流程：
  │           ├─ 旧配置复制到 backups/docker-compose.{timestamp}.{short_hash}.yml
  │           ├─ 若已存在相同哈希的备份文件则跳过（去重）
  │           └─ 裁剪旧备份：仅保留最近 10 份（MAX_BACKUPS=10）
  │
  ├─ 更新数据库：
  │     ├─ server.proxy.last_saved_settings = 新哈希
  │     └─ server.proxy.last_saved_proxy_configuration = 完整 YAML
  │
  └─ SSH 写入远端服务器：
        echo '{base64}' | base64 -d | tee $proxy_path/docker-compose.yml
```

### 3.2 双重存储设计

配置同时存储在两个位置：

| 存储位置 | 用途 | 一致性保证 |
|---------|------|-----------|
| 数据库 `server.proxy.last_saved_proxy_configuration` | 快速读取来源，避免 SSH | 每次 SaveProxyConfiguration 时更新 |
| 服务器磁盘 `$proxy_path/docker-compose.yml` | Docker Compose 实际使用的文件 | 通过 base64 管道写入 |

数据库是"真实源"（source of truth），磁盘是"执行源"。两者通过 `last_saved_settings` MD5 哈希做一致性校验。

---

## 4. 反向代理启动与重启的协同流程

### 4.1 启动流程（StartProxy Action）

[StartProxy::handle()](file:///d:/fz/0601-1/solo-dogfeeding/code/94-coolify/app/Actions/Proxy/StartProxy.php#L16-L100) 是代理启动的核心入口：

```
前置检查
  ├─ proxyType 是否为 NONE？
  ├─ force_stop 是否已设？
  └─ 是否为构建服务器？
  │
设置状态为 'starting' → 保存 → 通知 UI (ProxyStatusChangedUI)
  │
获取并持久化配置
  ├─ GetProxyConfiguration::run($server)        ← 读取配置
  ├─ SaveProxyConfiguration::run($server, $conf) ← 写入磁盘+数据库
  └─ 记录 last_applied_settings 哈希（标记"已应用"配置）
  │
构建启动命令
  ├─ Swarm 模式：docker stack deploy
  └─ Standalone 模式：
        ├─ 创建 $proxy_path/dynamic 目录
        ├─ Caddy: 写入默认 Caddyfile (import /dynamic/*.caddy)
        ├─ docker compose pull
        ├─ ensureProxyNetworksExist() — 确保外部网络存在
        ├─ docker compose up -d --wait --remove-orphans
        └─ connectProxyToNetworks() — 将代理接入所有应用网络
  │
执行方式
  ├─ async=true → remote_process()（异步，返回 Activity）
  └─ async=false → instant_remote_process()（同步阻塞）
```

### 4.2 重启流程（RestartProxyJob）

[RestartProxyJob::handle()](file:///d:/fz/0601-1/solo-dogfeeding/code/94-coolify/app/Jobs/RestartProxyJob.php#L36-L73) 将停止和启动组合为一个原子操作序列：

```
设置状态 'restarting' → 清除 force_stop
  │
buildRestartCommands() 构建单条命令链
  │
  ├─ STOP 阶段（30s 优雅超时）：
  │     ├─ docker stop -t=30 coolify-proxy
  │     ├─ docker rm -f coolify-proxy
  │     └─ 轮询等待容器移除（最多 15 次×1s）
  │
  ├─ 配置获取与写入（同 StartProxy）
  │
  └─ START 阶段（同 StartProxy 启动命令）
  │
remote_process() 异步执行完整命令链
  └─ 完成后触发 ProxyStatusChanged 事件
```

关键特性：使用 `WithoutOverlapping` 中间件（key: `restart-proxy-{uuid}`，120s 过期），防止同一服务器的重启作业并发执行。

### 4.3 自动恢复触发链路

代理有两条自动恢复路径，构成高可用保障：

#### 路径 A：定时巡检（ServerCheckJob）

每分钟由 [ServerManagerJob](file:///d:/fz/0601-1/solo-dogfeeding/code/94-coolify/app/Jobs/ServerManagerJob.php) 调度：

```
ServerManagerJob（每分钟）
  └─ processServerTasks()
        └─ ServerCheckJob::dispatch($server)
              └─ handle()
                    ├─ 拉取服务器所有容器列表
                    ├─ 查找 coolify-proxy 容器
                    │     ├─ 未找到 → CheckProxy::run() → StartProxy::run(async:false)
                    │     └─ 已找到 → 更新状态 → ConnectProxyToNetworksJob::dispatchSync()
                    └─ 检查数据库代理、日志容器等
```

#### 路径 B：Sentinel 实时推送（PushServerUpdateJob）

Sentinel 代理实时推送容器状态：

```
Sentinel → SentinelController → PushServerUpdateJob
  └─ updateProxyStatus()
        ├─ 遍历容器列表，检查是否存在 coolify-proxy（running）
        │     ├─ 未找到 → CheckProxy::run() → StartProxy::run(async:false)
        │     └─ 已找到 → 定期 ConnectProxyToNetworksJob（每小时一次，带缓存）
```

两条路径均通过 [CheckProxy::run()](file:///d:/fz/0601-1/solo-dogfeeding/code/94-coolify/app/Actions/Proxy/CheckProxy.php#L17-L114) 做启动前置检查（端口冲突、Cloudflare Tunnel 模式、代理类型等）。

---

## 5. 代理配置与应用域名绑定的判定路径

### 5.1 域名数据模型

应用域名存储在 [Application.fqdn](file:///d:/fz/0601-1/solo-dogfeeding/code/94-coolify/app/Models/Application.php#L2073-L2080) 字段，格式为逗号分隔的 URL 列表：

```
https://app.example.com,https://api.example.com/v1
```

通过 `fqdns()` Attribute 自动解析为数组。每个 URL 可包含：scheme（http/https）、host、path、port。

### 5.2 标签生成判定流程

应用部署时，[generateLabelsApplication()](file:///d:/fz/0601-1/solo-dogfeeding/code/94-coolify/bootstrap/helpers/docker.php#L651-L793) 是域名绑定的核心判定函数：

```
generateLabelsApplication($application, $preview = null)
  │
  ├─ 确定暴露端口
  │     └─ 静态应用 → 80
  │     └─ 普通应用 → ports_exposes_array[0]
  │
  ├─ 判定是否有 FQDN
  │     ├─ 主部署：$application->fqdn 非空 → 逗号切分
  │     └─ Preview 部署：$preview->fqdn 非空 → 逗号切分
  │           └─ fqdn 为空 → 不生成任何代理标签
  │
  └─ 判定服务器的 generate_exact_labels 设置
        │
        ├─ generate_exact_labels = true（精确模式）
        │     └─ 仅为当前代理类型生成标签
        │           ├─ TRAEFIK → fqdnLabelsForTraefik()
        │           └─ CADDY → fqdnLabelsForCaddy()
        │
        └─ generate_exact_labels = false（兼容模式，默认）
              └─ 同时生成 TRAEFIK 和 CADDY 两套标签
                    ├─ fqdnLabelsForTraefik()
                    └─ fqdnLabelsForCaddy()
```

### 5.3 Traefik 标签规则生成细节

[fqdnLabelsForTraefik()](file:///d:/fz/0601-1/solo-dogfeeding/code/94-coolify/bootstrap/helpers/docker.php#L414-L650) 对每个域名按如下规则判定：

```
对每个 $domain URL：
  │
  ├─ 解析为 $host, $path, $schema(https/http), $port
  │
  ├─ schema === 'https'：
  │     ├─ 生成 HTTPS Router：
  │     │     ├─ rule = Host(`{host}`) && PathPrefix(`{path}`)
  │     │     ├─ entryPoints = https
  │     │     ├─ tls.certresolver = letsencrypt
  │     │     └─ path != '/' 时附加 stripprefix 中间件
  │     │
  │     └─ 生成 HTTP Router（可选重定向）：
  │           └─ is_force_https_enabled → middlewares = redirect-to-https
  │
  └─ schema === 'http'：
        └─ 生成 HTTP Router（无 TLS）

附加中间件判定（根据应用设置）：
  ├─ is_gzip_enabled → gzip compress
  ├─ redirect_direction ('www'/'non-www') → www 跳转正则中间件
  ├─ is_http_basic_auth_enabled → basicauth 中间件
  └─ 用户自定义 labels 中提取的 middleware 名称
```

生成的标签被注入 Docker Compose 文件，Traefik 通过 `--providers.docker=true` 自动发现容器标签并创建路由。

### 5.4 网络连通性保障

代理容器必须与应用容器在同一 Docker 网络中才能转发流量。这由两个机制保障：

1. **启动时**：[connectProxyToNetworks()](file:///d:/fz/0601-1/solo-dogfeeding/code/94-coolify/bootstrap/helpers/proxy.php#L108-L134) 在 `docker compose up` 之后执行，将代理接入所有已知应用网络。

2. **运行时**：[ConnectProxyToNetworksJob](file:///d:/fz/0601-1/solo-dogfeeding/code/94-coolify/app/Jobs/ConnectProxyToNetworksJob.php) 周期性地（每小时，带缓存）检查并补连遗漏的网络。

网络收集范围包括：Standalone Docker 网络、Swarm 网络、运行中 Service 的网络、运行中 Compose 应用的网络、运行中 Preview 部署的网络。

---

## 6. 配置写入失败对实际流量的影响范围

### 6.1 故障场景分层分析

#### 场景 1：SaveProxyConfiguration 写入失败（SSH/网络故障）

**影响范围：仅限新启动/重启的代理，运行中代理不受影响**

- 数据库中的 `last_saved_proxy_configuration` 和 `last_saved_settings` **已更新**（在 SSH 写入前执行）
- 服务器磁盘上的 `docker-compose.yml` **仍是旧版本**
- **正在运行的代理容器完全不受影响** — Traefik/Caddy 不依赖磁盘上的 compose 文件运行
- 下次重启/启动代理时，会从数据库读取新配置再次尝试写入
- 若 SSH 故障持续存在但代理正在运行：**流量正常转发，代理无法更新配置但不中断服务**

#### 场景 2：StartProxy 中 SaveProxyConfiguration 成功但 docker compose up 失败

**影响范围：代理容器未启动，所有依赖此代理的流量全部中断**

- 配置文件已正确写入磁盘
- 但容器未成功创建（可能原因：镜像拉取失败、端口冲突、网络驱动错误等）
- `server.proxy.status` 停留在 `starting` 或被标记为 `error`
- 所有应用域名均无法访问 → **全站 502/连接超时**
- ServerCheckJob/PushServerUpdateJob 会在下一个周期检测到代理缺失，自动重试启动

#### 场景 3：动态配置文件（dynamic/ 目录）写入失败

动态配置包括 `coolify.yaml`（Coolify 自身路由）和 `default_redirect_503.yaml`（未匹配域名的 503/重定向）：

| 动态文件 | 写入失败影响 |
|---------|------------|
| `coolify.yaml` | Coolify 面板自身域名路由失效 → 无法通过代理访问 Coolify UI/API |
| `default_redirect_503.yaml` | 未匹配到任何应用路由的域名直接返回 Traefik 默认 404，而非 503 或自定义重定向 |
| 其他自定义动态配置 | 对应自定义路由规则失效 |

**注意**：动态配置由 Traefik 的 `--providers.file.watch=true` 热加载，写入成功后立即生效，无需重启代理。写入失败则旧规则继续生效。

#### 场景 4：应用容器标签生成失败（generateLabelsApplication 异常）

**影响范围：仅单个应用，其他应用和代理正常**

- 该应用容器不带代理标签 → Traefik/Caddy 不创建对应路由
- 访问该应用域名 → 返回 503（默认重定向）或 404
- 其他已部署且标签正确的应用完全不受影响
- 代理本身继续正常运行

#### 场景 5：应用部署时 connectProxyToNetworks 失败

**影响范围：仅该应用的网络可达性**

- 代理容器未接入该应用的 Docker 网络
- 应用容器运行正常，标签也正确
- 但代理转发请求时无法解析应用容器名/IP → **Bad Gateway 502**
- 其他已正确连网的应用不受影响
- 下一次 ConnectProxyToNetworksJob 周期执行（最长 1 小时）时会自动补连

#### 场景 6：RestartProxyJob 中 STOP 成功但 START 失败

**影响范围：全站流量中断，直至下一次自动恢复**

这是最严重的故障场景：

```
时间线：
  T0: 代理正常运行，流量正常
  T1: RestartProxyJob 执行 STOP → 容器被移除 → 流量立即中断
  T2: START 阶段 docker compose up 失败（如端口被新进程占用）
  T3: 作业异常退出，server.proxy.status = 'error'
  T4: 等待 ServerCheckJob（最长 1 分钟）或 PushServerUpdateJob 检测到代理缺失
  T5: 自动重试启动代理
```

中断时长取决于调度周期：最短秒级（Sentinel 实时推送），最长约 1 分钟。

### 6.2 故障影响范围总结矩阵

| 故障点 | 代理进程 | 已部署应用流量 | Coolify 面板 | 新部署应用 | 影响范围 |
|-------|---------|--------------|-------------|----------|---------|
| SaveProxyConfiguration SSH 失败 | ✅ 运行中 | ✅ 正常 | ✅ 正常 | ❌ 无法重启代理 | 仅配置持久化 |
| docker compose up 启动失败 | ❌ 未启动 | ❌ 全部中断 | ❌ 无法访问 | ❌ 无法部署 | 全站 |
| coolify.yaml 写入失败 | ✅ 运行中 | ✅ 正常 | ❌ 无法访问 | ✅ 正常 | 仅 Coolify 自身 |
| default_redirect 写入失败 | ✅ 运行中 | ⚠️ 未知域名返回 404 | ✅ 正常 | ✅ 正常 | 仅默认回退路由 |
| 应用标签生成异常 | ✅ 运行中 | ⚠️ 单应用 404/503 | ✅ 正常 | ⚠️ 该应用异常 | 单个应用 |
| 网络连接失败 | ✅ 运行中 | ⚠️ 单应用 502 | ✅ 正常 | ⚠️ 该应用异常 | 单个应用 |
| 重启中断（停了起不来） | ❌ 已停止 | ❌ 全部中断 | ❌ 无法访问 | ❌ 无法部署 | 全站（短暂） |

### 6.3 数据一致性与回滚保障

1. **配置备份**：SaveProxyConfiguration 每次变更前自动备份上一版本到 `backups/` 目录，保留最近 10 份。可手动回滚：`cp backups/docker-compose.{timestamp}.yml docker-compose.yml && docker compose up -d`。

2. **数据库兜底**：即使磁盘文件完全损坏，数据库中存储的 `last_saved_proxy_configuration` 可随时重新生成磁盘文件。

3. **自动恢复**：ServerCheckJob（分钟级）和 PushServerUpdateJob（Sentinel 实时）构成双重检测机制，代理异常退出后自动拉起。

4. **优雅停机**：StopProxy 和 RestartProxyJob 均使用 `docker stop -t=30`，给代理 30 秒时间排空已有连接再退出。

---

## 7. 完整生命周期时序图

```
用户操作/定时任务
    │
    ▼
StartProxy/RestartProxyJob 被触发
    │
    ├─► GetProxyConfiguration
    │     ├─ 读 DB → 校验类型 → 成功? → 返回
    │     ├─ 失败 → 读磁盘 → 回写 DB → 返回
    │     └─ 失败 → generateDefaultProxyConfiguration → 返回
    │
    ├─► SaveProxyConfiguration
    │     ├─ 计算哈希 → 变更检测 → 备份(可选)
    │     ├─ 更新 DB (last_saved_settings, last_saved_proxy_configuration)
    │     └─ SSH 写磁盘 (base64 → docker-compose.yml)
    │
    ├─► 构建启动命令
    │     ├─ 确保网络存在 (ensureProxyNetworksExist)
    │     ├─ docker compose pull / stack deploy
    │     └─ docker compose up -d --wait
    │
    ├─► connectProxyToNetworks
    │     └─ 将代理接入所有应用网络
    │
    └─► 运行时监控
          ├─ ServerCheckJob (每分钟)
          ├─ PushServerUpdateJob (Sentinel 推送)
          └─ ConnectProxyToNetworksJob (每小时补连)
          │
          └─ 检测到代理缺失? → 回到 StartProxy 流程
```
