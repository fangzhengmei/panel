# Egg/Nest 启动配置注入链路分析

## 1. 整体架构概览

Pterodactyl 面板通过 **Nest（分类）** → **Egg（游戏模板）** → **Server（服务器实例）** 的层级结构实现游戏服务器的配置管理。整个启动配置注入链路涉及模板加载、变量验证、环境变量注入、启动命令生成以及与 Wings 守护进程的通信。

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

## 2. 数据模型与配置存储

### 2.1 Nest 模型 (`app/Models/Nest.php`)

Nest 是 Egg 的上层分类，主要用于组织管理不同类型的游戏模板。

```php
// 核心关系
public function eggs(): HasMany       // 一对多关联 Egg
public function servers(): HasMany    // 一对多关联 Server
```

### 2.2 Egg 模型 (`app/Models/Egg.php`)

Egg 是整个配置系统的核心模板载体，存储了游戏服务器的所有配置模板信息。

**核心字段**：

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

**配置继承机制**：Egg 支持通过 `config_from` 和 `copy_script_from` 字段实现配置继承，通过访问器自动解析：

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

### 2.3 EggVariable 模型 (`app/Models/EggVariable.php`)

定义 Egg 的环境变量模板，每个 Server 创建时会基于此创建实例化变量。

**核心字段**：

| 字段 | 类型 | 说明 |
|------|------|------|
| `env_variable` | string | 环境变量名（如 `SERVER_JARFILE`） |
| `default_value` | string | 默认值 |
| `user_viewable` | bool | 用户是否可见 |
| `user_editable` | bool | 用户是否可编辑 |
| `rules` | string | Laravel 验证规则（如 `required|string|max:20`） |

**保留变量名**（不可使用）：
```php
// app/Models/EggVariable.php:43
public const RESERVED_ENV_NAMES = 'SERVER_MEMORY,SERVER_IP,SERVER_PORT,ENV,HOME,USER,STARTUP,SERVER_UUID,UUID';
```

### 2.4 ServerVariable 模型 (`app/Models/ServerVariable.php`)

存储每个 Server 实例的实际变量值，与 EggVariable 通过 `variable_id` 关联。

```php
// 核心字段：server_id, variable_id, variable_value
```

---

## 3. 模板加载流程

### 3.1 Egg JSON 模板格式

以 Minecraft Paper Egg 为例 (`database/Seeders/eggs/minecraft/egg-paper.json`)：

```json
{
    "meta": {
        "version": "PTDL_v2",
        "update_url": null
    },
    "name": "Paper",
    "startup": "java -Xms128M -XX:MaxRAMPercentage=95.0 -jar {{SERVER_JARFILE}}",
    "config": {
        "files": "{\"server.properties\":{\"parser\":\"properties\",\"find\":{\"server-ip\":\"0.0.0.0\",\"server-port\":\"{{server.build.default.port}}\"}}}",
        "startup": "{\"done\": \")! For help, type \"}",
        "logs": "{}",
        "stop": "stop"
    },
    "scripts": {
        "installation": {
            "script": "#!/bin/ash\n# 安装脚本...",
            "container": "ghcr.io/pterodactyl/installers:alpine",
            "entrypoint": "ash"
        }
    },
    "variables": [
        {
            "name": "Server Jar File",
            "env_variable": "SERVER_JARFILE",
            "default_value": "server.jar",
            "user_viewable": true,
            "user_editable": true,
            "rules": "required|regex:/^([\\w\\d._-]+)(\\.jar)$/"
        }
    ]
}
```

### 3.2 EggParserService (`app/Services/Eggs/EggParserService.php`)

负责解析导入的 Egg JSON 文件，支持 PTDL_v1 和 PTDL_v2 两种格式。

**核心流程**：

```php
// app/Services/Eggs/EggParserService.php:19-32
public function handle(UploadedFile $file): array
{
    $parsed = json_decode($file->openFile()->fread($file->getSize()), true, 512, JSON_THROW_ON_ERROR);
    return $this->convertToV2($parsed);
}
```

**格式转换** (`convertToV2`)：
- 将旧版单镜像字段 `image` 转换为新版 `docker_images` 数组
- 为变量添加 `field_type` 字段（默认为 `text`）

**填充模型**：

```php
// app/Services/Eggs/EggParserService.php:37-56
public function fillFromParsed(Egg $model, array $parsed): Egg
{
    return $model->forceFill([
        'name' => Arr::get($parsed, 'name'),
        'config_files' => Arr::get($parsed, 'config.files'),
        'config_startup' => Arr::get($parsed, 'config.startup'),
        'startup' => Arr::get($parsed, 'startup'),
        'script_install' => Arr::get($parsed, 'scripts.installation.script'),
        // ... 其他字段
    ]);
}
```

