# Egg/Nest 启动配置注入链路分析

## 1. 整体架构概览

Pterodactyl 面板通过 **Nest（分类）** → **Egg（游戏模板）** → **Server（服务器实例）** 的层级结构实现游戏服务器的配置管理。

**核心认知前提**：启动配置的生成存在两条独立的路径——**页面展示路径**与**守护进程拉取路径**。二者使用不同的服务类、不同的数据结构，产出的"启动命令"含义不同，不可混淆。

```
页面展示路径（只读、面向用户）：
  StartupCommandService → str_replace 替换 → 纯文本返回前端

守护进程拉取路径（运行时、面向 Wings）：
  Wings 请求 GET /api/remote/servers/{uuid}
  → Panel 返回 { settings, process_configuration }
  → Wings 拿到后如何使用这些数据，属于 Wings 端行为
```

**可证实的事实与推断边界的标注约定**：
- ✅ **已证实**：Panel 代码直接可证的结论
- ⚠️ **推断边界**：涉及 Wings 端行为，Panel 代码无法直接证实，基于 API 契约或注释的合理推断

---

## 2. 两条启动命令路径（关键区分）

### 2.1 页面展示路径：StartupCommandService

**用途**：仅用于前端页面展示。**Panel 代码中没有任何地方将此命令传给 Wings。**

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

✅ **已证实**：
- `StartupCommandService` 的输出仅用于 `ServerTransformer` 和 `StartupController`
- 对非 `user_viewable` 的变量用 `[hidden]` 替代
- 输出是一条纯文本命令字符串

### 2.2 守护进程拉取路径

**用途**：Wings 通过 Remote API 从 Panel 拉取完整配置结构。

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

✅ **已证实**：Panel 返回两个独立结构：

| 字段 | 生成服务 | 内容 |
|------|---------|------|
| `settings` | `ServerConfigurationStructureService` | 运行时配置：环境变量、资源限制、端口映射、启动命令模板等 |
| `process_configuration` | `EggConfigurationService` | 进程行为配置：启动检测规则、停止命令、配置文件替换规则 |

✅ **已证实**：`settings.invocation` 的值是 `$server->startup`（`ServerConfigurationStructureService.php:53`），即数据库中存储的原始模板（如 `java -Xms128M -jar {{SERVER_JARFILE}}`），不是替换后的命令。

⚠️ **推断边界**：Wings 收到 `settings.invocation`（原始模板）和 `settings.environment`（环境变量字典）后，如何组装出最终启动命令并执行，属于 Wings 端行为，Panel 代码无法证实。常见的推断是 Wings 用 environment 中的值替换 invocation 中的占位符，但此推断不能由 Panel 代码独立证明。

⚠️ **推断边界**：页面展示路径与守护进程拉取路径产出的"启动命令"在替换逻辑上存在差异（前者是 Panel 端 `str_replace`，后者是 Wings 端处理），极端情况下两者结果可能不一致，但这需要阅读 Wings 源码才能确认。

---

## 3. 守护进程拉取配置的鉴权机制

### 3.1 双向通信模型

Panel 与 Wings 之间存在两个方向的通信：

**Panel → Wings**（主动推送/指令）：

✅ **已证实**：Panel 使用 Node 的 `daemon_token`（解密后）作为 Bearer Token 发送请求：

