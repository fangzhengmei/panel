# Pterodactyl Panel 启动变量校验、覆盖优先级与下发流程

本文档通过代码走查，系统梳理 Pterodactyl Panel 中游戏服启动变量从 Egg 模板定义、管理员与用户分别可配置范围、校验规则、覆盖优先级，到最终下发至 Wings 守护进程并参与启动命令组装的完整代码走向。

---

## 1. 数据模型：三层存储结构

启动变量在数据库中通过两张表、三层结构存储：

### 1.1 `egg_variables` — Egg 模板层变量定义

模型：[EggVariable.php](file:///d:/fz/0508-3/solo-dogfeeding/code/202-panel/app/Models/EggVariable.php)

| 字段 | 说明 |
|------|------|
| `name` | 变量显示名 |
| `description` | 变量描述 |
| `env_variable` | 环境变量名（正则校验 `^[\w]{1,191}$`） |
| `default_value` | 默认值，当服务器未设置该变量时的兜底值 |
| `user_viewable` | 用户是否可见（布尔） |
| `user_editable` | 用户是否可编辑（布尔） |
| `rules` | Laravel Validation 规则字符串（如 `required\|string\|max:20`） |
| `required` | （动态属性）由 `rules` 中是否包含 `required` 解析得出 |

**保留变量名禁止使用**（见 [EggVariable.php#L43](file:///d:/fz/0508-3/solo-dogfeeding/code/202-panel/app/Models/EggVariable.php#L43)）：
```
SERVER_MEMORY, SERVER_IP, SERVER_PORT, ENV, HOME, USER, STARTUP, SERVER_UUID, UUID
```

这些由 Panel 内置注入，不允许 Egg 自定义。

### 1.2 `server_variables` — 服务器实例层变量值

模型：[ServerVariable.php](file:///d:/fz/0508-3/solo-dogfeeding/code/202-panel/app/Models/ServerVariable.php)

| 字段 | 说明 |
|------|------|
| `server_id` | 关联服务器 |
| `variable_id` | 关联 `egg_variables.id` |
| `variable_value` | 该服务器上此变量的实际值 |

关键关系：`EggVariable 1:N ServerVariable`

### 1.3 `servers.startup` — 服务器启动命令模板

模型 [Server.php](file:///d:/fz/0508-3/solo-dogfeeding/code/202-panel/app/Models/Server.php) 中 `startup` 字段存启动命令字符串，形如：
```
java -Xms128M -jar {{SERVER_JARFILE}}
```
其中 `{{VAR_NAME}}` 占位符由变量值替换。

### 1.4 Server Model 的变量关联

[Server.php#L288-L301](file:///d:/fz/0508-3/solo-dogfeeding/code/202-panel/app/Models/Server.php#L288-L301) 定义了 `variables()` 关系：

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

通过 `LEFT JOIN` 直接把 `server_variables.variable_value` 挂载为 `server_value` 属性，**一次查询即可同时拿到 Egg 定义和服务器覆盖值**。

---

## 2. Egg 模板变量定义

### 2.1 Egg JSON 结构

示例：[egg-paper.json](file:///d:/fz/0508-3/solo-dogfeeding/code/202-panel/database/Seeders/eggs/minecraft/egg-paper.json)

```json
{
  "variables": [
    {
      "name": "Server Jar File",
      "description": "The name of the server jarfile to run the server with.",
      "env_variable": "SERVER_JARFILE",
      "default_value": "server.jar",
      "user_viewable": true,
      "user_editable": true,
      "rules": "required|regex:/^([\\w\\d._-]+)(\\.jar)$/",
      "field_type": "text"
    }
  ]
}
```

### 2.2 变量创建服务

[VariableCreationService.php](file:///d:/fz/0508-3/solo-dogfeeding/code/202-panel/app/Services/Eggs/Variables/VariableCreationService.php)

- 检查 `env_variable` 是否命中保留名，命中抛 `ReservedVariableNameException`
- 通过 `ValidatesValidationRules` trait 校验 `rules` 字符串本身是合法的 Laravel 验证规则（用假数据跑一次 `->make()->fails()`，捕获 `BadMethodCallException`）
- `user_viewable` / `user_editable` 由 `options` 数组是否包含对应字符串推导

### 2.3 变量更新服务

[VariableUpdateService.php](file:///d:/fz/0508-3/solo-dogfeeding/code/202-panel/app/Services/Eggs/Variables/VariableUpdateService.php)

额外检查：同 Egg 下 `env_variable` 必须唯一。

---

## 3. 权限边界：管理员 vs 普通用户

### 3.1 权限控制 Trait

[HasUserLevels.php](file:///d:/fz/0508-3/solo-dogfeeding/code/202-panel/app/Traits/Services/HasUserLevels.php)

```php
trait HasUserLevels
{
    private int $userLevel = User::USER_LEVEL_USER; // 默认普通用户
    public function setUserLevel(int $level): self { ... }
    public function isUserLevel(int $level): bool { ... }
}
```

服务在调用前由控制器调用 `setUserLevel()` 设定身份。

### 3.2 变量校验服务中的权限过滤

[VariableValidatorService.php#L28-L38](file:///d:/fz/0508-3/solo-dogfeeding/code/202-panel/app/Services/Servers/VariableValidatorService.php#L28-L38)

```php
public function handle(int $egg, array $fields = []): Collection
{
    $query = EggVariable::query()->where('egg_id', $egg);
    if (!$this->isUserLevel(User::USER_LEVEL_ADMIN)) {
        // 非管理员：只校验 user_editable=true 且 user_viewable=true 的变量
        $query = $query->where('user_editable', true)->where('user_viewable', true);
    }
    $variables = $query->get();
    // ...
}
```

**关键结论**：
- **管理员**：可校验（即可写入）所有 Egg 变量，无论是否标记 user_editable / user_viewable
- **普通用户**：只能操作同时满足 `user_viewable=true` AND `user_editable=true` 的变量

### 3.3 客户端 API 控制器

[StartupController.php（客户端）](file:///d:/fz/0508-3/solo-dogfeeding/code/202-panel/app/Http/Controllers/Api/Client/Servers/StartupController.php)

`update()` 方法中在业务层再次做了三道闸：
1. 变量必须存在且 `user_viewable = true`（[L57-L58](file:///d:/fz/0508-3/solo-dogfeeding/code/202-panel/app/Http/Controllers/Api/Client/Servers/StartupController.php#L57-L58)）
2. 变量必须 `user_editable = true`（[L59-L61](file:///d:/fz/0508-3/solo-dogfeeding/code/202-panel/app/Http/Controllers/Api/Client/Servers/StartupController.php#L59-L61)）
3. 路由层要求 `Permission::ACTION_STARTUP_UPDATE`（[UpdateStartupVariableRequest.php#L12](file:///d:/fz/0508-3/solo-dogfeeding/code/202-panel/app/Http/Requests/Api/Client/Servers/Startup/UpdateStartupVariableRequest.php#L12)）

### 3.4 管理员独有的修改项

[StartupModificationService.php#L50-L87](file:///d:/fz/0508-3/solo-dogfeeding/code/202-panel/app/Services/Servers/StartupModificationService.php#L50-L87)

只有 `isUserLevel(User::USER_LEVEL_ADMIN)` 为真时，`updateAdministrativeSettings()` 才执行，允许：
- 修改 `startup` 启动命令模板本身
- 修改 `docker_image`（容器镜像）
- 修改 `skip_scripts`
- 切换 `egg_id`（即更换服务器的 Egg）

普通用户无法触及以上字段。

---

## 4. 变量校验规则

### 4.1 校验入口

[VariableValidatorService.php#L40-L50](file:///d:/fz/0508-3/solo-dogfeeding/code/202-panel/app/Services/Servers/VariableValidatorService.php#L40-L50)

```php
foreach ($variables as $variable) {
    $data['environment'][$variable->env_variable] = array_get($fields, $variable->env_variable);
    $rules['environment.' . $variable->env_variable] = $variable->rules;
    $customAttributes['environment.' . $variable->env_variable] = trans('...', ['env' => $variable->name]);
}
$validator = $this->validator->make($data, $rules, [], $customAttributes);
if ($validator->fails()) {
    throw new ValidationException($validator);
}
```

每个 EggVariable 的 `rules` 字段直接作为 Laravel 验证规则使用，支持全部 Laravel 内置规则（`required`、`string`、`regex`、`max`、`in`、`numeric` 等）。

### 4.2 校验规则的合法性保障

Egg 变量创建/更新时，[ValidatesValidationRules.php](file:///d:/fz/0508-3/solo-dogfeeding/code/202-panel/app/Traits/Services/ValidatesValidationRules.php) 会用一个假值 `['__TEST' => 'test']` 跑一遍验证，若规则名不存在则抛出 `BadValidationRuleException`，阻止非法规则入库。

### 4.3 单变量更新的额外校验

客户端 API `StartupController@update` 针对单变量更新单独再跑一次：
```php
$this->validate($request, ['value' => $variable->rules]);
```
（[StartupController.php#L66](file:///d:/fz/0508-3/solo-dogfeeding/code/202-panel/app/Http/Controllers/Api/Client/Servers/StartupController.php#L66)）

---

## 5. 变量覆盖优先级（从低到高）

这是运维最关心的"谁覆盖谁"问题。贯穿多个服务，统一使用同一个表达式：

```php
$variable->server_value ?? $variable->default_value
```

### 5.1 优先级总表

| 优先级 | 来源 | 代码位置 | 说明 |
|--------|------|----------|------|
| 1（最低） | `egg_variables.default_value` | 模板定义 | Egg 自带默认，所有服务器共用 |
| 2 | `server_variables.variable_value` | 服务器实例覆盖 | 每台服独立，在创建或修改启动参数时写入 |
| 3 | Panel 内置环境映射（`STARTUP`、`P_SERVER_UUID` 等） | [EnvironmentService.php#L67-L72](file:///d:/fz/0508-3/solo-dogfeeding/code/202-panel/app/Services/Servers/EnvironmentService.php#L67-L72) | 固定键，如用 `server.startup`、`server.uuid` 等属性填充 |
| 4 | 配置文件 `pterodactyl.environment_variables` | [EnvironmentService.php#L47-L52](file:///d:/fz/0508-3/solo-dogfeeding/code/202-panel/app/Services/Servers/EnvironmentService.php#L47-L52) | 从 `config/pterodactyl.php` 读取，支持闭包或 object_get 路径 |
| 5（最高） | 运行时动态注册 | [EnvironmentService.php#L55-L57](file:///d:/fz/0508-3/solo-dogfeeding/code/202-panel/app/Services/Servers/EnvironmentService.php#L55-L57) | 通过 `EnvironmentService::setEnvironmentKey()` 注入的闭包 |

**运维常见"被重置"根因**：
- 当 `server_variables` 中没有对应记录（即管理员/用户未在该服上显式设置），代码走 `??` 右边，取 `default_value`
- 若管理员在 Egg 模板层面修改了 `default_value`，所有未显式覆盖该变量的服务器下次启动都会感知到新默认值 → 表现为"被重置"
- 变量同步/重建时，如果 `server_variables` 行丢失（如导入导出流程问题），也会退回 Egg 默认值

### 5.2 服务器创建时的变量写入

[ServerCreationService.php#L76-L78](file:///d:/fz/0508-3/solo-dogfeeding/code/202-panel/app/Services/Servers/ServerCreationService.php#L76-L78) 创建服务器时以管理员身份调用：
```php
$eggVariableData = $this->validatorService
    ->setUserLevel(User::USER_LEVEL_ADMIN)
    ->handle(Arr::get($data, 'egg_id'), Arr::get($data, 'environment', []));
```

随后 [storeEggVariables()](file:///d:/fz/0508-3/solo-dogfeeding/code/202-panel/app/Services/Servers/ServerCreationService.php#L188-L201) 把校验结果批量 `insert` 进 `server_variables`，若用户未传某个变量，`value` 为 `null`，入库存 `''` 空字符串。

### 5.3 变量修改时的写入

[StartupModificationService.php#L39-L47](file:///d:/fz/0508-3/solo-dogfeeding/code/202-panel/app/Services/Servers/StartupModificationService.php#L39-L47)

```php
foreach ($results as $result) {
    ServerVariable::query()->updateOrCreate(
        ['server_id' => $server->id, 'variable_id' => $result->id],
        ['variable_value' => $result->value ?? '']
    );
}
```

用 `updateOrCreate`：存在则更新，不存在则插入。

---

## 6. 启动命令组装

### 6.1 StartupCommandService

[StartupCommandService.php](file:///d:/fz/0508-3/solo-dogfeeding/code/202-panel/app/Services/Servers/StartupCommandService.php)

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

要点：
- 内置三个变量 `SERVER_MEMORY` / `SERVER_IP` / `SERVER_PORT` 优先替换，来源分别是服务器资源配置和主分配
- 遍历 `$server->variables`（Egg 定义 LEFT JOIN 服务器值）
- 如果变量不可见或要求隐藏，显示 `[hidden]` 而不泄露真实值
- 对可见变量，同样遵循 `server_value ?? default_value` 规则
- 最终对 `server.startup` 字符串做 `str_replace`

### 6.2 EnvironmentService（下发给 Wings 的环境变量）

[EnvironmentService.php](file:///d:/fz/0508-3/solo-dogfeeding/code/202-panel/app/Services/Servers/EnvironmentService.php)

```php
public function handle(Server $server): array
{
    // Step 1: Egg 变量（带服务器覆盖）
    $variables = $server->variables->toBase()->mapWithKeys(function (EggVariable $variable) {
        return [$variable->env_variable => $variable->server_value ?? $variable->default_value];
    });

    // Step 2: 内置映射
    foreach ($this->getEnvironmentMappings() as $key => $object) {
        $variables->put($key, object_get($server, $object));
    }
    // getEnvironmentMappings() 返回：
    //   STARTUP        => server.startup
    //   P_SERVER_LOCATION => server.location.short
    //   P_SERVER_UUID  => server.uuid

    // Step 3: 配置文件定义
    foreach (config('pterodactyl.environment_variables', []) as $key => $object) {
        $variables->put($key, is_callable($object) ? call_user_func($object, $server) : object_get($server, $object));
    }

    // Step 4: 运行时动态注册（优先级最高）
    foreach ($this->additional as $key => $closure) {
        $variables->put($key, call_user_func($closure, $server));
    }

    return $variables->toArray();
}
```

注意 `$variables->put()` 是**相同 key 直接覆盖**，因此后执行的步骤优先级更高。

---

## 7. 下发到 Wings 守护进程

### 7.1 配置结构组装

[ServerConfigurationStructureService.php](file:///d:/fz/0508-3/solo-dogfeeding/code/202-panel/app/Services/Servers/ServerConfigurationStructureService.php)

`returnCurrentFormat()` 返回给 Wings 的结构中：

```php
[
    'uuid' => $server->uuid,
    'suspended' => $server->isSuspended(),
    'environment' => $this->environment->handle($server),  // EnvironmentService 的输出
    'invocation'  => $server->startup,                     // 原始启动命令（含 {{...}} 占位符，Wings 自己也会替换）
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
    'allocations' => [ /* ... */ ],
    'mounts' => [ /* ... */ ],
    'egg' => [ /* ... */ ],
]
```

同时还有 `returnLegacyFormat()` 兼容旧版 Wings。

### 7.2 Wings 拉取配置的入口

[ServerDetailsController.php](file:///d:/fz/0508-3/solo-dogfeeding/code/202-panel/app/Http/Controllers/Api/Remote/Servers/ServerDetailsController.php)

```php
public function __invoke(Request $request, string $uuid): JsonResponse
{
    // 校验节点归属...
    return new JsonResponse([
        'settings' => $this->configurationStructureService->handle($server),
        'process_configuration' => $this->eggConfigurationService->handle($server),
    ]);
}
```

路由见 `routes/api-remote.php`。Wings 在以下场景主动拉取：
- 自身启动时批量拉该节点所有服配置（`list()` 方法，分页 50）
- 服务器启动前
- 收到 Panel 的 `sync` 调用后

### 7.3 Panel 主动触发 Wings 同步

[DaemonServerRepository.php#L63-L72](file:///d:/fz/0508-3/solo-dogfeeding/code/202-panel/app/Repositories/Wings/DaemonServerRepository.php#L63-L72)

```php
public function sync(): void
{
    $this->getHttpClient()->post("/api/servers/{$this->server->uuid}/sync");
}
```

**重要：变量变更默认不自动 sync**。只有 [BuildModificationService.php#L66-L72](file:///d:/fz/0508-3/solo-dogfeeding/code/202-panel/app/Services/Servers/BuildModificationService.php#L66-L72)（构建参数变更：内存/CPU/磁盘/端口等）会显式调用 `sync()`，而 `StartupModificationService`（启动变量变更）**不会**调用 sync。

这解释了为什么改了启动变量后**需要手动重启服务器**才生效：
- `StartupModificationService` 只改 Panel 数据库
- Wings 下次启动服务器时才会重新调用 `/api/remote/servers/{uuid}` 拉最新配置
- 如果服务器正在运行，容器内的环境变量和启动命令进程不会热更新

### 7.4 Egg 配置解析（config_files / config_stop 等）

[EggConfigurationService.php](file:///d:/fz/0508-3/solo-dogfeeding/code/202-panel/app/Services/Eggs/EggConfigurationService.php)

负责处理 Egg JSON 中 `config` 块：
- `config.startup`：done 字符串、strip_ansi 等启动检测配置
- `config.stop`：停止命令（以 `^` 开头转为 kill signal，如 `^C` → SIGINT）
- `config.files`：配置文件解析器（properties/yaml/json/ini 等）和占位符替换规则

Egg 继承机制见 [Egg.php](file:///d:/fz/0508-3/solo-dogfeeding/code/202-panel/app/Models/Egg.php) 中 `getInherit*Attribute` 访问器：本 Egg 字段非空则用自己的，否则从 `config_from` 指向的父 Egg 读取。

---

## 8. 变量变更与服务器重启的关系

### 8.1 变更路径汇总

| 变更操作 | 修改服务 | 是否触发 Wings sync | 是否需要重启生效 |
|----------|----------|---------------------|------------------|
| 修改服务器构建参数（内存/CPU/端口） | BuildModificationService | ✅ 是（[L68](file:///d:/fz/0508-3/solo-dogfeeding/code/202-panel/app/Services/Servers/BuildModificationService.php#L68)） | 部分可热更（如 cgroup 限制），部分需重启 |
| 修改启动变量 / 启动命令 / Docker 镜像 | StartupModificationService | ❌ 否 | ✅ 必须重启服务器 |
| 修改 Egg 模板 default_value | VariableUpdateService | ❌ 否 | ✅ 必须重启，且服务器未显式覆盖该变量才生效 |
| 修改 Egg 模板 rules/env_variable 名 | VariableUpdateService | ❌ 否 | ⚠️ 需视情况；若改了 env_variable 名，旧 server_variables 行的 variable_id 仍指向新定义，可能不会匹配新占位符 |
| 服务器（重新）安装 | ReinstallServerService | ❌ 由 install 流程本身处理 | ✅ 安装完后首次启动即生效 |
| 节点 / Wings 重启 | — | ✅ Wings 启动时批量拉所有服 | — |

### 8.2 为什么改了 Egg 默认值后老服务器"没变化"

因为 `server_variables` 中已经存在该 `variable_id` 的记录（哪怕值是空字符串 `''`），此时：
```php
$variable->server_value ?? $variable->default_value
```
取的是 `server_value`（`''` 不是 null，`??` 不触发兜底）。只有在 `server_variables` 表中完全没有该服务器+该变量的行时才会退回 Egg 默认值。

### 8.3 Activity Log 审计

客户端单变量更新 [StartupController.php#L80-L89](file:///d:/fz/0508-3/solo-dogfeeding/code/202-panel/app/Http/Controllers/Api/Client/Servers/StartupController.php#L80-L89) 对比新旧值：

```php
if ($original !== $request->input('value')) {
    Activity::event('server:startup.edit')
        ->subject($variable)
        ->property(['variable' => $variable->env_variable, 'old' => $original, 'new' => $request->input('value') ?? ''])
        ->log();
}
```

管理员批量修改走 `StartupModificationService` 本身不记 Activity，仅由 `ServerObserver` 触发的 `Server\Updating` / `Updated` 事件（若有对应监听）。

---

## 9. 排障速查

**症状：启动参数被"强行重置"**

排查顺序：
1. 查 `server_variables` 表中该 `server_id` + 对应 `variable_id` 是否有记录？值是什么？
2. 查 `egg_variables.default_value` 近期是否被 Egg 维护者更新？
3. 查 `servers.startup` 字段是否被管理员切换 Egg 或修改启动命令模板？
4. 查 Activity Log 中 `server:startup.edit` 事件，看谁何时改了值
5. 确认该变量的 `user_viewable` / `user_editable` 标记，确认用户侧是否真的能改到
6. 如果 Wings 侧显示的是旧值，确认服务器是否重启过（Wings 只在启动前拉配置）

**核心代码索引：**

| 关注点 | 文件 |
|--------|------|
| 变量模型 | [EggVariable.php](file:///d:/fz/0508-3/solo-dogfeeding/code/202-panel/app/Models/EggVariable.php)、[ServerVariable.php](file:///d:/fz/0508-3/solo-dogfeeding/code/202-panel/app/Models/ServerVariable.php) |
| 服务器变量关联 | [Server.php#L288-L301](file:///d:/fz/0508-3/solo-dogfeeding/code/202-panel/app/Models/Server.php#L288-L301) |
| 变量校验 | [VariableValidatorService.php](file:///d:/fz/0508-3/solo-dogfeeding/code/202-panel/app/Services/Servers/VariableValidatorService.php) |
| 启动变量修改 | [StartupModificationService.php](file:///d:/fz/0508-3/solo-dogfeeding/code/202-panel/app/Services/Servers/StartupModificationService.php) |
| 启动命令组装 | [StartupCommandService.php](file:///d:/fz/0508-3/solo-dogfeeding/code/202-panel/app/Services/Servers/StartupCommandService.php) |
| 环境变量生成（下发用） | [EnvironmentService.php](file:///d:/fz/0508-3/solo-dogfeeding/code/202-panel/app/Services/Servers/EnvironmentService.php) |
| Wings 配置结构 | [ServerConfigurationStructureService.php](file:///d:/fz/0508-3/solo-dogfeeding/code/202-panel/app/Services/Servers/ServerConfigurationStructureService.php) |
| Wings 拉取接口 | [ServerDetailsController.php](file:///d:/fz/0508-3/solo-dogfeeding/code/202-panel/app/Http/Controllers/Api/Remote/Servers/ServerDetailsController.php) |
| Panel→Wings sync 调用 | [DaemonServerRepository.php#L63-L72](file:///d:/fz/0508-3/solo-dogfeeding/code/202-panel/app/Repositories/Wings/DaemonServerRepository.php#L63-L72) |
| 客户端更新接口 | [StartupController.php（客户端）](file:///d:/fz/0508-3/solo-dogfeeding/code/202-panel/app/Http/Controllers/Api/Client/Servers/StartupController.php) |
| 管理员更新接口 | [StartupController.php（应用 API）](file:///d:/fz/0508-3/solo-dogfeeding/code/202-panel/app/Http/Controllers/Api/Application/Servers/StartupController.php)、[ServersController@saveStartup](file:///d:/fz/0508-3/solo-dogfeeding/code/202-panel/app/Http/Controllers/Admin/ServersController.php#L178-L197) |
