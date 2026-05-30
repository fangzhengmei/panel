# Egg/Nest 启动配置注入链路分析

## 1. 整体架构概览

Pterodactyl 面板通过 **Nest（分类）** → **Egg（游戏模板）** → **Server（服务器实例）** 的层级结构实现游戏服务器的配置管理。

**⚠️ 核心认知前提**：启动配置的生成存在两条完全独立的路径——**页面展示路径**与**守护进程执行路径**。二者使用不同的服务类、不同的变量替换逻辑，产出的"启动命令"含义不同，不可混淆。

```
页面展示路径（只读、面向用户）：
  StartupCommandService → 纯文本替换 → 前端展示

守护进程执行路径（运行时、面向 Wings）：
  ServerConfigurationStructureService + EggConfigurationService
  → 完整配置结构 → Wings 拉取后自行组装并执行
```

数据模型关系：

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│    Nest     │────▶│     Egg     │────▶│   Server    │
│  (游戏分类) │     │  (配置模板) │     │  (运行实例) │
└─────────────┘     └─────────────┘     └─────────────┘
       │                   │                   │
       │                   │                   ▼
       │                   │         ┌──────────────────┐
       │                   │         │  ServerVariable  │
       │                   │         │ (实例变量值存储) │
       │                   │         └──────────────────┘
       │                   │                   ▲
       │                   ▼                   │
       │            ┌─────────────┐            │
       │            │ EggVariable │────────────┘
       │            │ (模板变量定义)│
       │            └─────────────┘
       ▼
  一对多关系