---

## 4. 变量验证机制

### 4.1 VariableValidatorService (`app/Services/Servers/VariableValidatorService.php`)

在服务器创建或变量更新时验证所有变量值是否符合 EggVariable 中定义的规则。

```php
// app/Services/Servers/VariableValidatorService.php:28-59
public function handle(int $egg, array $fields = []): Collection
{
    $query = EggVariable::query()->where('egg_id', $egg);
    
    // 非管理员只能验证用户可编辑、可见的变量
    if (!$this->isUserLevel(User::USER_LEVEL_ADMIN)) {
        $query = $query->where('user_editable', true)->where('user_viewable', true);
    }
    
    $variables = $query->get();
    
    // 构建验证规则
    foreach ($variables as $variable) {
        $data['environment'][$variable->env_variable] = array_get($fields, $variable->env_variable);
        $rules['environment.' . $variable->env_variable] = $variable->rules;
    }
    
    $validator = $this->validator->make($data, $rules, [], $customAttributes);
    if ($validator->fails()) {
        throw new ValidationException($validator);
    }
    
    return Collection::make($variables)->map(function ($item) use ($fields) {
        return (object) [
            'id' => $item->id,
            'key' => $item->env_variable,
            'value' => $fields[$item->env_variable] ?? null,
        ];
    });
}
```

---

## 5. 环境变量注入

### 5.1 EnvironmentService (`app/Services/Servers/EnvironmentService.php`)

负责生成服务器运行时的完整环境变量数组，供 Wings 守护进程使用。

**变量来源优先级**（从低到高）：

1. **EggVariable 默认值** → 来自 `egg_variables.default_value`
2. **ServerVariable 实例值** → 来自 `server_variables.variable_value`
3. **内置映射变量** → `getEnvironmentMappings()` 定义
4. **配置文件变量** → `config/pterodactyl.php` 中 `environment_variables`
5. **动态注入变量** → 通过 `setEnvironmentKey()` 运行时注入

```php
// app/Services/Servers/EnvironmentService.php:33-60
public function handle(Server $server): array
{
    // 1. 从 EggVariable 和 ServerVariable 加载
    $variables = $server->variables->toBase()->mapWithKeys(function (EggVariable $variable) {
        return [$variable->env_variable => $variable->server_value ?? $variable->default_value];
    });

    // 2. 内置映射变量
    foreach ($this->getEnvironmentMappings() as $key => $object) {
        $variables->put($key, object_get($server, $object));
    }

    // 3. 配置文件定义的变量
    foreach (config('pterodactyl.environment_variables', []) as $key => $object) {
        $variables->put($key, is_callable($object) ? call_user_func($object, $server) : object_get($server, $object));
    }

    // 4. 动态注入变量
    foreach ($this->additional as $key => $closure) {
        $variables->put($key, call_user_func($closure, $server));
    }

    return $variables->toArray();
}

// 内置映射变量
private function getEnvironmentMappings(): array
{
    return [
        'STARTUP' => 'startup',
        'P_SERVER_LOCATION' => 'location.short',
        'P_SERVER_UUID' => 'uuid',
    ];
}
```

**配置文件变量** (`config/pterodactyl.php:150-152`)：
```php
'environment_variables' => [
    'P_SERVER_ALLOCATION_LIMIT' => 'allocation_limit',
],
```

---

## 6. 启动命令生成

### 6.1 StartupCommandService (`app/Services/Servers/StartupCommandService.php`)

将 Egg 的启动命令模板中的变量占位符替换为实际值，生成最终的启动命令。

```php
// app/Services/Servers/StartupCommandService.php:12-23
public function handle(Server $server, bool $hideAllValues = false): string
{
    // 内置变量替换
    $find = ['{{SERVER_MEMORY}}', '{{SERVER_IP}}', '{{SERVER_PORT}}'];
    $replace = [$server->memory, $server->allocation->ip, $server->allocation->port];

    // Egg 定义的变量替换
    foreach ($server->variables as $variable) {
        $find[] = '{{' . $variable->env_variable . '}}';
        $replace[] = ($variable->user_viewable && !$hideAllValues) 
            ? ($variable->server_value ?? $variable->default_value) 
            : '[hidden]';
    }

    return str_replace($find, $replace, $server->startup);
}
```

**示例**：
- 模板：`java -Xms128M -XX:MaxRAMPercentage=95.0 -jar {{SERVER_JARFILE}}`
- 变量：`SERVER_JARFILE=server.jar`, `SERVER_MEMORY=1024`, `SERVER_PORT=25565`
- 结果：`java -Xms128M -XX:MaxRAMPercentage=95.0 -jar server.jar`

---

## 7. 配置文件解析与占位符替换

