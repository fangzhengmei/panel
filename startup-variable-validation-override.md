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

## 5. 变量覆盖优先级：三种状态与真实回退条件

贯穿多个服务的核心表达式是：

```php
$variable->server_value ?? $variable->default_value
```

但这句话**不能脱离 `$variable->server_value` 的三种状态**来理解。`??` 只对 `null` 生效，空字符串 `''` **不会**触发回退。结合创建流程、回填逻辑、保存逻辑的代码证据，三种状态的行为完全不同。

### 5.1 先破后立：服务器创建后，实例层已经为所有 Egg 变量写了行

[ServerCreationService.php#L76-L78](file:///d:/fz/0508-3/solo-dogfeeding/code/202-panel/app/Services/Servers/ServerCreationService.php#L76-L78) 创建服务器时以**管理员身份**调用 `VariableValidatorService`：

```php
$eggVariableData = $this->validatorService
    ->setUserLevel(User::USER_LEVEL_ADMIN)
    ->handle(Arr::get($data, 'egg_id'), Arr::get($data, 'environment', []));
```

VariableValidatorService 的行为（[VariableValidatorService.php#L28-L58](file:///d:/fz/0508-3/solo-dogfeeding/code/202-panel/app/Services/Servers/VariableValidatorService.php#L28-L58)）：
- 管理员身份不加 `user_editable/user_viewable` 过滤，**把 Egg 的全部 EggVariable 都查出来**（无论是否标记为用户可见/可编辑）
- 对每一个 EggVariable 生成 collection 条目，`value = $fields[$item->env_variable] ?? null`（用户没传就是 `null`）

随后 [storeEggVariables()](file:///d:/fz/0508-3/solo-dogfeeding/code/202-panel/app/Services/Servers/ServerCreationService.php#L188-L201)：

```php
foreach ($variables as $result) {
    $records[] = [
        'server_id' => $server->id,
        'variable_id' => $result->id,
        'variable_value' => $result->value ?? '',  // null → '' 入库
    ];
}
$this->serverVariableRepository->insert($records);
```

**结论 A：服务器创建完成那一刻，`server_variables` 表已经为 Egg 中**每一个** EggVariable 都插了一行。不存在"没显式设置就没有行"这种情况。** 没传值的变量值是 `''`（空字符串），不是 `null`。

### 5.2 三种数据库状态与 `??` 的真实行为

理解了创建流程，就只有**三种**可能状态：

| # | 状态 | `server_value`（LEFT JOIN 出来的） | `$server_value ?? $default_value` 结果 | 何时会发生 |
|---|------|-------------------------------------|---------------------------------------|------------|
| ① | **有记录，值非空**（如 `'1.20.1'`） | `'1.20.1'` | `'1.20.1'` | 用户或管理员在创建时或之后填了具体值并保存 |
| ② | **有记录，值是空字符串 `''`** | `''` | `''`（**不回退**，因为 `'' !== null`，`??` 不触发） | 创建时没传值但被 storeEggVariables 插了空行；或管理员/用户把值清空后保存 |
| ③ | **完全无记录**（LEFT JOIN 找不到） | `null` | `$default_value`（**回退到 Egg 默认**） | 只有一种：服务器创建之后 Egg 模板又**新增了**变量（新的 EggVariable，老服务器创建时它还不存在） |

**这是整篇文档最重要的表**：状态②和状态③在"没设置值"这个语义上很像，但 PHP `??` 的行为完全不同，直接决定 Egg 默认值的改动是否会影响运行时。

### 5.3 变量修改（StartupModificationService）时的写入

[StartupModificationService.php#L39-L47](file:///d:/fz/0508-3/solo-dogfeeding/code/202-panel/app/Services/Servers/StartupModificationService.php#L39-L47)

```php
foreach ($results as $result) {
    ServerVariable::query()->updateOrCreate(
        ['server_id' => $server->id, 'variable_id' => $result->id],
        ['variable_value' => $result->value ?? '']
    );
}
```

用 `updateOrCreate`，意味着：
- 处于状态②的变量（原本是空串）：有匹配行 → 更新为表单当前值。如果用户什么也没改，表单提交的就是之前回填的值，见下一节。
- 处于状态③的变量（Egg 新增、还没记录）：无匹配行 → **新插入一行**，值为表单中该 input 的值。保存之后这台服务器对该变量就**从③退不回 Egg 默认了**。

### 5.4 界面回填逻辑：后台 vs 客户端 —— "显示默认"不等于"实际会回退"

**后台启动页（Blade + jQuery）：**

[ServerViewController.php#L71-L87](file:///d:/fz/0508-3/solo-dogfeeding/code/202-panel/app/Http/Controllers/Admin/Servers/ServerViewController.php#L71-L87) 向 JS 注入 `Pterodactyl.server_variables`：

```php
$variables = $this->environmentService->handle($server);
$this->plainInject(['server_variables' => $variables]);
```

EnvironmentService 的 Step 1 就已经用 `server_value ?? default_value` 跑过一次，所以注入的是"运行时会使用的最终值"（不是原始 DB 值）。

[startup.blade.php#L151-L170](file:///d:/fz/0508-3/solo-dogfeeding/code/202-panel/resources/views/admin/servers/view/startup.blade.php#L151-L170) 回填：

```js
$.each(_.get(objectChain, 'variables', []), function (i, item) {
    var setValue = _.get(Pterodactyl.server_variables, item.env_variable, item.default_value);
    $('#egg_variable_' + item.env_variable).val(setValue);
});
```

三种状态在后台 Input 里显示的值：

| 状态 | EnvironmentService 产出 `Pterodactyl.server_variables[env]` | Input 显示 | 含义 |
|------|--------------------------------------------------------------|------------|------|
| ① 非空值 | 非空字符串原值 | 原值 | 直观正确 |
| ② 空串记录 | `'' ?? default_value` → **仍是 `''`**（`??` 不触发） | 空（用户看到一个空白输入框） | ❗ 这里显示空，**不**是显示 default，因为 `??` 没生效。但实际运行时也确实用 `''`，所以显示和运行一致 |
| ③ 无记录（Egg 新增） | `null ?? default_value` → **Egg default** | Egg 当前的 `default_value` | ✅ 显示 default，运行时也用 default，一致 |

**用户控制台前端（React）：**

[VariableBox.tsx#L114-L123](file:///d:/fz/0508-3/solo-dogfeeding/code/202-panel/resources/scripts/components/server/startup/VariableBox.tsx#L114-L123)：

```tsx
<Input
    defaultValue={variable.serverValue ?? ''}
    placeholder={variable.defaultValue}
/>
```

后端通过 [EggVariableTransformer.php#L23-L31](file:///d:/fz/0508-3/solo-dogfeeding/code/202-panel/app/Transformers/Api/Client/EggVariableTransformer.php#L23-L31) 传出**原始**的 `server_value` 和 `default_value`，前端自己处理：

| 状态 | `variable.serverValue` | Input 显示 | placeholder（灰色提示） | 含义 |
|------|------------------------|------------|-------------------------|------|
| ① 非空值 | 非空字符串 | 原值 | Egg default | 直观正确 |
| ② 空串记录 | `''`（JS 里不是 null） | **空字符串**（显示一个空框） | Egg default | ❗ 运行时用 `''`，但 placeholder 会"假装"显示 default。**这是唯一一处界面会"显示默认值"但运行时并不会回退的情况**，用户极易被误导：灰色提示的 default 是假的，实际启动用的是空串 |
| ③ 无记录（Egg 新增） | `null` → `null ?? ''` = `''` | **空框** | Egg default | 运行时用的是 Egg default，但前端展示成了空框。显示与实际不一致 |

**前后端对比的关键结论：**

- **后台启动页**：对状态②显示空、对状态③显示 default —— 与实际运行时的行为一致
- **用户控制台**：对状态②显示空框但 placeholder 显示 default（**placeholder 不是真实值**），对状态③也显示空框 placeholder 显示 default（但实际运行时真的会用 default）—— 两种状态在用户界面长得一样，实际行为天差地别

### 5.5 真正的优先级总表（含状态分支）

对 Egg 自定义环境变量（env_variable 不与内置 key 冲突）：

```
运行时会生效的值
  ├─ 如果 server_variables 有该 variable_id 记录（状态①②，服务器创建时就有的变量都是这种）
  │    ├─ variable_value 非空  →  用它            ←── 优先级最高（用户/管理员明确设置）
  │    └─ variable_value = ''  →  用 ''（空串）   ←── 注意：不会回退 default！
  └─ 如果 server_variables 无该 variable_id 记录（状态③，仅 Egg 事后新增的变量）
       每次启动动态取 egg_variables.default_value  ←── Egg 改 default 会直接影响（未保存前）
            │
            ▼
  之后更高优先级对**同名 key** 的覆盖（EnvironmentService 内部同 key put）：
    3. Panel 内置环境映射（STARTUP、P_SERVER_UUID 等）
    4. 配置文件 pterodactyl.environment_variables
    5. 运行时 EnvironmentService::setEnvironmentKey() 动态闭包（最高）
```

### 5.6 "被重置"的五种真实场景

| 现象 | 数据库状态 | 根因 |
|------|------------|------|
| 创建时没填的变量，界面永远显示空框，Egg 改了 default 也没反应 | ②（空串记录） | 因为创建时已经插入了 `variable_value=''` 行，`??` 永远不触发回退。这不是"被重置"，是**一直就是空串** |
| Egg 新增了一个变量，老服重启后值突然变了 | ③（无记录） | 状态③每次启动都去读 Egg 的最新 default，Egg 改了就"跟着走" → 这才是真正的**无感知动态变化**来源 |
| Egg 新增了一个变量，老服一直用 default，某管理员打开启动页点了一次"保存"（什么都没改），然后 Egg 再改 default 就失效了 | ③ → 保存后变成 ① | 保存时 updateOrCreate 把当时看到的 default（后台 Input 里显示的就是 EnvironmentService 回退后的 default）固化进了 server_variables。从此以后它就是"有记录"的了，再也不会回退 |
| 用户说在控制台看到某变量是 `latest`（灰色 placeholder），实际启动却用空串 | ②（空串记录） | 前端 VariableBox 里 placeholder 显示的是 default_value，但 defaultValue 属性绑定的是 `serverValue ?? ''`。这是界面误导，运行时用的是 `''` |
| 某变量一直工作正常，某次后台保存启动页后变成空串 | ②（之前是 ① 被改成空串） | 后端 StartupModificationService 中，VariableValidatorService 把 Egg 全部变量都收出来，updateOrCreate 会把表单提交的**所有变量**都写库。如果后台 Input 显示空（状态②），管理员保存时就会把其他变量原封不动地重新 update——如果管理员在页面上碰巧清了某个 input 再保存，这个变化会落库 |

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
