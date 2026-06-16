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

---

## 8. NGINX 代理类型：枚举占位与实际缺失

### 8.1 枚举定义与路径映射

[ProxyTypes](file:///d:/fz/0601-1/solo-dogfeeding/code/94-coolify/app/Enums/ProxyTypes.php) 枚举中定义了四种代理类型：`NONE`、`TRAEFIK`、`NGINX`、`CADDY`。NGINX 在枚举层面与 TRAEFIK/CADDY 地位相同，[Server::proxyPath()](file:///d:/fz/0601-1/solo-dogfeeding/code/94-coolify/app/Models/Server.php#L616-L630) 也为 NGINX 分配了独立路径后缀 `/nginx`：

```
TRAEFIK → /traefik
CADDY   → /caddy
NGINX   → /nginx
```

但路径映射是 NGINX 在代码中唯一真正可用的功能。

### 8.2 各代码分支对 NGINX 的处理差异

| 代码位置 | NGINX 分支状态 | 具体行为 |
|---------|--------------|---------|
| [ProxyTypes 枚举](file:///d:/fz/0601-1/solo-dogfeeding/code/94-coolify/app/Enums/ProxyTypes.php) | ✅ 已定义 | `NGINX` 枚举值存在 |
| [Server::proxyPath()](file:///d:/fz/0601-1/solo-dogfeeding/code/94-coolify/app/Models/Server.php#L616-L630) | ✅ 已映射 | 返回 `/nginx` 路径后缀 |
| [configMatchesProxyType()](file:///d:/fz/0601-1/solo-dogfeeding/code/94-coolify/app/Actions/Proxy/GetProxyConfiguration.php#L76-L92) | ✅ 已校验 | 检查 `services.nginx` 键名 |
| [generateDefaultProxyConfiguration()](file:///d:/fz/0601-1/solo-dogfeeding/code/94-coolify/bootstrap/helpers/proxy.php#L226) | ❌ 返回 null | `else` 分支：非 TRAEFIK/CADDY 均返回 null |
| [generateLabelsApplication()](file:///d:/fz/0601-1/solo-dogfeeding/code/94-coolify/bootstrap/helpers/docker.php#L651-L793) | ❌ 无分支 | switch 仅处理 TRAEFIK/CADDY，无 NGINX case |
| [fqdnLabelsForTraefik()](file:///d:/fz/0601-1/solo-dogfeeding/code/94-coolify/bootstrap/helpers/docker.php#L414-L650) | ❌ 不适用 | 仅生成 Traefik 格式标签 |
| [fqdnLabelsForCaddy()](file:///d:/fz/0601-1/solo-dogfeeding/code/94-coolify/bootstrap/helpers/docker.php) | ❌ 不适用 | 仅生成 Caddy 格式标签 |
| [setupDefaultRedirect()](file:///d:/fz/0601-1/solo-dogfeeding/code/94-coolify/app/Models/Server.php#L350) | ❌ 无分支 | `$default_redirect_file` 变量在 NGINX 分支未赋值 |
| [setupDynamicProxyConfiguration()](file:///d:/fz/0601-1/solo-dogfeeding/code/94-coolify/app/Models/Server.php#L447) | ❌ 无分支 | 仅生成 Traefik YAML / Caddy Caddyfile |
| [proxy.blade.php UI](file:///d:/fz/0601-1/solo-dogfeeding/code/94-coolify/resources/views/livewire/server/proxy.blade.php#L172-L174) | ❌ 已注释 | NGINX 按钮 `disabled` 且被注释 |

### 8.3 NGINX 尝试启动的代码走向

若用户通过数据库或 API 强制将代理类型设为 NGINX，启动流程会经历以下路径：

```
StartProxy::handle()
  │
  ├─ GetProxyConfiguration::run()
  │     ├─ 第一级：DB 中 last_saved_proxy_configuration 为空（从未成功保存过 NGINX 配置）
  │     ├─ 第二级：backfillFromDisk() → SSH 读取 /nginx/docker-compose.yml → 文件不存在 → 失败
  │     └─ 第三级：generateDefaultProxyConfiguration() → 返回 null
  │
  ├─ SaveProxyConfiguration::run($server, null)
  │     └─ 配置为 null，写入空内容到磁盘
  │
  ├─ docker compose up -d → YAML 为空/无效 → 启动失败
  │
  └─ 代理状态停留在 'starting' → 被 ServerCheckJob 标记为异常
```

**结论**：NGINX 是纯粹的枚举占位符，Coolify 当前版本**不可使用** NGINX 作为反向代理。枚举定义、路径映射和类型校验是遗留骨架，缺少核心实现（默认配置生成、标签生成、动态配置生成、UI 入口）。若强行设置为 NGINX 类型，代理将无法启动。

---

## 9. dynamic 目录规则文件的触发点与覆盖含义

### 9.1 dynamic 目录下的文件清单

| 文件名 | 代理类型 | 生成方 | 作用 |
|-------|---------|-------|------|
| `coolify.yaml` | Traefik | `setupDynamicProxyConfiguration()` | Coolify 面板自身的 Traefik 路由规则（Host 规则 + TLS + 中间件） |
| `default_redirect_503.yaml` | Traefik | `setupDefaultRedirect()` | 未匹配任何应用域名的兜底 503/重定向规则 |
| `Caddyfile` | Caddy | `setupDynamicProxyConfiguration()` | Caddy 入口配置（`import /dynamic/*.caddy`） |
| `*.caddy` | Caddy | 用户通过 UI 创建 | 用户自定义 Caddy 动态路由片段 |
| `*.yaml`（非 coolify.yaml） | Traefik | 用户通过 UI 创建 | 用户自定义 Traefik 动态路由规则 |

### 9.2 各生成函数的触发链路

#### setupDynamicProxyConfiguration()

此函数为 Coolify 面板自身生成代理路由，触发点有 3 个：

| 触发场景 | 入口 | 代码位置 |
|---------|------|---------|
| 代理状态变为 `running` | `ProxyStatusChangedNotification` 监听器 | [ProxyStatusChangedNotification.php](file:///d:/fz/0601-1/solo-dogfeeding/code/94-coolify/app/Listeners/ProxyStatusChangedNotification.php) |
| Coolify 启动初始化 | `Init` Artisan 命令 | [Init.php](file:///d:/fz/0601-1/solo-dogfeeding/code/94-coolify/app/Console/Commands/Init.php#L140) |
| 服务器设置页面保存 | `Proxy` Livewire 组件 | 通过 Server 模型方法调用 |

函数内部根据代理类型分支：

```
setupDynamicProxyConfiguration()
  │
  ├─ TRAEFIK:
  │     ├─ 读取 coolify.yaml 内容（包含 Coolify 面板域名、TLS、中间件配置）
  │     ├─ 通过 SSH 写入 $proxy_path/dynamic/coolify.yaml
  │     └─ Traefik 的 --providers.file.watch=true 自动热加载
  │
  ├─ CADDY:
  │     ├─ 生成 Caddyfile 内容（import /dynamic/*.caddy）
  │     ├─ 通过 SSH 写入 $proxy_path/dynamic/Caddyfile
  │     └─ Caddy 的 import 机制自动生效
  │
  └─ NGINX: 无分支，不生成任何文件
```

#### setupDefaultRedirect()

此函数生成未匹配域名的兜底规则，触发点同上（代理 `running` 状态事件）：

```
setupDefaultRedirect()
  │
  ├─ TRAEFIK:
  │     ├─ 生成 default_redirect_503.yaml
  │     │     ├─ redirect_type=503 → 返回 503 Service Unavailable
  │     │     └─ redirect_type=redirect → 302 重定向到目标 URL
  │     └─ 通过 SSH 写入 $proxy_path/dynamic/default_redirect_503.yaml
  │
  ├─ CADDY: 无对应功能（Caddy 默认行为已处理）
  │
  └─ NGINX: 无分支，$default_redirect_file 变量未赋值（PHP 会产生 undefined variable 警告）
```

### 9.3 覆盖语义

每次触发 `setupDynamicProxyConfiguration()` 或 `setupDefaultRedirect()` 时，**完整覆盖**对应文件内容：

- `coolify.yaml` — 始终根据 Coolify 面板当前域名设置重新生成，全量覆盖
- `default_redirect_503.yaml` — 始终根据服务器当前 redirect_type 设置重新生成，全量覆盖
- `Caddyfile` — 始终覆盖为 `import /dynamic/*.caddy`

用户通过 UI 创建的自定义动态配置（非保留文件名）**不会被覆盖**，因为这两个函数仅写入固定文件名。`coolify.yaml` 是保留名称，用户创建动态配置时会被阻止使用该名称。

### 9.4 proxy_settings.json 不存在

经代码搜索确认，**`proxy_settings.json` 文件在 Coolify 代码库中不存在**。Blade 模板 [dynamic-configurations.blade.php](file:///d:/fz/0601-1/solo-dogfeeding/code/94-coolify/resources/views/livewire/server/proxy/dynamic-configurations.blade.php) 中的 `proxy_settings` 仅是 textarea 的 HTML `name` 属性，用于只读展示 dynamic 目录下的保留文件内容，并不对应任何磁盘上的 JSON 文件。

---

## 10. changeProxy 代理切换：旧配置清理与标签重新生成

### 10.1 changeProxy 执行流程

[Server::changeProxy()](file:///d:/fz/0601-1/solo-dogfeeding/code/94-coolify/app/Models/Server.php#L1487) 在用户切换代理类型时调用：

```
changeProxy($new_type)
  │
  ├─ 1. 数据库清理
  │     ├─ proxy->type = $new_type
  │     ├─ proxy->status = 'restarting'
  │     ├─ proxy->last_saved_settings = null
  │     ├─ proxy->last_saved_proxy_configuration = null
  │     └─ proxy->last_applied_settings = null
  │
  └─ 2. 启动新代理
        └─ StartProxy::run($server, async: true)
              ├─ GetProxyConfiguration → 走第三级（DB 为 null → 磁盘文件不匹配 → 默认生成）
              ├─ SaveProxyConfiguration → 写入新类型配置
              └─ docker compose up -d → 启动新类型代理容器
```

### 10.2 旧配置清理分析

#### 数据库侧（✅ 已清理）

`changeProxy()` 将 `last_saved_settings`、`last_saved_proxy_configuration`、`last_applied_settings` 全部置为 null，确保新代理启动时不会误读旧类型配置。

#### 服务器磁盘侧（❌ 未清理）

切换代理类型后，以下旧文件**不会被删除**：

| 遗留内容 | 位置 | 影响 |
|---------|------|------|
| 旧 `docker-compose.yml` | `$proxy_path/docker-compose.yml` | 启动时被新配置覆盖（无害） |
| 旧 `backups/` 目录 | `$proxy_path/backups/` | 历史备份保留，不自动清理（无害但占用磁盘） |
| 旧 `dynamic/coolify.yaml` | `$proxy_path/dynamic/coolify.yaml` | Traefik → Caddy 切换后：Caddy 忽略 .yaml 文件（无害） |
| 旧 `dynamic/default_redirect_503.yaml` | `$proxy_path/dynamic/default_redirect_503.yaml` | 同上（无害） |
| 旧 `dynamic/Caddyfile` | `$proxy_path/dynamic/Caddyfile` | Caddy → Traefik 切换后：Traefik 忽略 Caddyfile（无害） |
| 旧 `dynamic/*.caddy` 用户文件 | `$proxy_path/dynamic/*.caddy` | Traefik 忽略 .caddy 文件（无害） |
| 旧 `dynamic/*.yaml` 用户文件 | `$proxy_path/dynamic/*.yaml` | Caddy 忽略 .yaml 文件（无害） |
| 旧代理 Docker 容器 | `coolify-proxy` 容器 | `docker compose up --remove-orphans` 会移除旧容器 |

**关键点**：不同代理类型的动态配置文件扩展名不同（Traefik 用 `.yaml`，Caddy 用 `.caddy`），因此旧格式的动态配置文件对新代理类型是**惰性的** — 存在但不被加载。Traefik 的 File provider 只读 `.yaml`/`.toml`，Caddy 的 import 指令只读 `.caddy` 文件。

**例外风险**：若 Traefik → Traefik 切换（如先 NONE 再 TRAEFIK），或 Caddy → Caddy 切换，旧的自定义动态配置文件不会被清理，可能产生意外路由规则。这在正常使用中不太可能发生。

### 10.3 应用标签重新生成

代理切换后，已部署应用的容器标签不会自动更新。标签的重新生成取决于 `generate_exact_labels` 设置：

#### 兼容模式（generate_exact_labels = false，默认）

**无需任何操作**。兼容模式下，`generateLabelsApplication()` 同时生成 Traefik 和 Caddy 两套标签：

```
兼容标签 = Traefik 标签 + Caddy 标签
```

无论切换到哪种代理类型，容器上已有对应格式的标签，新代理启动后即可发现路由。这是默认行为，也是推荐的安全模式。

#### 精确模式（generate_exact_labels = true）

**需要手动重新部署应用**。精确模式下，`generateLabelsApplication()` 仅生成当前代理类型的标签：

```
TRAEFIK 时标签 = 仅 Traefik 标签
CADDY 时标签   = 仅 Caddy 标签
```

切换代理后，已有容器上的标签与新代理类型不匹配：

| 切换方向 | 旧标签 | 新代理 | 结果 |
|---------|-------|-------|------|
| TRAEFIK → CADDY | 仅 Traefik 标签 | Caddy | Caddy 无法发现任何路由 → **全站 404** |
| CADDY → TRAEFIK | 仅 Caddy 标签 | Traefik | Traefik 无法发现任何路由 → **全站 404** |

恢复方式：
1. 逐个重新部署应用（触发 `generateLabelsApplication()` 生成新类型标签）
2. 或在应用 Advanced 页面点击"Reset Default Labels"（[Advanced.php](file:///d:/fz/0601-1/solo-dogfeeding/code/94-coolify/app/Livewire/Project/Application/Advanced.php#L163) 的 `resetDefaultLabels()` 方法），然后重新部署
3. 或将 `generate_exact_labels` 改回 `false`（兼容模式），然后重新部署任一应用触发标签刷新

UI 中已有明确提示（[proxy.blade.php](file:///d:/fz/0601-1/solo-dogfeeding/code/94-coolify/resources/views/livewire/server/proxy.blade.php#L36-L38)）：启用精确标签模式后切换代理类型，需要手动重新生成应用标签。

### 10.4 切换流程时序

```
用户选择新代理类型 → changeProxy()
  │
  ├─ DB 清理（nullify last_saved_*, last_applied_settings）
  │
  ├─ StartProxy::run()
  │     ├─ GetProxyConfiguration → 默认生成新类型配置
  │     ├─ SaveProxyConfiguration → 覆盖写入 docker-compose.yml
  │     ├─ docker compose up --remove-orphans → 启动新容器 + 移除旧容器
  │     └─ connectProxyToNetworks → 接入应用网络
  │
  ├─ ProxyStatusChangedNotification 触发
  │     ├─ setupDefaultRedirect() → 写入新类型兜底规则
  │     └─ setupDynamicProxyConfiguration() → 写入新类型 Coolify 自身路由
  │
  ├─ 旧 dynamic/ 文件变为惰性（扩展名不被新代理加载）
  │
  └─ 已部署应用标签状态：
        ├─ 兼容模式 → 无需操作，双标签覆盖
        └─ 精确模式 → 需重新部署所有应用以刷新标签
```

---

## 11. Traefik certresolver 与 ACME 证书自动化

### 11.1 ACME 配置生成

Traefik 的 Let's Encrypt 配置在 [generateDefaultProxyConfiguration()](file:///d:/fz/0601-1/solo-dogfeeding/code/94-coolify/bootstrap/helpers/proxy.php#L316-L318) 中硬编码生成：

```yaml
command:
  - '--certificatesresolvers.letsencrypt.acme.httpchallenge=true'
  - '--certificatesresolvers.letsencrypt.acme.httpchallenge.entrypoint=http'
  - '--certificatesresolvers.letsencrypt.acme.storage=/traefik/acme.json'
```

关键配置：
- **挑战类型**：HTTP-01（通过 80 端口验证域名所有权）
- **证书解析器名称**：`letsencrypt`（全局唯一，所有应用共享）
- **存储位置**：`/traefik/acme.json`（映射到宿主机 `$proxy_path/acme.json`）
- **邮件地址**：未在命令中设置，使用 Traefik 默认行为（需要用户在自定义命令中补充 `--certificatesresolvers.letsencrypt.acme.email=your@email.com`）

### 11.2 证书申请触发机制

证书申请完全由标签驱动，触发点在 [fqdnLabelsForTraefik()](file:///d:/fz/0601-1/solo-dogfeeding/code/94-coolify/bootstrap/helpers/docker.php#L565-L566)：

```
应用部署 → generateLabelsApplication()
  │
  └─ 对每个 HTTPS 域名：
        ├─ traefik.http.routers.{name}.tls=true
        └─ traefik.http.routers.{name}.tls.certresolver=letsencrypt
```

**触发条件链**：
1. 应用 FQDN 字段中包含 `https://` 开头的域名
2. `generateLabelsApplication()` 识别 schema 为 `https`
3. 生成包含 `tls=true` 和 `tls.certresolver=letsencrypt` 的标签
4. Traefik 通过 Docker provider 发现新标签
5. Traefik 自动发起 ACME HTTP-01 挑战
6. Let's Encrypt 服务器访问 `http://{domain}/.well-known/acme-challenge/{token}` 验证
7. 验证通过后，Traefik 颁发证书并写入 `acme.json`

**申请时机**：
- 应用首次部署并启动时（新容器标签出现）
- 新增 HTTPS 域名并重新部署时
- 现有证书到期前 30 天（Traefik 自动续期）

### 11.3 证书续期失败时的流量状态

Coolify **不介入**证书续期逻辑，续期完全由 Traefik 内部处理。续期失败的渐进影响：

| 时间点 | 证书状态 | 流量状态 |
|-------|---------|---------|
| T0（正常运行） | 有效证书 | ✅ HTTPS 正常，浏览器显示安全锁 |
| T-30天（到期前30天） | 有效 | ✅ Traefik 开始尝试自动续期 |
| T-15天 | 仍有效 | ✅ 续期重试中，用户无感知 |
| T（到期日） | 已过期 | ⚠️ 浏览器显示"不安全"警告，但仍可访问（用户点击"高级"继续） |
| T+30天 | 过期很久 | ❌ 部分浏览器可能完全阻止访问，返回证书错误 |

**续期失败的常见原因**：
1. 80 端口被防火墙拦截 → HTTP-01 挑战无法到达
2. 域名 DNS 解析失效 → Let's Encrypt 无法找到服务器
3. 服务器 IP 变更 → DNS 记录未同步
4. Cloudflare 代理开启且 SSL 模式设置不当 → 干扰挑战
5. `acme.json` 文件权限错误（必须 600）

**Coolify 的缺失环节**：代码中没有对 `acme.json` 有效性、证书到期时间、续期失败日志的监控。用户需要：
- 手动检查 Traefik 日志：`docker logs coolify-proxy | grep acme`
- 或在自定义命令中添加 `--log.level=WARN` 以暴露续期错误

---

## 12. ConnectProxyToNetworksJob 每小时补连判定逻辑

### 12.1 调度判定链路

补连机制的核心在 [PushServerUpdateJob::updateProxyStatus()](file:///d:/fz/0601-1/solo-dogfeeding/code/94-coolify/app/Jobs/PushServerUpdateJob.php#L682-L688)：

```
PushServerUpdateJob（Sentinel 推送）
  │
  └─ updateProxyStatus()
        ├─ 代理存在且运行中？
        │
        └─ 检查缓存键 connect-proxy:{server_id}
              ├─ 缓存存在 → 跳过（静默退出）
              └─ 缓存不存在 →
                    ├─ Cache::put(key, true, 3600)  ← TTL 1小时
                    └─ ConnectProxyToNetworksJob::dispatch()
```

**判定参数**：
- 默认间隔：3600 秒（1小时），来自 [config/constants.php](file:///d:/fz/0601-1/solo-dogfeeding/code/94-coolify/config/constants.php#L116) 的 `constants.proxy.connect_networks_interval_seconds`
- 可通过环境变量 `PROXY_CONNECT_NETWORKS_INTERVAL_SECONDS` 覆盖
- 缓存使用 Laravel 默认缓存驱动（通常是 Redis 或 file）
- `WithoutOverlapping` 中间件：key 为 `connect-proxy-networks-{uuid}`，60秒过期

### 12.2 "漏网之鱼"网络检测方式

[collectDockerNetworksByServer()](file:///d:/fz/0601-1/solo-dogfeeding/code/94-coolify/bootstrap/helpers/proxy.php#L40-L106) 是漏网检测的核心，全量收集以下网络：

```
应连接网络集合 =
  ├─ Standalone Docker 网络（server.standaloneDockers）
  ├─ Swarm 网络（server.swarmDockers）
  ├─ 运行中 Service 的网络（$service->networks()）
  ├─ 运行中 Compose 应用的网络（$app->uuid）
  └─ 运行中 Preview 部署的网络（{$app_uuid}-{$pr_id}）
```

然后与 [collectProxyDockerNetworksByServer()](file:///d:/fz/0601-1/solo-dogfeeding/code/94-coolify/bootstrap/helpers/proxy.php#L25-L39) 获取的**实际已连接网络**对比？

**答案：不对比**。代码不做差集计算，而是直接对所有"应连接网络"执行幂等连接命令：

```
docker network connect {network} coolify-proxy >/dev/null 2>&1 || true
```

- `>/dev/null 2>&1`：静默所有输出
- `|| true`：即使已连接（Docker 会报错 `network is already connected`）也不中断命令链

**漏网之鱼捕获场景**：
1. 代理崩溃重启后丢失网络连接
2. Swarm 模式下通过 UI 手动添加的新网络
3. 新部署的应用在 `connectProxyToNetworks()` 执行后才创建网络
4. 应用删除网络后重建（网络 ID 变更）
5. Docker daemon 重启导致网络连接状态异常

### 12.3 即时触发 vs 周期补连

| 触发方式 | 场景 | 代码位置 |
|---------|------|---------|
| 即时触发（同步） | 应用部署完成时 | [ApplicationDeploymentJob](file:///d:/fz/0601-1/solo-dogfeeding/code/94-coolify/app/Jobs/ApplicationDeploymentJob.php#L796) |
| 即时触发（同步） | 服务启动时 | [StartService](file:///d:/fz/0601-1/solo-dogfeeding/code/94-coolify/app/Actions/Service/StartService.php#L44) |
| 即时触发（同步） | 代理启动完成时 | [StartProxy](file:///d:/fz/0601-1/solo-dogfeeding/code/94-coolify/app/Actions/Proxy/StartProxy.php) |
| 即时触发（同步） | ServerCheckJob 检测到代理运行中 | [ServerCheckJob](file:///d:/fz/0601-1/solo-dogfeeding/code/94-coolify/app/Jobs/ServerCheckJob.php#L95) `dispatchSync()` |
| 周期补连（异步） | PushServerUpdateJob 每小时一次 | [PushServerUpdateJob](file:///d:/fz/0601-1/solo-dogfeeding/code/94-coolify/app/Jobs/PushServerUpdateJob.php#L684-L688) |

**补连覆盖的盲区**：即时触发覆盖 99% 的场景，每小时补连是兜底，仅捕获以下边缘情况：
- 代理崩溃重启后 ServerCheckJob 未及时检测到
- 应用部署时 `docker network connect` 命令执行失败但未被捕获
- 底层 Docker 网络状态异常（如 daemon 重启）

---

## 13. CheckProxy 端口冲突检测依据与并行检查界限

### 13.1 端口冲突检测依据

[CheckProxy::handle()](file:///d:/fz/0601-1/solo-dogfeeding/code/94-coolify/app/Actions/Proxy/CheckProxy.php#L17-L114) 的检测流程：

```
1. 从 docker-compose.yml 解析待检测端口
   ├─ TRAEFIK: services.traefik.ports → ["80:80", "443:443", ...]
   └─ CADDY: services.caddy.ports → ["80:80", "443:443", ...]
   提取冒号前的宿主端口：[80, 443, 443, 8080] → 去重后 [80, 443, 8080]

2. 对每个端口执行三级降级检测（按优先级）
   ├─ 第一级：ss 命令（优先）
   │     ss -Htuln state listening sport = :{port}
   │     识别 0.0.0.0:{port} 和 :::{port} 双栈
   │
   ├─ 第二级：netstat 命令（ss 不可用时）
   │     netstat -tuln | grep ':{port} '
   │
   └─ 第三级：nc 命令（两者都不可用时）
         nc -z -w1 127.0.0.1 {port}

3. 智能排除非冲突场景
   ├─ ✅ 端口被 coolify-proxy 自身占用 → 放行
   ├─ ✅ 端口仅被 docker-proxy 或 coolify 相关进程占用 → 放行
   ├─ ✅ 标准双栈监听（IPv4+IPv6 各一个）→ 放行
   └─ ❌ 其他进程占用 → 冲突
```

**关键检测逻辑**在 [buildPortCheckCommands()](file:///d:/fz/0601-1/solo-dogfeeding/code/94-coolify/app/Actions/Proxy/CheckProxy.php#L166-L239) 的 Shell 脚本中：

```bash
# 第一步：先检查是否是 coolify-proxy 自己在用
CONTAINER_ID=$(docker ps -a --filter name=coolify-proxy --format '{{.ID}}');
if [ ! -z "$CONTAINER_ID" ]; then
    if docker inspect $CONTAINER_ID --format '{{json .NetworkSettings.Ports}}' | grep -q '"80/tcp"'; then
        echo 'proxy_using_port'; exit 0;  # 自身占用，无冲突
    fi;
fi;

# 第二步：ss 检测，计数监听条目
count=$(echo "$ss_output" | grep -c ':80 ');
if [ $count -le 2 ] && (echo "$ss_output" | grep -q 'docker\|coolify'); then
    echo 'port_free'; exit 0;  # docker 或 coolify 占用，放行
fi;
```

### 13.2 单服务器内并行检查

同一服务器的多个端口使用 [Process::concurrently()](file:///d:/fz/0601-1/solo-dogfeeding/code/94-coolify/app/Actions/Proxy/CheckProxy.php#L128-L133) 并行检测：

```php
$results = Process::concurrently(function ($pool) use ($server, $ports, $proxyContainerName) {
    foreach ($ports as $port) {
        $commands = $this->buildPortCheckCommands($server, $port, $proxyContainerName);
        $pool->command($commands['ssh_command'])->timeout(10);
    }
});
```

**并行特性**：
- 并发级别：Laravel Process 池自动管理，默认与 CPU 核心数相当
- 单个端口超时：10 秒（`->timeout(10)`）
- 失败降级：并发检测抛出异常时，自动回退到 [isPortConflict()](file:///d:/fz/0601-1/solo-dogfeeding/code/94-coolify/app/Actions/Proxy/CheckProxy.php#L285-L416) 的顺序检测
- 错误处理：单个端口检测失败（进程异常退出）时，假设"无冲突"以避免误报

### 13.3 跨服务器并行检查界限

跨服务器的"并行"**不使用 Process::concurrently**，而是通过队列分发实现异步并行：

```
ServerManagerJob（每分钟运行一次，单进程）
  │
  ├─ getServers() → 获取所有服务器集合
  │
  ├─ dispatchConnectionChecks()
  │     └─ 对每个服务器：ServerConnectionCheckJob::dispatch($server)
  │           ↳ 全部进入队列，由 Horizon 多进程并行消费
  │
  └─ processScheduledTasks()
        └─ 对每个服务器顺序遍历：
              └─ processServerTasks($server)
                    └─ 条件满足时：ServerCheckJob::dispatch($server)
                          ↳ 同样进入队列并行消费
```

**并行界限总结表**：

| 层面 | 并行方式 | 并发控制 | 超时 | 代码位置 |
|-----|---------|---------|------|---------|
| 单服务器多端口 | Process::concurrently() 同步并行 | 进程池自动管理 | 每个端口 10s | [CheckProxy.php L128](file:///d:/fz/0601-1/solo-dogfeeding/code/94-coolify/app/Actions/Proxy/CheckProxy.php#L128) |
| 多服务器 | 队列 Job 异步并行 | Horizon worker 数量决定 | 每个 Job 60s | [ServerManagerJob.php L83-L100](file:///d:/fz/0601-1/solo-dogfeeding/code/94-coolify/app/Jobs/ServerManagerJob.php#L83-L100) |
| 同一服务器多 Job | WithoutOverlapping 中间件 | keyed by server uuid，60s 过期 | - | [ConnectProxyToNetworksJob.php L33](file:///d:/fz/0601-1/solo-dogfeeding/code/94-coolify/app/Jobs/ConnectProxyToNetworksJob.php#L33) |

**跨服务器并行的天然瓶颈**：
- `dispatchConnectionChecks()` 是 `$servers->each()` 顺序遍历，1000 台服务器需要遍历完才全部入队
- 但入队后由 Horizon 多 worker 真正并行执行 SSH 检查
- Sentinel 健康的服务器跳过 SSH 检查，进一步提升效率

### 13.4 检测结果的行为差异

| 检测结果 | fromUI=true（用户手动启动） | fromUI=false（自动恢复） |
|---------|----------------------------|-------------------------|
| 有端口冲突 | throw Exception 展示给用户 | 静默 return false，不启动 |
| 无端口冲突 | return true，继续启动流程 | return true，继续启动流程 |
| Cloudflare Tunnel 模式 | - | return false，不启动代理 |
| 代理已 running | return false，不重复启动 | return false，不重复启动 |