### 7.1 EggConfigurationService (`app/Services/Eggs/EggConfigurationService.php`)

负责将 Egg 配置模板中的占位符替换为实际服务器配置值，生成 Wings 可直接使用的配置。

**核心流程**：

```php
// app/Services/Eggs/EggConfigurationService.php:22-34
public function handle(Server $server): array
{
    $configs = $this->replacePlaceholders(
        $server,
        json_decode($server->egg->inherit_config_files)
    );

    return [
        'startup' => $this->convertStartupToNewFormat(json_decode($server->egg->inherit_config_startup, true)),
        'stop' => $this->convertStopToNewFormat($server->egg->inherit_config_stop),
        'configs' => $configs,
    ];
}
```

### 7.2 占位符替换机制 (`replacePlaceholders`)

支持三种占位符格式：
- `{{server.X}}` → 服务器属性（如 `{{server.build.default.port}}`）
- `{{env.X}}` → 环境变量（如 `{{env.SERVER_JARFILE}}`）
- `{{config.X}}` → Wings 配置（不替换，直接传给 Wings）

```php
// app/Services/Eggs/EggConfigurationService.php:155-204
protected function matchAndReplaceKeys(mixed $value, array $structure): mixed
{
    preg_match_all('/{{(?<key>[\w.-]*)}}/', $value, $matches);

    foreach ($matches['key'] as $key) {
        // 只处理 server., env., config. 开头的占位符
        if (!Str::startsWith($key, ['server.', 'env.', 'config.'])) {
            continue;
        }

        // 替换旧版占位符
        $value = $this->replaceLegacyModifiers($key, $value);

        // 替换 {{server.X}} 占位符
        if (Str::startsWith($key, 'server.')) {
            $plucked = Arr::get($structure, preg_replace('/^server\./', '', $key), '');
            $value = str_replace("{{{$key}}}", $plucked, $value);
            continue;
        }

        // 替换 {{env.X}} 占位符
        $plucked = Arr::get($structure, preg_replace('/^env\./', 'build.env.', $key), '');
        $value = str_replace("{{{$key}}}", $plucked, $value);
    }

    return $value;
}
```

**旧版兼容替换** (`replaceLegacyModifiers`)：
- `config.docker.interface` → `config.docker.network.interface`
- `env.SERVER_MEMORY` → `server.build.memory`
- `env.SERVER_IP` → `server.build.default.ip`
- `env.SERVER_PORT` → `server.build.default.port`

### 7.3 停止命令格式转换

```php
// app/Services/Eggs/EggConfigurationService.php:57-72
protected function convertStopToNewFormat(string $stop): array
{
    if (!Str::startsWith($stop, '^')) {
        return ['type' => 'command', 'value' => $stop];
    }

    $signal = substr($stop, 1);
    return ['type' => 'signal', 'value' => strtoupper($signal)];
}
```

- 普通命令：`stop` → `{type: 'command', value: 'stop'}`
- 信号命令：`^SIGINT` → `{type: 'signal', value: 'SIGINT'}`

---

## 8. 服务器配置结构生成

### 8.1 ServerConfigurationStructureService (`app/Services/Servers/ServerConfigurationStructureService.php`)

生成发送给 Wings 的完整服务器配置结构，包含所有运行时参数。

```php
// app/Services/Servers/ServerConfigurationStructureService.php:43-92
protected function returnCurrentFormat(Server $server): array
{
    return [
        'uuid' => $server->uuid,
        'meta' => [
            'name' => $server->name,
            'description' => $server->description,
        ],
        'suspended' => $server->isSuspended(),
        'environment' => $this->environment->handle($server),
        'invocation' => $server->startup,
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
        'container' => [
            'image' => $server->image,
            'requires_rebuild' => false,
        ],
        'allocations' => [
            'force_outgoing_ip' => $server->egg->force_outgoing_ip,
            'default' => [
                'ip' => $server->allocation->ip,
                'port' => $server->allocation->port,
            ],
            'mappings' => $server->getAllocationMappings(),
        ],
        'mounts' => $server->mounts->map(...),
        'egg' => [
            'id' => $server->egg->uuid,
            'file_denylist' => $server->egg->inherit_file_denylist,
        ],
    ];
}
```

---

## 9. 与 Wings 守护进程的通信

### 9.1 DaemonRepository 基类 (`app/Repositories/Wings/DaemonRepository.php`)

提供与 Wings 通信的基础 HTTP 客户端配置。

```php
// app/Repositories/Wings/DaemonRepository.php:49-64
public function getHttpClient(array $headers = []): Client
{
    return new Client([
        'verify' => $this->app->environment('production'),
        'base_uri' => $this->node->getConnectionAddress(),
        'timeout' => config('pterodactyl.guzzle.timeout'),
        'connect_timeout' => config('pterodactyl.guzzle.connect_timeout'),
        'headers' => array_merge($headers, [
            'Authorization' => 'Bearer ' . $this->node->getDecryptedKey(),
            'Accept' => 'application/json',
            'Content-Type' => 'application/json',
        ]),
    ]);
}
```