```php
// app/Repositories/Wings/DaemonRepository.php:49-64
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

✅ **已证实**：`Node::getDecryptedKey()` (`app/Models/Node.php:189-193`) 解密数据库中的 `daemon_token` 字段返回。

**Wings → Panel**（拉取配置/上报状态）：

✅ **已证实**：Wings 请求 `/api/remote/*` 路由，经过 `daemon` 中间件组：

```php
// app/Http/Kernel.php:86-89
'daemon' => [
    SubstituteBindings::class,
    DaemonAuthenticate::class,
],
```

### 3.2 DaemonAuthenticate 中间件 (`app/Http/Middleware/Api/Daemon/DaemonAuthenticate.php`)

✅ **已证实**的鉴权流程：

```php
public function handle(Request $request, \Closure $next): mixed
{
    // 豁免：daemon.configuration 路由
    if (in_array($request->route()->getName(), $this->except)) {
        return $next($request);
    }

    $bearer = $request->bearerToken();
    $parts = explode('.', $bearer);

    // Token 格式：{token_id}.{token_value}
    $node = $this->repository->findFirstWhere(['daemon_token_id' => $parts[0]]);

    if (hash_equals((string) $this->encrypter->decrypt($node->daemon_token), $parts[1])) {
        $request->attributes->set('node', $node);
        return $next($request);
    }

    throw new AccessDeniedHttpException('You are not authorized to access this resource.');
}
```

✅ **已证实**：
- Token 格式为 `{token_id}.{token_value}`，通过 `token_id` 定位 Node 记录
- `daemon_token` 在数据库中加密存储，验证时需先解密再 `hash_equals` 比对
- 验证通过后 Node 模型注入 `$request->attributes`
- `daemon.configuration` 路由豁免鉴权

### 3.3 二次鉴权：服务器归属校验

✅ **已证实**：即使节点鉴权通过，`ServerDetailsController` 还会验证服务器是否属于该节点：

```php
$valid = $transfer
    ? $node->id === $transfer->old_node || $node->id === $transfer->new_node
    : $node->id === $server->node_id;
```

---

## 4. 配置注入的数据来源与覆盖关系

### 4.1 EnvironmentService 的变量来源

✅ **已证实**：`EnvironmentService::handle()` (`app/Services/Servers/EnvironmentService.php:33-60`) 按顺序叠加变量，后写入的同名键覆盖先写入的（`put` 方法）：

```php
public function handle(Server $server): array
{
    // 步骤1：Egg 变量（含实例值或默认值）
    $variables = $server->variables->toBase()->mapWithKeys(function (EggVariable $variable) {
        return [$variable->env_variable => $variable->server_value ?? $variable->default_value];
    });

    // 步骤2：内置映射
    foreach ($this->getEnvironmentMappings() as $key => $object) {
        $variables->put($key, object_get($server, $object));
    }

    // 步骤3：配置文件变量
    foreach (config('pterodactyl.environment_variables', []) as $key => $object) {
        $variables->put($key, is_callable($object) ? call_user_func($object, $server) : object_get($server, $object));
    }

    // 步骤4：动态注入变量
    foreach ($this->additional as $key => $closure) {
        $variables->put($key, call_user_func($closure, $server));
    }

    return $variables->toArray();
}
```

✅ **已证实**：`server_value ?? default_value` 是**同一个取值操作**，不是两个优先级层。`server_value` 来自 `Server.variables` 关系的 LEFT JOIN（`Server.php:288-301`），如果该服务器有 `ServerVariable` 记录则 `server_value` 非 null，否则为 null 回退到 `default_value`。

✅ **已证实**：如果步骤 2-4 中存在与步骤 1 同名的键，会覆盖步骤 1 的值。按 `put` 顺序，覆盖关系为：

| 写入顺序 | 来源 | 代码位置 | 能否覆盖同名键 |
|:--------:|------|----------|:------------:|
| 1 | Egg 变量（`server_value ?? default_value`） | `EnvironmentService.php:35-37` | — |
| 2 | 内置映射（`STARTUP`, `P_SERVER_LOCATION`, `P_SERVER_UUID`） | `EnvironmentService.php:42-44` | ✅ 覆盖 |
| 3 | 配置文件变量（`config/pterodactyl.php` 中 `environment_variables`） | `EnvironmentService.php:47-52` | ✅ 覆盖 |
| 4 | 动态注入变量（`setEnvironmentKey()`） | `EnvironmentService.php:55-57` | ✅ 覆盖 |

✅ **已证实**：配置文件变量（步骤3）当前配置为：

```php
// config/pterodactyl.php:150-152
'environment_variables' => [
    'P_SERVER_ALLOCATION_LIMIT' => 'allocation_limit',
],
```

⚠️ **推断边界**：步骤2-4 中的内置映射键名（如 `STARTUP`）和 Egg 变量键名如果冲突，Egg 变量会被覆盖。这是一个潜在的覆盖风险，但 Panel 代码通过 `EggVariable::RESERVED_ENV_NAMES` 禁止了 `SERVER_MEMORY`、`SERVER_IP`、`SERVER_PORT`、`STARTUP` 等保留名作为 Egg 变量名，避免了最常见的冲突场景。

### 4.2 EggConfigurationService 的占位符替换

✅ **已证实**：`EggConfigurationService` 处理的是 Egg `config_files` 中的配置文件替换规则（如修改 server.properties），与 `EnvironmentService` 是**两个独立的维度**：

- **EnvironmentService** → 生成 `settings.environment`（Docker 容器的环境变量）
- **EggConfigurationService** → 生成 `process_configuration.configs`（配置文件的查找替换规则）

✅ **已证实**：`EggConfigurationService::replacePlaceholders()` 内部调用 `ServerConfigurationStructureService::handle($server, [], true)` 获取 legacy 格式结构作为替换数据源。`env.X` 占位符在 legacy 结构中映射到 `build.env.X` 路径（`EggConfigurationService.php:194-197`），而这个 `build.env` 中的值正是 `EnvironmentService` 生成的环境变量。

✅ **已证实**：占位符替换的行为：

| 占位符格式 | Panel 端行为 | 代码位置 |
|-----------|-------------|---------|
| `{{server.X}}` | Panel 端替换为 legacy 结构中对应路径的值 | `EggConfigurationService.php:185-189` |
| `{{env.X}}` | Panel 端替换为 legacy 结构中 `build.env.X` 的值 | `EggConfigurationService.php:194-200` |
| `{{config.X}}` | Panel 端不替换，原样传递 | `EggConfigurationService.php:179-181` |

---

## 5. 配置变更后 Panel 的响应行为

### 5.1 Panel 代码中 sync 调用点

✅ **已证实**：Panel 中仅以下两处调用了 `DaemonServerRepository::sync()`：

1. `BuildModificationService::handle()` (`app/Services/Servers/BuildModificationService.php:68`)
2. `SuspensionService::toggle()` (`app/Services/Servers/SuspensionService.php:52`)

✅ **已证实**：以下操作**没有**调用 sync：

| 操作 | 代码入口 | 是否调用 sync |
|------|----------|:------------:|
| 修改环境变量 | `StartupController::update()` | ❌ |
| 修改 Docker 镜像 | `SettingsController::dockerImage()` | ❌ |
| 修改服务器名称/描述 | `SettingsController::rename()` | ❌ |

### 5.2 sync 的 Panel 端语义

✅ **已证实**：`DaemonServerRepository::sync()` (`app/Repositories/Wings/DaemonServerRepository.php:63-72`) 发送的是一个**无请求体**的 POST 请求：

```php
public function sync(): void
{
    Assert::isInstanceOf($this->server, Server::class);
    try {
        $this->getHttpClient()->post("/api/servers/{$this->server->uuid}/sync");
    } catch (GuzzleException $exception) {
        throw new DaemonConnectionException($exception);
    }
}
```

✅ **已证实**：Panel 不通过 sync API 向 Wings 推送任何配置数据。sync 只是一个触发信号。

⚠️ **推断边界**：Wings 收到 sync 请求后如何处理，属于 Wings 端行为。合理推断是 Wings 会回调 Panel 的 `GET /api/remote/servers/{uuid}` 拉取最新配置，但 Panel 代码无法证实这一点。

⚠️ **推断边界**：关于 sync 后各类配置变更的"立即生效"行为（如资源限制动态调整 cgroup、端口映射更新等），均属于 Wings 端的实现行为。Panel 代码能证明的仅限于：
- `BuildModificationService` 注释中说 "apply resource modifications on the fly"（`BuildModificationService.php:64`），表明 Panel 开发者的**设计意图**是让 sync 使资源修改立即生效
- `SuspensionService` 中 sync 失败时回滚暂停状态（`SuspensionService.php:53-57`），表明暂停操作依赖 sync 成功才能算完成

### 5.3 为什么修改变量后不调用 sync

✅ **已证实**：`StartupController::update()` (`app/Http/Controllers/Api/Client/Servers/StartupController.php:53-98`) 的完整流程是：

1. 验证变量可编辑性
2. 用 EggVariable 的 `rules` 验证新值
3. 更新 `server_variables` 表
4. 重新计算展示用启动命令
5. 记录 Activity 日志
6. 返回结果——**没有调用 sync**

⚠️ **推断边界**：不调用 sync 的原因，Panel 代码本身没有注释说明。之前文档中"运行中进程无法重新读取环境变量"的说法是一般性的操作系统知识，不是 Panel 代码可证实的结论。但可以确认的是：变量修改后 Panel 只更新了数据库，没有主动通知 Wings。

### 5.4 BuildModificationService 的容错设计

✅ **已证实**：`BuildModificationService` 中 sync 失败不回滚数据库（`BuildModificationService.php:62-72`）：

```php
$updateData = $this->structureService->handle($server);

// Because Wings always fetches an updated configuration from the Panel when booting
// a server this type of exception can be safely "ignored" and just written to the logs.
// Ideally this request succeeds, so we can apply resource modifications on the fly, but
// if it fails we can just continue on as normal.
if (!empty($updateData['build'])) {
    try {
        $this->daemonServerRepository->setServer($server)->sync();
    } catch (DaemonConnectionException $exception) {
        Log::warning($exception, ['server_id' => $server->id]);
    }
}
```

✅ **已证实**：注释明确说明了设计意图——"Wings always fetches an updated configuration from the Panel when booting a server"，即 Wings 启动服务器时会从 Panel 拉取最新配置。因此 sync 失败只是延迟生效，不影响最终一致性。

✅ **已证实**：`SuspensionService` 中 sync 失败**会回滚**暂停状态（`SuspensionService.php:53-57`），与 BuildModificationService 行为不同，表明暂停操作对同步成功有更强的依赖。

### 5.5 创建服务器时的数据流

✅ **已证实**：`ServerCreationService::handle()` (`app/Services/Servers/ServerCreationService.php:96-99`) 调用 `DaemonServerRepository::create()`，传给 Wings 的数据只有：

```php
$this->daemonServerRepository->setServer($server)->create(
    Arr::get($data, 'start_on_completion', false) ?? false
);
```

而 `create()` 方法（`DaemonServerRepository.php:42-56`）只发送 `uuid` 和 `start_on_completion`：

```php
public function create(bool $startOnCompletion = true): void
{
    $this->getHttpClient()->post('/api/servers', [
        'json' => [
            'uuid' => $this->server->uuid,
            'start_on_completion' => $startOnCompletion,
        ],
    ]);
}
```

✅ **已证实**：Panel 创建服务器时不向 Wings 推送任何配置数据，只通知 Wings 该 uuid 的服务器需要创建。Wings 需要自行回调 Panel 获取完整配置。

⚠️ **推断边界**：Wings 收到 create 请求后如何获取配置、执行安装脚本、启动服务器，属于 Wings 端行为。Panel 能证明的只是 Wings 可通过 `GET /api/remote/servers/{uuid}` 获取 `settings` + `process_configuration`，以及通过 `GET /api/remote/servers/{uuid}/install` 获取安装脚本信息。

---

## 6. Wings 拉取配置的完整时序

### 6.1 Panel 代码可证实的 API 时序

```
Panel                                     Wings 可调用的 API
  │                                            │
  │  POST /api/servers ──────────────────────▶ │  只传 uuid + start_on_completion
  │                                            │
  │  ◀──── GET /api/remote/servers             │  Wings 拉取 settings + process_configuration
  │        返回 {settings, process_configuration}
  │                                            │
  │  ◀──── GET /api/remote/servers/{uuid}/install │ Wings 拉取安装脚本
  │        返回 {scripts, config, env}
  │                                            │
  │  ◀──── POST /api/remote/servers/{uuid}/install │ Wings 上报安装结果
  │                                            │
  │  ◀──── POST /api/remote/servers/{uuid}/sync   │ sync 触发信号（无请求体）
```

✅ **已证实**：以上所有 API 端点均在 `routes/api-remote.php` 中注册，Controller 实现可查。

⚠️ **推断边界**：这些 API 的调用顺序、是否全部调用、是否还有其他 Wings 内部行为，Panel 代码无法证实。

### 6.2 安装脚本的环境变量

✅ **已证实**：`EggInstallController` (`app/Http/Controllers/Api/Remote/EggInstallController.php`) 返回安装脚本时附带完整环境变量：

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
    'env' => $this->environment->handle($server),
]);
```

✅ **已证实**：安装脚本使用的环境变量与运行时环境变量由同一个 `EnvironmentService::handle()` 生成，确保安装脚本可以访问 `SERVER_JARFILE` 等变量。

---

## 7. 数据模型与配置存储

### 7.1 Egg 模型 (`app/Models/Egg.php`)

核心字段：

| 字段 | 类型 | 说明 |
|------|------|------|
| `config_files` | JSON | 配置文件解析规则 |
| `config_startup` | JSON | 启动检测配置（done 字符串、strip_ansi 等） |
| `config_logs` | JSON | 日志配置规则 |
| `config_stop` | string | 停止命令（支持 `^SIGINT` 信号格式） |
| `startup` | string | 启动命令模板，含变量占位符 |
| `config_from` | int | 继承配置的父 Egg ID |
| `copy_script_from` | int | 继承安装脚本的父 Egg ID |
| `docker_images` | array | 可用 Docker 镜像列表 |

✅ **已证实**：配置继承机制通过 `config_from` 和 `copy_script_from` 字段实现，访问器自动解析：

```php
// app/Models/Egg.php:197-266
public function getInheritConfigFilesAttribute(): ?string
{
    if (!is_null($this->config_files) || is_null($this->config_from)) {
        return $this->config_files;
    }
    return $this->configFrom->config_files;
}
```

### 7.2 EggVariable 模型 (`app/Models/EggVariable.php`)

| 字段 | 类型 | 说明 |
|------|------|------|
| `env_variable` | string | 环境变量名 |
| `default_value` | string | 默认值 |
| `user_viewable` | bool | 用户是否可见 |
| `user_editable` | bool | 用户是否可编辑 |
| `rules` | string | Laravel 验证规则 |

✅ **已证实**：保留变量名（`EggVariable.php:43`）：
```php
public const RESERVED_ENV_NAMES = 'SERVER_MEMORY,SERVER_IP,SERVER_PORT,ENV,HOME,USER,STARTUP,SERVER_UUID,UUID';
```

### 7.3 ServerVariable 模型 (`app/Models/ServerVariable.php`)

核心字段：`server_id`, `variable_id`, `variable_value`

### 7.4 Server.variables 关系

✅ **已证实**：`Server::variables()` (`Server.php:288-301`) 通过 LEFT JOIN 同时获取 EggVariable 定义和 ServerVariable 实例值：

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

✅ **已证实**：如果某个 EggVariable 没有对应的 ServerVariable 记录，`server_value` 为 null；`EnvironmentService` 中 `server_value ?? default_value` 即：有实例值用实例值，否则回退到默认值。

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

✅ **已证实**：
- `handle()` 解析 PTDL_v1 和 PTDL_v2 格式的 JSON 文件
- `convertToV2()` 将旧版单镜像字段 `image` 转换为 `docker_images` 数组，为变量添加 `field_type`
- `fillFromParsed()` 将解析结果映射到 Egg 模型字段

---

## 9. 变量验证机制

✅ **已证实**：`VariableValidatorService` (`app/Services/Servers/VariableValidatorService.php`) 在服务器创建时验证所有变量值：
- 管理员级别：验证所有变量
- 用户级别：只验证 `user_editable=true` 且 `user_viewable=true` 的变量

✅ **已证实**：`StartupController::update()` 对单个变量的更新使用 EggVariable 自身的 `rules` 进行验证：

```php
$this->validate($request, ['value' => $variable->rules]);
```

---

## 10. 配置文件解析与占位符替换

### 10.1 EggConfigurationService (`app/Services/Eggs/EggConfigurationService.php`)

✅ **已证实**：生成 `process_configuration`，包含三个部分：

1. **startup**：启动检测规则（done 字符串转换为数组、user_interaction 置空、strip_ansi）
2. **stop**：停止命令（command 类型或 signal 类型）
3. **configs**：配置文件替换规则（已替换 `server.X` 和 `env.X` 占位符）

✅ **已证实**：旧版兼容映射 (`replaceLegacyModifiers`)：
- `env.SERVER_MEMORY` → `server.build.memory`
- `env.SERVER_IP` → `server.build.default.ip`
- `env.SERVER_PORT` → `server.build.default.port`
- `config.docker.interface` → `config.docker.network.interface`

### 10.2 停止命令格式转换

✅ **已证实**：

```php
// EggConfigurationService.php:57-72
protected function convertStopToNewFormat(string $stop): array
{
    if (!Str::startsWith($stop, '^')) {
        return ['type' => 'command', 'value' => $stop];
    }
    $signal = substr($stop, 1);
    return ['type' => 'signal', 'value' => strtoupper($signal)];
}
```

---

## 11. 服务器配置结构生成

### ServerConfigurationStructureService (`app/Services/Servers/ServerConfigurationStructureService.php`)

✅ **已证实**：生成 Wings 拉取的 `settings` 部分：

```php
protected function returnCurrentFormat(Server $server): array
{
    return [
        'uuid' => $server->uuid,
        'meta' => ['name' => $server->name, 'description' => $server->description],
        'suspended' => $server->isSuspended(),
        'environment' => $this->environment->handle($server),
        'invocation' => $server->startup,                       // 原始模板
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
        'container' => ['image' => $server->image, 'oom_disabled' => $server->oom_disabled, 'requires_rebuild' => false],
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

✅ **已证实**：`invocation` 的值就是 `$server->startup`，即数据库中的原始模板字符串。

✅ **已证实**：此服务还支持 `legacy` 格式输出（`returnLegacyFormat`），供 `EggConfigurationService::replacePlaceholders()` 内部使用。legacy 格式中环境变量位于 `build.env` 路径下。

---

## 12. Panel → Wings 通信（主动指令方向）

✅ **已证实**：Panel 向 Wings 发送指令时，使用 Node 的 `daemon_token` 解密值作为 Bearer Token。

核心 API：

| API | 请求体 | 用途 | Panel 调用点 |
|-----|-------|------|-------------|
| `POST /api/servers` | `{uuid, start_on_completion}` | 创建服务器 | `ServerCreationService` |
| `POST /api/servers/{uuid}/sync` | 无 | 同步配置 | `BuildModificationService`, `SuspensionService` |
| `POST /api/servers/{uuid}/power` | `{action}` | 电源操作 | 用户启动/停止/重启 |
| `POST /api/servers/{uuid}/reinstall` | 无 | 重装 | `ReinstallServerService` |
| `DELETE /api/servers/{uuid}` | 无 | 删除 | `ServerDeletionService` |

✅ **已证实**：所有 Panel → Wings 的 API 都**不携带配置数据**，只传递标识符和动作。Wings 需要配置数据时回调 Panel 的 Remote API。

---

## 13. 关键设计要点

### 13.1 页面展示与守护进程拉取的分离

✅ **已证实**：
- `StartupCommandService` 只服务于页面展示，调用点限于 `ServerTransformer` 和 `StartupController`
- `ServerConfigurationStructureService` 返回的 `settings.invocation` 是原始模板
- 两者恰好都叫 `invocation`，但含义不同：前者是替换后的文本，后者是模板字符串

⚠️ **推断边界**：Wings 如何使用 `invocation` + `environment` 组装启动命令，属于 Wings 端行为。

### 13.2 Panel 不推送配置数据，Wings 按需拉取

✅ **已证实**：Panel 向 Wings 发送的所有 API（create/sync/power/reinstall/delete）都不携带服务器配置数据。Wings 需要配置时，通过 `GET /api/remote/servers/{uuid}` 拉取。

✅ **已证实**：`BuildModificationService` 注释（`BuildModificationService.php:62-65`）确认 Wings 在启动服务器时会从 Panel 拉取最新配置，这是 sync 失败可以容忍的依据。

### 13.3 配置变更的分类响应

✅ **已证实**：Panel 代码对不同类型的配置变更有明确的处理差异：

| 变更类型 | Panel 代码行为 | 是否 sync |
|---------|---------------|:---------:|
| 资源限制（memory/cpu/disk/io） | 更新 DB + sync | ✅ |
| 暂停/恢复 | 更新 DB + sync（失败回滚） | ✅ |
| 环境变量 | 仅更新 DB | ❌ |
| Docker 镜像 | 仅更新 DB | ❌ |
| 名称/描述 | 仅更新 DB | ❌ |

⚠️ **推断边界**：sync 后各类配置变更对运行中服务器的实际影响（如"资源限制立即调整 cgroup"），属于 Wings 端实现行为。Panel 代码能证实的是：Panel 对资源限制类变更调用了 sync，对环境变量类变更没有调用。注释中的 "apply resource modifications on the fly" 表明了开发者的设计意图，但不是对 Wings 行为的代码证明。

### 13.4 容错设计

✅ **已证实**：
- `EggConfigurationService::replacePlaceholders()` 对无效配置跳过处理（`EggConfigurationService.php:90-92`），防止单个 Egg 错误导致 Wings boot 崩溃
- `BuildModificationService` sync 失败仅记录日志，不回滚数据库
- `SuspensionService` sync 失败回滚暂停状态，行为更严格

### 13.5 环境变量键名冲突的防护

✅ **已证实**：`EggVariable::RESERVED_ENV_NAMES` 禁止使用 `SERVER_MEMORY`、`SERVER_IP`、`SERVER_PORT`、`STARTUP` 等保留名作为 Egg 变量名，防止与内置映射和系统变量冲突。

⚠️ **推断边界**：此防护仅限于 Egg 变量创建时的验证。`config/pterodactyl.php` 中 `environment_variables` 配置的键名与 Egg 变量名冲突时，配置文件的值会覆盖 Egg 变量值（因为 `put` 顺序靠后），这个行为是代码可证实的，但 Panel 没有额外的冲突检测机制。

---

## 14. 核心文件索引

| 文件 | 主要职责 |
|------|----------|
| `app/Models/Egg.php` | Egg 数据模型，配置继承访问器 |
| `app/Models/EggVariable.php` | 变量模板定义，保留名防护 |
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
| `app/Services/Servers/SuspensionService.php` | 暂停/恢复 + sync（失败回滚） |
| `app/Http/Controllers/Api/Remote/Servers/ServerDetailsController.php` | Wings 配置拉取 API + 服务器列表 |
| `app/Http/Controllers/Api/Remote/EggInstallController.php` | Wings 安装脚本拉取 API |
| `app/Http/Controllers/Api/Client/Servers/StartupController.php` | 前端启动变量修改 API（不调用 sync） |
| `app/Http/Controllers/Api/Client/Servers/SettingsController.php` | Docker 镜像修改 API（不调用 sync） |
| `app/Http/Middleware/Api/Daemon/DaemonAuthenticate.php` | Wings → Panel 鉴权中间件 |
| `app/Repositories/Wings/DaemonServerRepository.php` | Panel → Wings 服务器管理 API |
| `app/Repositories/Wings/DaemonPowerRepository.php` | Panel → Wings 电源操作 API |