```

---

## 2. 两条启动命令路径（关键区分）

### 2.1 页面展示路径：StartupCommandService

**用途**：仅用于前端页面展示，让用户"看到"启动命令长什么样。**Wings 不使用这个命令执行服务器。**

**调用点**（仅两处）：

1. `ServerTransformer::transform()` (`app/Transformers/Api/Client/ServerTransformer.php:68`)
   - 客户端 API 返回服务器信息时，生成 `invocation` 字段
   - 根据用户权限决定是否隐藏变量值（`hideAllValues` 参数）

2. `StartupController::index()` / `StartupController::update()` (`app/Http/Controllers/Api/Client/Servers/StartupController.php:32,78`)
   - 启动设置页面获取和更新变量时，返回 `startup_command` 和 `raw_startup_command`

**替换逻辑** (`app/Services/Servers/StartupCommandService.php:12-23`)：

```php
public function handle(Server $server, bool $hideAllValues = false): string
{
    $find = ['{{SERVER_MEMORY}}', '{{SERVER_IP}}', '{{SERVER_PORT}}'];
    $replace = [$server->memory, $server->allocation->ip, $server->allocation->port];

    foreach ($server->variables as $variable) {
        $find[] = '{{' . $variable->env_variable . '}}';
        $replace[] = ($variable->user_viewable && !$hideAllValues)
            ? ($variable->server_value ?? $variable->default_value)
            : '[hidden]';
    }

    return str_replace($find, $replace, $server->startup);
}
```

**特点**：
- 简单的字符串替换，不涉及嵌套结构
- 对非 `user_viewable` 的变量用 `[hidden]` 替代
- 输出是一条纯文本命令字符串
- **不参与实际服务器启动过程**

### 2.2 守护进程执行路径：Wings 自行组装

**用途**：Wings 启动服务器时，从 Panel 拉取完整配置结构，由 Wings 自行组装出启动命令并执行。

**Wings 拉取配置的 API**（`app/Http/Controllers/Api/Remote/Servers/ServerDetailsController.php:38-59`）：

```php
public function __invoke(Request $request, string $uuid): JsonResponse
{
    // 鉴权：验证请求节点是否有权访问此服务器
    $valid = $transfer
        ? $node->id === $transfer->old_node || $node->id === $transfer->new_node
        : $node->id === $server->node_id;

    if (!$valid) {
        throw new HttpForbiddenException('Requesting node does not have permission to access this server.');
    }

    return new JsonResponse([
        'settings' => $this->configurationStructureService->handle($server),
        'process_configuration' => $this->eggConfigurationService->handle($server),
    ]);
}
```

**返回的两个独立部分**：

| 字段 | 服务 | 职责 |
|------|------|------|
| `settings` | `ServerConfigurationStructureService` | 服务器运行时配置（环境变量、资源限制、端口映射、启动命令模板等） |
| `process_configuration` | `EggConfigurationService` | 进程行为配置（启动检测规则、停止命令、配置文件替换规则） |

**关键区别**：`settings.invocation` 传给 Wings 的是**原始启动命令模板**（如 `java -Xms128M -jar {{SERVER_JARFILE}}`），不是替换后的命令。Wings 收到后，**自行用 `settings.environment` 中的环境变量替换模板中的占位符**，然后执行。

这意味着：
- 页面展示的启动命令 = Panel 端 `str_replace` 的结果
- Wings 实际执行的命令 = Wings 用 `environment` 替换 `invocation` 模板的结果
- 两者替换逻辑不同，极端情况下输出可能不一致

---

## 3. 守护进程拉取配置的鉴权机制

### 3.1 双向鉴权模型

Panel 与 Wings 之间的通信存在两个方向，使用不同的鉴权方式：

**Panel → Wings**（主动推送/指令）：

```
Panel 使用 Node 的 daemon_token（加密存储）作为 Bearer Token
→ DaemonRepository::getHttpClient() 设置 Authorization: Bearer {decrypted_token}
→ Wings 收到后验证 token
```

代码见 `app/Repositories/Wings/DaemonRepository.php:49-64`：
```php
public function getHttpClient(array $headers = []): Client
{
    return new Client([
        'base_uri' => $this->node->getConnectionAddress(),
        'headers' => [
            'Authorization' => 'Bearer ' . $this->node->getDecryptedKey(),
        ],
    ]);
}
```

**Wings → Panel**（拉取配置/上报状态）：

```
Wings 使用 Node 的 daemon_token_id + daemon_token（两段式 Token）作为 Bearer Token
→ 请求 /api/remote/* 路由
→ DaemonAuthenticate 中间件验证
```

### 3.2 DaemonAuthenticate 中间件 (`app/Http/Middleware/Api/Daemon/DaemonAuthenticate.php`)

所有 `/api/remote/*` 路由都经过 `daemon` 中间件组（`app/Http/Kernel.php:86-89`）：

```php
'daemon' => [
    SubstituteBindings::class,
    DaemonAuthenticate::class,
],
```

鉴权流程 (`app/Http/Middleware/Api/Daemon/DaemonAuthenticate.php:34-66`)：

```php
public function handle(Request $request, \Closure $next): mixed
{
    // 豁免路由：daemon.configuration（Wings 首次启动拉取自身配置）
    if (in_array($request->route()->getName(), $this->except)) {
        return $next($request);
    }

    $bearer = $request->bearerToken();
    $parts = explode('.', $bearer);

    // Token 格式：{token_id}.{token_value}
    // token_id → 查找 Node 记录
    // token_value → 与 Node.daemon_token（解密后）比对
    $node = $this->repository->findFirstWhere(['daemon_token_id' => $parts[0]]);

    if (hash_equals((string) $this->encrypter->decrypt($node->daemon_token), $parts[1])) {
        $request->attributes->set('node', $node);
        return $next($request);
    }

    throw new AccessDeniedHttpException('You are not authorized to access this resource.');
}
```

**关键细节**：
- Token 格式为 `{token_id}.{token_value}`，通过 `token_id` 定位 Node 记录
- `daemon_token` 在数据库中是加密存储的，验证时需先解密
- 验证通过后，Node 模型被注入到 `$request->attributes`，后续 Controller 可直接使用
- `daemon.configuration` 路由豁免鉴权（Wings 首次部署时需要拉取自身配置）

### 3.3 二次鉴权：服务器归属校验

即使 Wings 节点通过了 Token 鉴权，拉取具体服务器配置时还会验证该服务器是否属于此节点 (`ServerDetailsController.php:48-54`)：

```php
$valid = $transfer
    ? $node->id === $transfer->old_node || $node->id === $transfer->new_node
    : $node->id === $server->node_id;

if (!$valid) {
    throw new HttpForbiddenException('Requesting node does not have permission to access this server.');
}
```

---

## 4. 配置注入的优先级（精确版）

### 4.1 EnvironmentService 的变量优先级

`EnvironmentService::handle()` 按顺序叠加变量，后写入的覆盖先写入的（`mapWithKeys` → `put`）：

| 优先级 | 来源 | 代码位置 | 说明 |
|--------|------|----------|------|
| 1（最低） | EggVariable.default_value | `EnvironmentService.php:35-37` | 模板定义的默认值 |
| 2 | ServerVariable.variable_value | 同上（通过 `server_value` 属性） | 用户/管理员设定的实例值 |
| 3 | 内置映射 | `EnvironmentService.php:42-44` | `STARTUP`, `P_SERVER_LOCATION`, `P_SERVER_UUID` |
| 4 | 配置文件变量 | `EnvironmentService.php:47-52` | `config/pterodactyl.php` 中 `environment_variables` |
| 5（最高） | 动态注入 | `EnvironmentService.php:55-57` | 通过 `setEnvironmentKey()` 运行时注入 |

**特别注意**：`server_value` 的来源是 `Server.variables` 关系（`Server.php:288-301`），它通过 LEFT JOIN 同时获取 EggVariable 定义和 ServerVariable 实例值：

```php
public function variables(): HasMany
{
    return $this->hasMany(EggVariable::class, 'egg_id', 'egg_id')
        ->select(['egg_variables.*', 'server_variables.variable_value as server_value'])
        ->leftJoin('server_variables', function (JoinClause $join) {
            $join->on('server_variables.variable_id', 'egg_variables.id')
                ->where('server_variables.server_id', $this->id);
        });
}
```

所以 `server_value ?? default_value` 的含义是：如果该服务器有自定义值则使用自定义值，否则回退到模板默认值。

### 4.2 EggConfigurationService 的占位符替换优先级

`EggConfigurationService` 处理的是 Egg `config_files` 中的配置文件替换规则（如修改 server.properties），与 EnvironmentService 是**两个独立的维度**：

- **EnvironmentService** → 生成 Docker 容器的环境变量（`settings.environment`）
- **EggConfigurationService** → 生成配置文件的查找替换规则（`process_configuration.configs`）

EggConfigurationService 的 `replacePlaceholders` 内部调用 `ServerConfigurationStructureService` 获取 legacy 格式结构作为替换数据源。在这个结构中，环境变量位于 `build.env` 路径下。

占位符替换的查找路径：

| 占位符格式 | 替换数据来源 | 替换时机 |
|-----------|-------------|---------|
| `{{server.build.default.port}}` | Legacy 结构的 `build.default.port` | Panel 端替换 |
| `{{env.SERVER_JARFILE}}` | Legacy 结构的 `build.env.SERVER_JARFILE` | Panel 端替换 |
| `{{config.docker.network.interface}}` | 不替换 | Wings 端替换 |

---

## 5. 配置变更后的生效时序

### 5.1 不同操作类型对应的生效策略

| 操作 | 代码入口 | 是否调用 sync | 生效时机 |
|------|----------|:------------:|----------|
| 修改内存/CPU/磁盘等构建参数 | `BuildModificationService::handle()` | ✅ | sync 后立即生效（运行中容器动态调整） |
| 暂停/恢复服务器 | `SuspensionService::toggle()` | ✅ | sync 后立即生效 |
| **修改环境变量** | `StartupController::update()` | ❌ | **下次重启时生效** |
| **修改 Docker 镜像** | `SettingsController::dockerImage()` | ❌ | **下次重启时生效** |
| 修改服务器名称/描述 | `SettingsController::rename()` | ❌ | 下次 Wings 拉取时生效（对运行无影响） |
| 创建服务器 | `ServerCreationService::handle()` | N/A | Wings create API 触发安装流程 |

### 5.2 为什么修改变量后不调用 sync？

查看 `StartupController::update()` (`app/Http/Controllers/Api/Client/Servers/StartupController.php:53-98`)：

```php
public function update(UpdateStartupVariableRequest $request, Server $server): array
{
    // 更新 server_variables 表
    $this->repository->updateOrCreate([...], [
        'variable_value' => $request->input('value') ?? '',
    ]);

    // 仅重新计算展示用启动命令
    $startup = $this->startupCommandService->handle($server);

    // 没有调用 DaemonServerRepository::sync()
    return ...;
}
```

**设计原因**：环境变量的变更影响的是容器的环境变量和启动命令模板中的占位符。对于正在运行的服务器，修改环境变量不会影响已启动的进程（进程已经读过了环境变量）。必须重启服务器才能让新值生效，所以 Panel 没有主动 sync 的必要——Wings 在下次启动该服务器时自然会从 Panel 拉取最新配置。

### 5.3 sync 的精确语义

`DaemonServerRepository::sync()` 调用 `POST /api/servers/{uuid}/sync`，让 Wings **立即**从 Panel 重新拉取配置。但这不等同于"立即生效"——取决于变更的内容类型：

- **资源限制（memory/cpu/disk）**：sync 后 Wings 动态更新容器 cgroup 限制，**运行中的服务器立即受新限制约束**
- **端口/分配变更**：sync 后更新端口映射，**需要重建容器才能完全生效**
- **环境变量/启动命令**：即使 sync 更新了 Wings 本地缓存，**运行中进程不会重新读取环境变量**，必须重启

### 5.4 BuildModificationService 的容错处理

`BuildModificationService::handle()` (`app/Services/Servers/BuildModificationService.php:60-73`) 中，sync 失败不会回滚数据库变更：

```php
$updateData = $this->structureService->handle($server);

if (!empty($updateData['build'])) {
    try {
        $this->daemonServerRepository->setServer($server)->sync();
    } catch (DaemonConnectionException $exception) {
        Log::warning($exception, ['server_id' => $server->id]);
        // 不回滚，仅记录日志
    }
}
```

注释说明了设计意图：Wings 在启动服务器时总是从 Panel 拉取最新配置，所以 sync 失败只是延迟生效，不影响最终一致性。

---

## 6. Wings 配置拉取的完整时序

### 6.1 服务器创建时序

```
Panel                                  Wings
  │                                      │
  │  ServerCreationService::handle()     │
  │  ├─ 验证变量                         │
  │  ├─ 写入 DB（Server + ServerVariable）│
  │  │                                   │
  │  ├─ POST /api/servers ──────────────▶│  只传 uuid + start_on_completion
  │  │                                   │
  │  │                                   ├─ GET /api/remote/servers/{uuid} ──▶ Panel
  │  │                                   │                                    │
  │  │                      ◀────────────│  {settings, process_configuration} │
  │  │                                   │                                    │
  │  │                                   ├─ GET /api/remote/servers/{uuid}/install ▶ Panel
  │  │                                   │                                    │
  │  │                      ◀────────────│  {container_image, entrypoint,     │
  │  │                                   │   script, env}                     │
  │  │                                   │                                    │
  │  │                                   ├─ 执行安装脚本
  │  │                                   ├─ POST /api/remote/servers/{uuid}/install ──▶ Panel
  │  │                                   │   （上报安装结果）                  │
  │  │                                   │                                    │
  │  │                                   ├─ 如 start_on_completion=true：
  │  │                                   │   用拉取的 settings.invocation + settings.environment
  │  │                                   │   组装启动命令并执行
```

### 6.2 Wings 启动服务器的关键步骤

1. Wings 收到启动指令（创建时 `start_on_completion`，或用户通过 `POST /api/servers/{uuid}/power` 发送 `start`）
2. Wings 向 Panel 发起 `GET /api/remote/servers/{uuid}`，获取 `settings` + `process_configuration`
3. Wings 使用 `settings.invocation`（启动命令模板）+ `settings.environment`（环境变量）自行组装最终启动命令
4. Wings 使用 `process_configuration.startup.done` 判断服务器是否启动完成
5. Wings 使用 `process_configuration.stop` 执行停止操作
6. Wings 使用 `process_configuration.configs` 在服务器首次启动前修改配置文件

---

## 7. 数据模型与配置存储

### 7.1 Egg 模型 (`app/Models/Egg.php`)

核心字段：

| 字段 | 类型 | 说明 |
|------|------|------|
| `config_files` | JSON | 配置文件解析规则（如 server.properties 的查找替换规则） |
| `config_startup` | JSON | 启动检测配置（done 字符串、strip_ansi 等） |
| `config_logs` | JSON | 日志配置规则 |
| `config_stop` | string | 停止命令（支持 `^SIGINT` 信号格式） |
| `startup` | string | 启动命令模板，含变量占位符 |
| `config_from` | int | 继承配置的父 Egg ID |
| `copy_script_from` | int | 继承安装脚本的父 Egg ID |
| `docker_images` | array | 可用 Docker 镜像列表 |

配置继承机制：通过 `config_from` 和 `copy_script_from` 字段实现，访问器自动解析：

```php
// app/Models/Egg.php:197-266
public function getInheritConfigFilesAttribute(): ?string
{
    if (!is_null($this->config_files) || is_null($this->config_from)) {
        return $this->config_files;  // 自身有值就用自身
    }
    return $this->configFrom->config_files;  // 否则从父 Egg 继承
}
```

### 7.2 EggVariable 模型 (`app/Models/EggVariable.php`)

| 字段 | 类型 | 说明 |
|------|------|------|
| `env_variable` | string | 环境变量名（如 `SERVER_JARFILE`） |
| `default_value` | string | 默认值 |
| `user_viewable` | bool | 用户是否可见 |
| `user_editable` | bool | 用户是否可编辑 |
| `rules` | string | Laravel 验证规则 |

保留变量名（`EggVariable.php:43`）：
```php
public const RESERVED_ENV_NAMES = 'SERVER_MEMORY,SERVER_IP,SERVER_PORT,ENV,HOME,USER,STARTUP,SERVER_UUID,UUID';
```

### 7.3 ServerVariable 模型 (`app/Models/ServerVariable.php`)

存储每个 Server 实例的实际变量值，核心字段：`server_id`, `variable_id`, `variable_value`

---

## 8. 模板加载流程

### 8.1 Egg JSON 模板格式

以 Minecraft Paper Egg 为例 (`database/Seeders/eggs/minecraft/egg-paper.json`)：

```json
{
    "meta": { "version": "PTDL_v2" },
    "name": "Paper",
    "startup": "java -Xms128M -XX:MaxRAMPercentage=95.0 -jar {{SERVER_JARFILE}}",
    "config": {
        "files": "{\"server.properties\":{\"parser\":\"properties\",\"find\":{\"server-port\":\"{{server.build.default.port}}\"}}}",
        "startup": "{\"done\": \")! For help, type \"}",
        "stop": "stop"
    },
    "scripts": {
        "installation": {
            "script": "#!/bin/ash\n# ...",
            "container": "ghcr.io/pterodactyl/installers:alpine",
            "entrypoint": "ash"
        }
    },
    "variables": [
        {
            "env_variable": "SERVER_JARFILE",
            "default_value": "server.jar",
            "user_viewable": true,
            "user_editable": true,
            "rules": "required|regex:/^([\\w\\d._-]+)(\\.jar)$/"
        }
    ]
}
```

### 8.2 EggParserService (`app/Services/Eggs/EggParserService.php`)

解析导入的 Egg JSON 文件，支持 PTDL_v1 和 PTDL_v2 两种格式。

- `convertToV2()`：将旧版单镜像字段 `image` 转换为 `docker_images` 数组，为变量添加 `field_type`
- `fillFromParsed()`：将解析结果映射到 Egg 模型字段

---

## 9. 变量验证机制

### VariableValidatorService (`app/Services/Servers/VariableValidatorService.php`)

在服务器创建时验证所有变量值是否符合 EggVariable 中定义的规则：

- 管理员级别：验证所有变量
- 用户级别：只验证 `user_editable=true` 且 `user_viewable=true` 的变量

在 `StartupController::update()` 中，对单个变量的更新使用 EggVariable 自身的 `rules` 进行验证：

```php
$this->validate($request, ['value' => $variable->rules]);
```

---

## 10. 配置文件解析与占位符替换

### 10.1 EggConfigurationService (`app/Services/Eggs/EggConfigurationService.php`)

生成 Wings 的 `process_configuration`，包含三个部分：

1. **startup**：启动检测规则（done 字符串、user_interaction、strip_ansi）
2. **stop**：停止命令（command 类型或 signal 类型）
3. **configs**：配置文件替换规则（已替换占位符）

占位符格式：

| 格式 | 替换数据来源 | 替换位置 |
|------|-------------|---------|
| `{{server.build.default.port}}` | Legacy 格式的服务器属性 | Panel 端 |
| `{{env.SERVER_JARFILE}}` | Legacy 格式的环境变量（`build.env.X`） | Panel 端 |
| `{{config.docker.network.interface}}` | 无 | Wings 端 |

旧版兼容映射 (`replaceLegacyModifiers`)：
- `env.SERVER_MEMORY` → `server.build.memory`
- `env.SERVER_IP` → `server.build.default.ip`
- `env.SERVER_PORT` → `server.build.default.port`

### 10.2 停止命令格式转换

```php
protected function convertStopToNewFormat(string $stop): array
{
    if (!Str::startsWith($stop, '^')) {
        return ['type' => 'command', 'value' => $stop];
    }
    $signal = substr($stop, 1);
    return ['type' => 'signal', 'value' => strtoupper($signal)];
}
```

- `stop` → `{type: 'command', value: 'stop'}`
- `^SIGINT` → `{type: 'signal', value: 'SIGINT'}`

---

## 11. 服务器配置结构生成

### ServerConfigurationStructureService (`app/Services/Servers/ServerConfigurationStructureService.php`)

生成 Wings 拉取的 `settings` 部分，包含所有运行时参数：

```php
protected function returnCurrentFormat(Server $server): array
{
    return [
        'uuid' => $server->uuid,
        'meta' => ['name' => $server->name, 'description' => $server->description],
        'suspended' => $server->isSuspended(),
        'environment' => $this->environment->handle($server),  // ← 完整环境变量
        'invocation' => $server->startup,                       // ← 原始启动命令模板
        'skip_egg_scripts' => $server->skip_scripts,
        'build' => [
            'memory_limit' => $server->memory,
            'swap' => $server->swap,
            'io_weight' => $server->io,
            'cpu_limit' => $server->cpu,
            'threads' => $server->threads,
            'disk_space' => $server->disk,
            'oom_disabled' => $server->oom_disabled,
        ],
        'container' => ['image' => $server->image, 'requires_rebuild' => false],
        'allocations' => [
            'force_outgoing_ip' => $server->egg->force_outgoing_ip,
            'default' => ['ip' => $server->allocation->ip, 'port' => $server->allocation->port],
            'mappings' => $server->getAllocationMappings(),
        ],
        'mounts' => $server->mounts->map(...),
        'egg' => ['id' => $server->egg->uuid, 'file_denylist' => $server->egg->inherit_file_denylist],
    ];
}
```

**关键**：`invocation` 字段传的是 `$server->startup`，即数据库中存储的原始模板（如 `java -Xms128M -jar {{SERVER_JARFILE}}`），**不是**替换后的命令。Wings 用 `environment` 中的值自行替换。

---

## 12. 安装脚本与环境变量

### EggInstallController (`app/Http/Controllers/Api/Remote/EggInstallController.php`)

Wings 拉取安装脚本时，也会获取环境变量：

```php
return response()->json([
    'scripts' => [
        'install' => $egg->copy_script_install,
        'privileged' => $egg->script_is_privileged,
    ],
    'config' => [
        'container' => $egg->copy_script_container,
        'entry' => $egg->copy_script_entry,
    ],
    'env' => $this->environment->handle($server),  // ← 安装脚本也使用完整环境变量
]);
```

注意：安装脚本的环境变量通过 `EnvironmentService` 生成，与运行时环境变量是同一套，确保安装脚本可以访问 `SERVER_JARFILE` 等变量。

---

## 13. Panel → Wings 通信（主动指令方向）

### DaemonRepository 基类 (`app/Repositories/Wings/DaemonRepository.php`)

Panel 向 Wings 发送指令时，使用 Node 的 `daemon_token`（解密后）作为 Bearer Token：

```php
public function getHttpClient(array $headers = []): Client
{
    return new Client([
        'base_uri' => $this->node->getConnectionAddress(),
        'headers' => [
            'Authorization' => 'Bearer ' . $this->node->getDecryptedKey(),
            'Accept' => 'application/json',
            'Content-Type' => 'application/json',
        ],
    ]);
}
```

核心 API：

| API | 用途 | 调用时机 |
|-----|------|---------|
| `POST /api/servers` | 创建服务器 | ServerCreationService |
| `POST /api/servers/{uuid}/sync` | 同步配置 | BuildModificationService, SuspensionService |
| `POST /api/servers/{uuid}/power` | 电源操作 | 用户启动/停止/重启 |
| `POST /api/servers/{uuid}/reinstall` | 重装 | ReinstallServerService |
| `DELETE /api/servers/{uuid}` | 删除 | ServerDeletionService |

---

## 14. 关键设计要点

### 14.1 页面展示与实际执行的分离

StartupCommandService 只服务于页面展示，Wings 不使用它生成的命令。Wings 收到 `invocation`（模板）+ `environment`（变量）后自行组装。这种分离意味着：
- 页面上显示的命令可能与 Wings 实际执行的不完全一致（虽然正常情况下应一致）
- Panel 端的 `{{SERVER_MEMORY}}` 替换是纯字符串替换，Wings 端的替换是环境变量注入

### 14.2 配置变更的分类生效策略

| 变更类型 | 是否 sync | 原因 |
|---------|:---------:|------|
| 资源限制 | ✅ sync | 可动态调整运行中容器的 cgroup 限制 |
| 暂停/恢复 | ✅ sync | Wings 需要立即阻止/允许服务器运行 |
| 环境变量 | ❌ | 运行中进程无法重新读取环境变量，sync 也无法生效 |
| Docker 镜像 | ❌ | 需要重建容器，sync 无法触发重建 |
| 名称/描述 | ❌ | 不影响运行时行为 |

### 14.3 配置同步的最终一致性

Wings 在**每次启动服务器时**都从 Panel 拉取最新配置。即使 sync 失败或未调用，下次重启时也会自动获取最新值，保证最终一致性。

### 14.4 容错设计

- `EggConfigurationService::replacePlaceholders()` 中对无效配置跳过处理，避免单个 Egg 错误导致所有服务器崩溃
- `BuildModificationService` 中 sync 失败仅记录日志，不回滚数据库
- `SuspensionService` 中 sync 失败则回滚暂停状态，保证状态一致

---

## 15. 核心文件索引

| 文件 | 主要职责 |
|------|----------|
| `app/Models/Egg.php` | Egg 数据模型，配置继承访问器 |
| `app/Models/EggVariable.php` | 变量模板定义 |
| `app/Models/ServerVariable.php` | 实例变量存储 |
| `app/Models/Node.php` | Node 模型，daemon_token 鉴权 |
| `app/Services/Eggs/EggParserService.php` | Egg JSON 模板解析 |
| `app/Services/Eggs/EggConfigurationService.php` | process_configuration 生成，配置文件占位符替换 |
| `app/Services/Servers/StartupCommandService.php` | **仅页面展示用**启动命令生成 |
| `app/Services/Servers/EnvironmentService.php` | 运行时环境变量生成 |
| `app/Services/Servers/ServerConfigurationStructureService.php` | Wings settings 结构生成 |
| `app/Services/Servers/VariableValidatorService.php` | 变量值验证 |
| `app/Services/Servers/ServerCreationService.php` | 服务器创建流程编排 |
| `app/Services/Servers/BuildModificationService.php` | 构建参数修改 + sync |
| `app/Services/Servers/SuspensionService.php` | 暂停/恢复 + sync |
| `app/Http/Controllers/Api/Remote/Servers/ServerDetailsController.php` | Wings 配置拉取 API |
| `app/Http/Controllers/Api/Remote/EggInstallController.php` | Wings 安装脚本拉取 API |
| `app/Http/Controllers/Api/Client/Servers/StartupController.php` | 前端启动变量修改 API |
| `app/Http/Middleware/Api/Daemon/DaemonAuthenticate.php` | Wings → Panel 鉴权中间件 |
| `app/Repositories/Wings/DaemonServerRepository.php` | Panel → Wings 服务器管理 API |
| `app/Repositories/Wings/DaemonPowerRepository.php` | Panel → Wings 电源操作 API |
