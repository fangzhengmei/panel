# 启动变量覆盖与校验全链路分析（startup variable validation & override）

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
| `default_value` | 默认值，仅当 `server_variables` 表中完全不存在该变量记录时才会兜底（空串记录不算"未设置"） |
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

### 5.3 变量修改：两条完全不同的保存路径

变量保存有**两条完全不同的代码路径**，混在一起讲是运维踩坑的主要来源。

##### 路径 A：后台启动页——整表提交，批量写库

入口：[ServersController@saveStartup](file:///d:/fz/0508-3/solo-dogfeeding/code/202-panel/app/Http/Controllers/Admin/ServersController.php#L178-L197) → [StartupModificationService](file:///d:/fz/0508-3/solo-dogfeeding/code/202-panel/app/Services/Servers/StartupModificationService.php#L29-L63)

后台启动页只有一个"保存"按钮，点击后表单中**所有变量的当前值**一起提交。后端拿到完整 `environment` 字典后，由 VariableValidatorService 校验全部变量，再循环 updateOrCreate：

```php
foreach ($results as $result) {
    ServerVariable::query()->updateOrCreate(
        ['server_id' => $server->id, 'variable_id' => $result->id],
        ['variable_value' => $result->value ?? '']
    );
}
```

特点：
- **一次请求写入所有变量**，哪怕管理员只改了一个值，其他变量的当前显示值也会重新落库
- 对状态③的变量来说，后台 Input 显示的是 EnvironmentService 回退到的 Egg default → 提交的也是这个 default → 固化后与运行时一致
- 管理员"什么都不改、只点保存"也会把全部变量固化——这是**整表提交**的副作用

##### 路径 B：客户端控制台——单变量即时保存

入口：[VariableBox.tsx#L30-L52](file:///d:/fz/0508-3/solo-dogfeeding/code/202-panel/resources/scripts/components/server/startup/VariableBox.tsx#L30-L52) → [updateStartupVariable.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/202-panel/resources/scripts/api/server/updateStartupVariable.ts) → `PUT /api/client/servers/{uuid}/startup/variable`

客户端**没有保存按钮**。每种控件的用户操作都直接触发 `setVariableValue`（经 500ms 防抖），立即向 API 发送单个变量的更新请求：

| 控件 | 触发事件 | 发送值 |
|------|---------|--------|
| Input | `onKeyUp`（每次键盘抬起） | `e.currentTarget.value`（输入框当前内容） |
| Select | `onChange`（选中项变化） | `e.target.value`（新选中项的值） |
| Switch | `onChange`（开关切换） | 翻转后的值（如 `'1'` ↔ `'0'` 或 `'true'` ↔ `'false'`） |

API 端 [StartupController@update](file:///d:/fz/0508-3/solo-dogfeeding/code/202-panel/app/Http/Controllers/Api/Client/Servers/StartupController.php#L53-L98) 只操作**请求中指定的那一个变量**：

```php
$this->repository->updateOrCreate([
    'server_id' => $server->id,
    'variable_id' => $variable->id,
], [
    'variable_value' => $request->input('value') ?? '',
]);
```

特点：
- **一次请求只写一个变量**，其他变量完全不受影响
- 不存在"整表保存"——用户改哪个变量就固化哪个，没有被动连坐
- 但对于**没有主动操作过的变量**（比如状态③的 Switch 显示关），因为用户没有触发 onChange，所以**不会被写入**，保持状态③不变
- Input 控件的 `onKeyUp` + 500ms 防抖意味着：用户只要在输入框里敲了任何字符，500ms 无后续输入后就会自动保存。**不需要按回车，也没有回车触发的逻辑**

##### 两条路径对状态③的关键差异

| 维度 | 后台整表保存（路径 A） | 客户端单变量即时保存（路径 B） |
|------|----------------------|------------------------------|
| 触发方式 | 点"保存"按钮 | Input 键盘抬起 / Select 选择 / Switch 切换 |
| 写入范围 | 全部变量一起写 | 只写被操作的那一个变量 |
| 状态③未操作变量的命运 | **全部被固化**（显示值落库） | **保持状态③不变**（未触发就不会写） |
| 状态③ Switch（关/开） | 点保存 → Egg default 被固化 → ✅ 一致 | 点开关 → 发送翻转值（如 '0'）→ ❌ 值变为假 |
| 状态③ Switch 什么都没点 | 点保存 → Egg default 被固化 → ✅ | **不触发、不写入** → 仍然保持动态跟随 |

### 5.4 界面显示逻辑全景：后台 vs 客户端、三种控件的差异

这一节是全文档最容易混淆的部分。`$server_value ?? $default_value` 这个后端表达式很简单，但**前端有多少种控件、就有多少套显示策略**，再乘以两种管理入口，组合出的差异很容易把用户和运维一起绕晕。

先交代一个前提：后端数据的两种出口形态不同。

- **后台启动页**数据来源：`EnvironmentService::handle($server)` 的输出，即已经用 `??` 算过一遍的"最终运行时值"（注入到 `Pterodactyl.server_variables`）
- **客户端 API**数据来源：[EggVariableTransformer.php](file:///d:/fz/0508-3/solo-dogfeeding/code/202-panel/app/Transformers/Api/Client/EggVariableTransformer.php#L23-L31) 传出的是**原始分开的** `server_value` 和 `default_value` 两个字段，由前端自己决定怎么用

---

#### 5.4.1 后台启动页（Blade + jQuery）：一种控件，口径一致

后台所有 Egg 变量统一渲染成 `<input type="text">`，不区分开关/下拉，由 [startup.blade.php#L151-L170](file:///d:/fz/0508-3/solo-dogfeeding/code/202-panel/resources/views/admin/servers/view/startup.blade.php#L151-L170) 控制：

```js
var setValue = _.get(Pterodactyl.server_variables, item.env_variable, item.default_value);
$('#egg_variable_' + item.env_variable).val(setValue);
```

由于 `Pterodactyl.server_variables` 就是 EnvironmentService 的输出（已跑过 `??`），所以后台 input 里显示的值**等于运行时实际会用到的值**，所见即所得。

| 状态 | EnvironmentService 产出 | Input 显示值 | 与运行时是否一致 |
|------|------------------------|-------------|-----------------|
| ① 有记录、值非空 | 原值 | 原值 | ✅ 一致 |
| ② 有记录、值 = `''` | `'' ?? default` → **仍是 `''`**（空串不是 null，`??` 不触发） | 空框 | ✅ 一致（运行时也用空串） |
| ③ 无记录（Egg 新增） | `null ?? default` → **`default_value`** | Egg 当前 default 值 | ✅ 一致 |

**保存行为（路径 A，整表提交）**：后台点"保存"时，所有变量的当前显示值一起提交（见 5.3 节路径 A）。对状态③来说，后台 Input 显示的值是 EnvironmentService 回退到的 Egg default → 提交的也是这个 default → `updateOrCreate` 把这个值写进 `server_variables` → 从状态③**固化为状态①**，值为保存那一刻的 Egg default。从此以后 Egg 再改 default 也不会影响这台服了。**注意：这是后台整表提交的效果，客户端走的是路径 B（单变量即时保存），不会因为"打开页面"就固化。**

---

#### 5.4.2 客户端控制台（React）：三种控件，三套策略

客户端 [VariableBox.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/202-panel/resources/scripts/components/server/startup/VariableBox.tsx) 会根据 Egg 变量的 `rules` 自动选择三种控件之一：

1. **Switch 开关**：规则包含 `boolean` / `in:0,1` / `in:1,0` / `in:true,false` / `in:false,true`
2. **Select 下拉框**：规则包含 `in:val1,val2,...` 且**不**是 Switch 类（即不是纯布尔二值）
3. **普通 Input**：其他所有情况（text / number / regex 等）

三种控件的 `defaultValue` / `defaultChecked` 策略完全不同，必须分别分析。

##### ① 普通 Input 控件（最常见）

```tsx
<Input
    defaultValue={variable.serverValue ?? ''}   // L122
    placeholder={variable.defaultValue}         // L123
/>
```

`defaultValue` 是 input 里真正预填的值，`placeholder` 只是没值时的灰色提示文字（不算真实值）。

| 状态 | `serverValue` | `defaultValue`（input 显示） | `placeholder`（灰色字） | 与运行时是否一致 |
|------|---------------|-------------------------------|-------------------------|-----------------|
| ① 非空 | `'1.20.1'` | `'1.20.1'` | `'1.19.4'`（Egg default） | ✅ 一致 |
| ② 空串 | `''` | `'' ?? ''` → **空框** | Egg default | ⚠️ **外观像 default，实际是空串**。placeholder 只是视觉提示，运行时用 `''`，与**显示空白的 input 一致**，但用户容易误以为灰色字就是当前值 |
| ③ 无记录（Egg 新增） | `null` | `null ?? ''` → **空框** | Egg default | ❌ **不一致**。运行时用的是 Egg default（`null ?? default`），但界面显示空框。用户看到空白 input + 灰色提示，直觉会觉得"还没设置，用默认"，但实际上运行时确实在用默认——只是显示上没体现出来 |

**关键区分**：状态②和状态③在客户端普通 Input 上**长得一模一样**（都是空框 + 灰色 default），但运行时行为完全不同（②用空串，③用 default）。用户靠肉眼完全无法分辨。

##### ② Select 下拉控件

```tsx
<Select
    defaultValue={variable.serverValue ?? variable.defaultValue}  // L99
    // ...options 来自 rules 的 in: 值列表
/>
```

Select 跟普通 Input 的策略**不一样**：它的 `defaultValue` 用的是 `serverValue ?? defaultValue`，**直接把 Egg 默认值作为兜底显示**，没有 placeholder 概念。

| 状态 | `serverValue` | `defaultValue`（Select 选中项） | 与运行时是否一致 |
|------|---------------|--------------------------------|-----------------|
| ① 非空 | `'vanilla'` | `'vanilla'` | ✅ 一致 |
| ② 空串 | `''` | `'' ?? default` → **`''`（空串）** | ⚠️ 要看 `options` 里有没有空串选项。Egg 的 `in:` 规则通常不会包含空串，所以 Select 很可能显示为**空白/无选中项**，运行时用空串 → **显示空白但值就是空，行为一致**，但用户体验差（看不到选中项） |
| ③ 无记录（Egg 新增） | `null` | `null ?? default` → **Egg default** | ✅ 一致 |

Select 控件在状态②（空串）时体验最差：既没有 placeholder 提示，又因为 options 里没有空串而显示空白，用户不知道当前值是什么。

##### ③ Switch 开关控件

```tsx
<Switch
    defaultChecked={
        isStringSwitch
            ? variable.serverValue === 'true'
            : variable.serverValue === '1'   // L78-L80
    }
/>
```

Switch 是**三种控件里最极端的**：完全没有 fallback 到 default_value 的逻辑，只跟 `serverValue` 比字符串。`null` 和 `''` 都会被判成 false。

| 状态 | `serverValue` | `defaultChecked`（开关显示） | 与运行时是否一致 |
|------|---------------|-----------------------------|-----------------|
| ① `'1'` / `'true'` | `'1'` / `'true'` | ✅ 开 | ✅ 一致 |
| ② `''`（空串） | `''` | ❌ 关（`'' !== '1'` 且 `'' !== 'true'`） | ⚠️ 如果 Egg default 就是 `0` / `false` → 一致；如果 Egg default 是 `1` / `true` → **不一致**（运行时用空串，但空串不是真值，所以其实运行时也等价于 false……要看 Egg 里该变量的语义，是否把空串当默认启用） |
| ③ `null`（无记录，Egg 新增） | `null` | ❌ 关（`null !== '1'` 且 `null !== 'true'`） | ❌ **不一致，且是三种控件里最严重的不一致**。运行时 `null ?? default` → 取 Egg default；如果 Egg 的 default 是 `'1'` / `'true'`，那运行时是启用，但界面显示关闭。用户看到开关是"关"，服务器实际用的是"开" |

**Switch 控件坑点总结**：
- 状态③ + Egg default 为真 → 界面显示关、实际运行开 → 表里不一
- 状态② + Egg default 为真 → 界面显示关、运行时是空串（大多数程序会把空串当 false 处理，所以实际效果通常也是关 → 巧合一致，但不是因为逻辑正确）
- Switch 没有"显示默认值"的概念，也没有灰色提示。用户只能看到开关是开还是关

---

#### 5.4.3 两界面对比总表（一眼找差异）

| 状态 | 运行时实际值 | 后台 Input | 客户端 Input | 客户端 Select | 客户端 Switch（假设 Egg default=真） |
|------|-------------|-----------|-------------|--------------|-------------------------------------|
| ① 非空（值为真） | 原值（真） | 原值 | 原值 | 原值 | 开 ✅ |
| ① 非空（值为假） | 原值（假） | 原值 | 原值 | 原值 | 关 ✅ |
| ② 空串 | `''` | 空框（一致） | 空框 + 灰色 default（显示误导） | 空白/无选中（体验差） | 关（巧合一致，因空串被当假） |
| ③ 无记录 | Egg default | Egg default（一致 ✅） | 空框 + 灰色 default（不一致 ❌，实际在用 default 但显示空） | Egg default（一致 ✅） | 关（不一致 ❌，若 Egg default=真则表里完全相反） |

#### 5.4.4 "显示默认"和"真实默认"的四种含义

"默认值"这个词在上下文中至少有四种不同的语义，讨论时必须先对齐：

| 语义 | 含义 | 场景 |
|------|------|------|
| Egg 模板默认值 | `egg_variables.default_value` 字段 | 数据库里写死的模板默认 |
| 运行时实际值 | `server_value ?? default_value` 最终计算结果 | 下发给 Wings、进容器 env、替换 `{{}}` 的真实值 |
| 界面显示默认值（placeholder 型） | 灰色提示文字，不是真实值 | 客户端普通 Input 的 `placeholder` |
| 界面选中默认值（控件选中型） | 控件真正的 `defaultValue` / `defaultChecked` | 后台 input 的 val、客户端 Select 的 defaultValue、客户端 Switch 的 defaultChecked |

最容易踩的坑：**客户端普通 Input 的 placeholder 显示着 "latest"，用户以为当前值是 latest，但真实运行时值是空串（状态②）**，或者虽然真实运行时确实是 latest 但界面没体现（状态③）——两种情况用户都搞不清。

---

#### 5.4.5 保存后的固化行为（分两条路径讲）

5.3 节已经讲过客户端和后台走的是完全不同的代码路径，这里按两条路径分别分析固化后果。

##### 路径 A：后台整表保存的固化

后台点"保存"会把**所有变量**的当前显示值一起写入 server_variables。

| 状态 | 后台 Input 显示的值 | 保存后写入 DB 的值 | 固化效果 |
|------|---------------------|-------------------|---------|
| ① 非空 | 原值 | 原值 | 值不变 |
| ② 空串 | 空框 | `''` | 值不变（还是空串） |
| ③ 无记录 | Egg default（EnvironmentService 回退后的值） | Egg default | **从③变为①，动态跟随能力丧失** |

后台整表保存对状态③来说是一个**全量固化**：所有未操作过的 Egg 后加变量也会被显示值（Egg default）固化进 DB。

##### 路径 B：客户端单变量即时保存的固化

客户端**只固化被用户主动操作过的那一个变量**。没有被操作的变量保持原状态不变。

| 控件 | 用户操作 | 状态③时发送的值 | 写入 DB 的值 | 固化效果 |
|------|---------|----------------|-------------|---------|
| Input | 在输入框内敲字（`onKeyUp` + 500ms 防抖） | 输入框当前内容 | 输入内容 | **从③变为①或②** |
| Input | 输入框空白时无操作 | — | — | **不触发，保持状态③** |
| Select | 选择某个选项（`onChange`） | 选中项值 | 选中项值 | **从③变为①** |
| Switch | 点击开关（`onChange`） | 翻转值（如 `'1'` 或 `'false'`） | 翻转值 | **从③变为①，且值被翻转** |

⚠️ **Switch 控件的特殊坑**：状态③时 Switch 显示"关"（因为 `defaultChecked` 只跟 serverValue 比，null 被视为关），但运行时是 Egg default（可能是"开"）。用户点击开关时，客户端发送的是**翻转后的值**（如 `'1'`），所以：
- 点一下 → 从"关"变成"开" → 发送 `'1'` → 固化为"开" → ✅ 与 Egg default 一致（前提是 Egg default 就是"开"）
- 再点一下 → 从"开"变成"关" → 发送 `'0'` → 固化为"关" → ❌ 值被永久改为假

**与后台整表保存的关键区别**：客户端的 Switch 不会因为"打开页面看了但没点"而被固化——只有用户真正点击了开关才会触发保存。而后台是"打开页面点保存，不管有没有改，全部变量都固化"。

#### 5.4.6 哪些判断只适用于"Egg 后加变量"（状态③）

以下现象/结论**仅在状态③（Egg 模板在服务器创建之后新增的变量）下成立**，其他状态下不适用：

1. "Egg 改 default_value 会影响这台服的运行时值" — 只有状态③才会动态跟随
2. "保存一次就固化了，Egg 再改也不生效" — 只有状态③→①/② 这个转变才有"固化"效应，状态①/②保存本来就是更新自己的值
3. "后台启动页显示的值与运行时一致" — 其实对三种状态都一致，但状态③下后台显示的是 Egg default 而不是空，这个特性在③的时候最容易被误认为"后台怎么和我客户端看到的不一样"
4. "客户端 Switch 控件显示与实际相反" — 只有状态③且 Egg default 为真时才会显示关但运行开。但用户点击开关时客户端会发送翻转值（即时保存），不会像后台那样"整表连坐"
5. "什么都没改点了保存，值就变了" — 只有后台整表保存（路径 A）才会这样：未操作的 Egg 后加变量也会被固化。客户端（路径 B）没有"整表保存"按钮，未操作的变量不会被写入
6. "服务器创建时间晚于 egg_variables.created_at" — 这个判断方法仅用于识别状态③

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
| 修改 Egg 模板 default_value | VariableUpdateService | ❌ 否 | ✅ 仅对状态③（server_variables 完全无该变量记录，即 Egg 在该服创建后新增的变量）生效，且需重启；对状态①/②（有记录，哪怕是空串）的老服完全无效 |
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

### 9.1 一步定位 SQL：判断三种状态

直接跑这个 SQL，拿到某服所有变量的真实状态三要素：

```sql
SELECT
    ev.env_variable,
    ev.default_value      AS egg_default,
    sv.variable_value     AS instance_value,
    CASE
        WHEN sv.id IS NULL THEN 'state_3_no_record'
        WHEN sv.variable_value = '' THEN 'state_2_empty_string'
        ELSE 'state_1_has_value'
    END                   AS reality_state,
    -- 模拟 PHP 中 `??` 的真实运行时值（不是 COALESCE 语义！）
    CASE
        WHEN sv.id IS NULL THEN ev.default_value   -- 无记录，走 Egg 默认
        ELSE sv.variable_value                     -- 有记录，哪怕是空串也用原值
    END                   AS runtime_actual_value,
    ev.created_at         AS egg_var_created_at,
    s.created_at          AS server_created_at
FROM egg_variables ev
CROSS JOIN servers s
LEFT JOIN server_variables sv
       ON sv.variable_id = ev.id AND sv.server_id = s.id
WHERE s.id = <YOUR_SERVER_ID>
  AND ev.egg_id = s.egg_id
ORDER BY ev.env_variable;
```

SQL 关键理解：`runtime_actual_value` 是下发给 Wings 时真正会使用的值；`egg_var_created_at > server_created_at` 的行**极大概率**是 状态③（无记录）（Egg 在服务器创建之后新增的变量）。

### 9.2 典型症状 → 根因 → 处理对照表

以下所有场景中的状态编号对应 5.2 节定义：①=有记录非空、②=有记录空串、③=无记录（仅 Egg 后加变量才会出现）。

| # | 用户/运维观察到的现象 | reality_state | 控件类型 | 代码根因 | 建议处理 |
|---|---------------------|---------------|----------|----------|----------|
| 1 | 创建时没填某变量，之后一直是空白，Egg 改了 default 也完全没反应 | ② 空串 | 后台 Input / 客户端 Input | 服务器创建流程 `storeEggVariables` 已为所有 Egg 变量插入空行 `''`；空串不是 null，`??` 不会走 Egg 的 default | 正常行为。**注意：这不是"被重置"，是一直就是空串**。如果希望让这台服跟随 Egg default：`DELETE FROM server_variables WHERE server_id=? AND variable_id=?`（慎用，更推荐手动在后台/客户端填值显式声明） |
| 2 | Egg 管理员新增了某变量 X，老服重启后"值变了"（没人改过） | ③ 无记录 | 所有 | server_variables 表无该 variable_id 行 → 走 `null ?? default_value` → 动态读取 Egg 当前 default。Egg 维护者改了 default 就会自动跟随 | **这是唯一会出现"无人修改但值变动"的场景，且仅适用于 Egg 后加变量**。若希望稳定不跟随：打开该服务器后台启动页，点一次保存（哪怕不改值），StartupModificationService 会把当前 default 固化入 server_variables 表 |
| 3 | 点了一次保存后，Egg 再改 default 就不生效了 | ③ → ① 或 ② | 所有 | 保存操作的 `updateOrCreate` 把当时显示的值写进了 DB。对状态③来说是从"动态跟随"变成"有记录"，永不回退 | 正常预期（仅对状态③是"固化"，状态①②保存本来就是更新值）。需要更新时手动改该服务器的值 |
| 4 | 用户控制台某变量 Input 框是空白，但灰色提示字显示着 "latest"，实际启动日志中用的是空串 | ② 空串 | 客户端 Input | [VariableBox.tsx#L122-L123](file:///d:/fz/0508-3/solo-dogfeeding/code/202-panel/resources/scripts/components/server/startup/VariableBox.tsx#L122-L123) `defaultValue={variable.serverValue ?? ''}` 绑的是空串，而 `placeholder={variable.defaultValue}` 只是视觉提示，**placeholder 不会作为真实值**。运行时走 `''` | 典型界面误导。客户端 Input 的保存触发方式是 `onKeyUp` + 500ms 防抖（见 [VariableBox.tsx#L115-L118](file:///d:/fz/0508-3/solo-dogfeeding/code/202-panel/resources/scripts/components/server/startup/VariableBox.tsx#L115-L118)），用户只要在输入框内敲了字符，500ms 后就会自动保存——**不需要按回车**。但输入框空白时不触发任何保存，所以状态②和状态③的空白 Input 只能通过手动填值来纠正 |
| 5 | 用户控制台某 Select 下拉框一片空白，看不到选中项 | ② 空串 | 客户端 Select | Select 的 `defaultValue` 是 `serverValue ?? defaultValue`，状态②空串时 `'' ?? default` 还是 `''`；但 Egg 的 `in:` 选项里通常没有空串，导致没有 option 被选中，显示空白 | 体验问题而非逻辑问题。让用户选一个值保存即可。或者检查 Egg 规则是否允许空值，必要时在 `in:` 里加上默认选项 |
| 6 | 用户控制台某开关显示"关"，但实际服务器运行时是启用的 | ③ 无记录（Egg 新增）且 Egg default=真 | 客户端 Switch | Switch 的 `defaultChecked` 只跟 `serverValue` 比，**完全不回退 default**。状态③时 `serverValue=null` → 显示关；但运行时 `null ?? default` → 用 Egg default=真 → 实际是开的 | **表里不一，必须修**。方法一：用户点一下开关（发送翻转值如 `'1'`，固化为开 → 与 Egg default=真一致）；方法二：管理员在后台启动页保存一次（整表保存，会把 Egg default 正确固化）。⚠️ 注意：方法一不能"点一下再切回来"，因为第二次切换会发送 `'0'`，值反而变成关了。客户端是单变量即时保存，每次操作立即落库 |
| 7 | 后台启动页某变量显示 "v1.20"，Egg 的 default 明明已经改成 "v1.21" 了 | ① 或 ② | 后台 Input | 服务器有自己的实例值，永不回退。后台显示的是 EnvironmentService 计算后的真实运行值，显示完全正确 | 正常。想同步到 Egg 新 default，直接在后台把值改成 "v1.21" 并保存（清空保存会变空串，**不会**自动回退到 Egg 新 default） |
| 8 | 后台显示的值和客户端显示的值**不一样** | 视情况 | 对比：后台 Input vs 客户端对应控件 | 后台用 EnvironmentService 的输出（即运行时最终值）；客户端按控件类型各自有不同的 fallback 策略。状态②和状态③在很多控件上显示得都跟后台不一样 | 以**后台**为准，后台显示的就是实际运行时值。客户端显示差异请对照 5.4.3 的对照表判断是哪一类偏差 |
| 9 | 一批老服务器某变量同时"变值"，没人动过它们各自的启动页 | ③ 无记录（多台服同时） | 所有 | Egg 管理员最近在模板上新增了一个变量，且这批老服都没保存过 → 全部动态跟随 Egg 的 default。Egg 维护者改了 default_value 导致集体变化 | 对这批服批量锁定：<br>批量 `INSERT INTO server_variables (server_id, variable_id, variable_value) SELECT s.id, ev.id, ev.default_value FROM servers s JOIN egg_variables ev ON ev.egg_id = s.egg_id AND ev.env_variable = 'X' WHERE s.node_id = ? AND NOT EXISTS (SELECT 1 FROM server_variables sv WHERE sv.server_id = s.id AND sv.variable_id = ev.id)` |
| 10 | 改了 Egg 里**老变量**的 default，老服完全没变化 | ① 或 ② | 所有 | 服务器创建时该 variable 已存在 → server_variables 必然有行。`??` 永不触发，老服不会感知到 default 的更新 | 正常。**改老变量的 default 只影响在这之后新创建的服务器**；对既有老服无效，得批量 UPDATE server_variables 才行 |
| 11 | 某变量在客户端说改成功了，刷新又变回旧值 | 任意 | 所有 | 客户端 StartupController 要求 `user_editable = true` 才允许改，否则抛 403。如果 API 响应被前端吞了，看起来就像"没保存" | 查该变量的 `egg_variables.user_editable`。false 的话让管理员在后台启动页改，或授权用户 `startup.update` 权限 + 变量设 `user_editable=true` |
| 12 | 改完变量立即看服务器详情页还是旧值 | 任意 | 所有 | StartupModificationService **不调用** `DaemonServerRepository.sync()`。Wings 只在启动服务器、自身重启、收到 sync HTTP 请求时拉新配置 | 正常。必须重启服务器（`/api/client/servers/:uuid/power` = restart）才会用新环境变量。构建参数（内存/CPU/端口）改了会自动 sync |

### 9.3 通用排查顺序（逐步收敛）

1. **先看真实状态**：跑 9.1 的 SQL，确认所有变量的 `reality_state` 和 `runtime_actual_value`，判断属于①/②/③哪一种
2. **看时间戳**：`egg_var_created_at > server_created_at` 的 variable 对这台服大概率是状态③（除非手动补过数据）。**状态③的所有"动态变化"结论都只适用于 Egg 后加变量**
3. **确定用户用的是哪个入口**：后台启动页还是客户端控制台？后台永远是对的（EnvironmentService 输出 = 运行时值），客户端要分控件类型
4. **判断控件类型**：
   - 变量规则含 `boolean` / `in:0,1` / `in:true,false` → Switch 控件（最容易表里不一）
   - 变量规则含 `in:` 且不是上面那些 → Select 下拉控件
   - 其他 → 普通 Input（最容易被 placeholder 误导）
5. **对照 5.4.3 的两界面对比总表**：看显示值与运行时是否一致，是哪一类偏差
6. **看启动命令 vs 环境变量**：`StartupCommandService`（组装 invocation 字符串里 `{{...}}` 的替换）和 `EnvironmentService`（Docker 容器 env 注入）都走同一套 `??` 逻辑，值是一致的。用 `php artisan tinker` → `app(EnvironmentService::class)->handle(Server::find(id))` 打印实际下发数组
7. **查数据库行级 audit**：Activity Log 看 `server:startup.edit`；server_variables 本身没有 created_at/updated_at 历史，必要时开 binlog 或临时加触发器
8. **确认权限**：某些变量的 `user_viewable=false` 或 `user_editable=false`，客户端看不到/改不了，必须管理员走后台或 Application API
9. **确认 Wings 侧是否收到**：最后在 Wings 节点上 `docker inspect <容器ID>` 看 `Config.Env`，如果和 Panel 上 EnvironmentService 输出不一致 → 是服务器没重启（容器用的是老环境变量）；如果和 Panel 一致 → 是游戏服自身读错了

---

## 10. 核心代码索引

| 关注点 | 文件 |
|--------|------|
| 变量模型 | [EggVariable.php](file:///d:/fz/0508-3/solo-dogfeeding/code/202-panel/app/Models/EggVariable.php)、[ServerVariable.php](file:///d:/fz/0508-3/solo-dogfeeding/code/202-panel/app/Models/ServerVariable.php) |
| 服务器变量关联（LEFT JOIN server_value） | [Server.php#L288-L301](file:///d:/fz/0508-3/solo-dogfeeding/code/202-panel/app/Models/Server.php#L288-L301) |
| 变量校验服务（管理员返回所有 EggVariable） | [VariableValidatorService.php](file:///d:/fz/0508-3/solo-dogfeeding/code/202-panel/app/Services/Servers/VariableValidatorService.php) |
| 创建时批量插入 server_variables（所有变量插空行） | [ServerCreationService.php#L188-L201](file:///d:/fz/0508-3/solo-dogfeeding/code/202-panel/app/Services/Servers/ServerCreationService.php#L188-L201) |
| 启动变量修改（updateOrCreate 固化） | [StartupModificationService.php](file:///d:/fz/0508-3/solo-dogfeeding/code/202-panel/app/Services/Servers/StartupModificationService.php) |
| 启动命令占位符替换（`server_value ?? default_value`） | [StartupCommandService.php](file:///d:/fz/0508-3/solo-dogfeeding/code/202-panel/app/Services/Servers/StartupCommandService.php) |
| 环境变量生成（下发用，5 级优先级同 key put） | [EnvironmentService.php](file:///d:/fz/0508-3/solo-dogfeeding/code/202-panel/app/Services/Servers/EnvironmentService.php) |
| 后台启动页 Blade 模板（统一 Input 控件） | [startup.blade.php#L151-L170](file:///d:/fz/0508-3/solo-dogfeeding/code/202-panel/resources/views/admin/servers/view/startup.blade.php#L151-L170) |
| 后台启动页控制器（注入 EnvironmentService 输出） | [ServerViewController.php#L71-L87](file:///d:/fz/0508-3/solo-dogfeeding/code/202-panel/app/Http/Controllers/Admin/Servers/ServerViewController.php#L71-L87) |
| 客户端 VariableBox 总入口（三种控件分发） | [VariableBox.tsx#L54-L128](file:///d:/fz/0508-3/solo-dogfeeding/code/202-panel/resources/scripts/components/server/startup/VariableBox.tsx#L54-L128) |
| 客户端普通 Input 控件（defaultValue vs placeholder） | [VariableBox.tsx#L114-L123](file:///d:/fz/0508-3/solo-dogfeeding/code/202-panel/resources/scripts/components/server/startup/VariableBox.tsx#L114-L123) |
| 客户端 Select 下拉控件（`serverValue ?? defaultValue`） | [VariableBox.tsx#L96-L110](file:///d:/fz/0508-3/solo-dogfeeding/code/202-panel/resources/scripts/components/server/startup/VariableBox.tsx#L96-L110) |
| 客户端 Switch 开关控件（完全不回退 default） | [VariableBox.tsx#L75-L90](file:///d:/fz/0508-3/solo-dogfeeding/code/202-panel/resources/scripts/components/server/startup/VariableBox.tsx#L75-L90) |
| 客户端防抖保存函数（500ms debounce，setVariableValue） | [VariableBox.tsx#L30-L52](file:///d:/fz/0508-3/solo-dogfeeding/code/202-panel/resources/scripts/components/server/startup/VariableBox.tsx#L30-L52) |
| 客户端 API 请求模块（PUT 单变量更新） | [updateStartupVariable.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/202-panel/resources/scripts/api/server/updateStartupVariable.ts) |
| 客户端 API 数据 Transformer（原始 server_value/default_value 分离） | [EggVariableTransformer.php#L23-L31](file:///d:/fz/0508-3/solo-dogfeeding/code/202-panel/app/Transformers/Api/Client/EggVariableTransformer.php#L23-L31) |
| Wings 配置结构组装 | [ServerConfigurationStructureService.php](file:///d:/fz/0508-3/solo-dogfeeding/code/202-panel/app/Services/Servers/ServerConfigurationStructureService.php) |
| Wings 拉取接口（被动触发配置同步） | [ServerDetailsController.php](file:///d:/fz/0508-3/solo-dogfeeding/code/202-panel/app/Http/Controllers/Api/Remote/Servers/ServerDetailsController.php) |
| Panel→Wings sync 调用（只有 BuildModificationService 在用） | [DaemonServerRepository.php#L63-L72](file:///d:/fz/0508-3/solo-dogfeeding/code/202-panel/app/Repositories/Wings/DaemonServerRepository.php#L63-L72) |
| 客户端单变量更新接口（含 user_editable 二次校验） | [StartupController.php（客户端）](file:///d:/fz/0508-3/solo-dogfeeding/code/202-panel/app/Http/Controllers/Api/Client/Servers/StartupController.php) |
| 管理员 Application API 更新接口 | [StartupController.php（应用 API）](file:///d:/fz/0508-3/solo-dogfeeding/code/202-panel/app/Http/Controllers/Api/Application/Servers/StartupController.php) |
| 后台 Web 保存启动页入口（ServersController 内部调用） | [ServersController@saveStartup](file:///d:/fz/0508-3/solo-dogfeeding/code/202-panel/app/Http/Controllers/Admin/ServersController.php#L178-L197) |