### 9.2 核心 API 调用

**创建服务器** (`DaemonServerRepository.php:42-56`)：
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

**同步配置** (`DaemonServerRepository.php:63-72`)：
```php
public function sync(): void
{
    $this->getHttpClient()->post("/api/servers/{$this->server->uuid}/sync");
}
```

**电源操作** (`DaemonPowerRepository.php:22-34`)：
```php
public function send(string $action): ResponseInterface
{
    return $this->getHttpClient()->post(
        sprintf('/api/servers/%s/power', $this->server->uuid),
        ['json' => ['action' => $action]]
    );
}
```

---

## 10. 完整调用链路

### 10.1 服务器创建流程

```
ServerCreationService::handle()
    │
    ├─▶ VariableValidatorService::handle()
    │     验证所有环境变量
    │
    ├─▶ 数据库事务
    │     ├─▶ createModel() - 创建 Server 记录
    │     ├─▶ storeAssignedAllocations() - 分配端口
    │     └─▶ storeEggVariables() - 存储 ServerVariable
    │
    └─▶ DaemonServerRepository::create()
          通知 Wings 创建服务器
          Wings 会回调 Panel 获取完整配置
```

### 10.2 Wings 获取配置流程

当 Wings 需要服务器配置时，Panel 通过以下流程生成：

```
Wings 请求 → Panel API → 
    │
    ├─▶ ServerConfigurationStructureService::handle()
    │     └─▶ EnvironmentService::handle()
    │           生成完整环境变量
    │
    ├─▶ EggConfigurationService::handle()
    │     ├─▶ convertStartupToNewFormat() - 启动检测配置
    │     ├─▶ convertStopToNewFormat() - 停止命令格式化
    │     └─▶ replacePlaceholders() - 配置文件占位符替换
    │           └─▶ matchAndReplaceKeys()
    │                 └─▶ iterate() - 递归遍历替换
    │
    └─▶ 返回完整配置给 Wings
```

### 10.3 配置变更同步流程

当服务器配置（如内存、端口）变更时：

```
BuildModificationService::handle()
    │
    ├─▶ 更新 Server 模型属性
    │
    └─▶ DaemonServerRepository::sync()
          通知 Wings 重新拉取配置
          Wings 会重新执行上面的配置获取流程
```

---

## 11. 关键设计要点

### 11.1 配置继承机制

Egg 通过 `config_from` 和 `copy_script_from` 实现配置复用，避免重复定义相同的配置规则。访问器模式确保上层代码无需关心继承关系。

### 11.2 变量替换的两层实现

1. **面板端替换**：`StartupCommandService` 和 `EggConfigurationService` 在面板端完成替换，减少 Wings 的复杂度
2. **Wings 端替换**：`{{config.X}}` 占位符留给 Wings 处理，用于需要动态获取的主机级配置

### 11.3 容错设计

在 `EggConfigurationService::replacePlaceholders()` 中对无效配置进行容错处理：
```php
// app/Services/Eggs/EggConfigurationService.php:90-92
if (!is_object($data) || !isset($data->find)) {
    continue;
}
```
避免单个 Egg 配置错误导致所有服务器无法启动。

### 11.4 配置同步的最终一致性

Wings 在服务器启动时总是从 Panel 拉取最新配置，因此即使 `sync()` 调用失败，下次启动时也会自动获取最新配置，保证最终一致性。

---

## 12. 核心文件索引

| 文件 | 主要职责 |
|------|----------|
| `app/Models/Egg.php` | Egg 数据模型，配置继承 |
| `app/Models/EggVariable.php` | 变量模板定义 |
| `app/Models/ServerVariable.php` | 实例变量存储 |
| `app/Services/Eggs/EggParserService.php` | Egg JSON 模板解析 |
| `app/Services/Eggs/EggConfigurationService.php` | Egg 配置占位符替换 |
| `app/Services/Servers/VariableValidatorService.php` | 变量值验证 |
| `app/Services/Servers/EnvironmentService.php` | 环境变量生成 |
| `app/Services/Servers/StartupCommandService.php` | 启动命令生成 |
| `app/Services/Servers/ServerConfigurationStructureService.php` | Wings 配置结构生成 |
| `app/Services/Servers/ServerCreationService.php` | 服务器创建流程编排 |
| `app/Repositories/Wings/DaemonServerRepository.php` | Wings 服务器管理 API |
| `app/Repositories/Wings/DaemonPowerRepository.php` | Wings 电源操作 API |
