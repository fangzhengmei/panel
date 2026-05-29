# Subuser ACL 权限校验与服务端控制实现分析

本文档深入分析 Pterodactyl Panel 中 Subuser（子用户）ACL 系统的实现机制，包括细粒度权限位配置、API 层逐请求校验流程、权限分配过滤规则，以及权限变更实时生效的原理。

---

## 一、细粒度权限位的定义与存储

### 1.1 权限位定义

权限定义集中在 `app/Models/Permission.php` 中，采用「类别.动作」的命名方式，共 11 个权限类别、**40 个具体权限位**。

**权限常量定义** (`app/Models/Permission.php:18-66`):
```php
// websocket - 1个
public const ACTION_WEBSOCKET_CONNECT = 'websocket.connect';

// control - 4个
public const ACTION_CONTROL_CONSOLE = 'control.console';
public const ACTION_CONTROL_START = 'control.start';
public const ACTION_CONTROL_STOP = 'control.stop';
public const ACTION_CONTROL_RESTART = 'control.restart';

// database - 5个
public const ACTION_DATABASE_READ = 'database.read';
public const ACTION_DATABASE_CREATE = 'database.create';
public const ACTION_DATABASE_UPDATE = 'database.update';
public const ACTION_DATABASE_DELETE = 'database.delete';
public const ACTION_DATABASE_VIEW_PASSWORD = 'database.view_password';

// schedule - 4个
public const ACTION_SCHEDULE_READ = 'schedule.read';
public const ACTION_SCHEDULE_CREATE = 'schedule.create';
public const ACTION_SCHEDULE_UPDATE = 'schedule.update';
public const ACTION_SCHEDULE_DELETE = 'schedule.delete';

// user - 4个
public const ACTION_USER_READ = 'user.read';
public const ACTION_USER_CREATE = 'user.create';
public const ACTION_USER_UPDATE = 'user.update';
public const ACTION_USER_DELETE = 'user.delete';

// backup - 5个
public const ACTION_BACKUP_READ = 'backup.read';
public const ACTION_BACKUP_CREATE = 'backup.create';
public const ACTION_BACKUP_DELETE = 'backup.delete';
public const ACTION_BACKUP_DOWNLOAD = 'backup.download';
public const ACTION_BACKUP_RESTORE = 'backup.restore';

// allocation - 4个
public const ACTION_ALLOCATION_READ = 'allocation.read';
public const ACTION_ALLOCATION_CREATE = 'allocation.create';
public const ACTION_ALLOCATION_UPDATE = 'allocation.update';
public const ACTION_ALLOCATION_DELETE = 'allocation.delete';

// file - 7个
public const ACTION_FILE_READ = 'file.read';
public const ACTION_FILE_READ_CONTENT = 'file.read-content';
public const ACTION_FILE_CREATE = 'file.create';
public const ACTION_FILE_UPDATE = 'file.update';
public const ACTION_FILE_DELETE = 'file.delete';
public const ACTION_FILE_ARCHIVE = 'file.archive';
public const ACTION_FILE_SFTP = 'file.sftp';

// startup - 3个
public const ACTION_STARTUP_READ = 'startup.read';
public const ACTION_STARTUP_UPDATE = 'startup.update';
public const ACTION_STARTUP_DOCKER_IMAGE = 'startup.docker-image';

// settings - 2个
public const ACTION_SETTINGS_RENAME = 'settings.rename';
public const ACTION_SETTINGS_REINSTALL = 'settings.reinstall';

// activity - 1个
public const ACTION_ACTIVITY_READ = 'activity.read';
```

**权限类别统计表**：

| 类别 | 权限数 | 权限列表 |
|------|--------|----------|
| websocket | 1 | connect |
| control | 4 | console, start, stop, restart |
| user | 4 | read, create, update, delete |
| file | 7 | read, read-content, create, update, delete, archive, sftp |
| backup | 5 | read, create, delete, download, restore |
| allocation | 4 | read, create, update, delete |
| startup | 3 | read, update, docker-image |
| database | 5 | read, create, update, delete, view_password |
| schedule | 4 | read, create, update, delete |
| settings | 2 | rename, reinstall |
| activity | 1 | read |
| **合计** | **40** | |

**权限类别结构** (`app/Models/Permission.php:101-209`):
```php
protected static array $permissions = [
    'websocket' => [
        'description' => '允许用户连接到服务器 WebSocket，查看控制台输出和实时状态',
        'keys' => ['connect' => '...'],
    ],
    'control' => [ /* 电源控制权限 */ ],
    'user' => [ /* 子用户管理权限 */ ],
    'file' => [ /* 文件管理权限 */ ],
    'backup' => [ /* 备份管理权限 */ ],
    'allocation' => [ /* 端口分配权限 */ ],
    'startup' => [ /* 启动参数权限 */ ],
    'database' => [ /* 数据库管理权限 */ ],
    'schedule' => [ /* 定时任务权限 */ ],
    'settings' => [ /* 服务器设置权限 */ ],
    'activity' => [ /* 活动日志权限 */ ],
];
```

### 1.2 权限存储结构

**数据模型演进**：2020 年的数据库迁移将独立的 `permissions` 表合并到 `subusers` 表的 JSON 字段中（`database/migrations/2020_03_22_163911_merge_permissions_table_into_subusers.php`）。

**当前存储结构** (`app/Models/Subuser.php:45-49`):
```php
protected $casts = [
    'user_id' => 'int',
    'server_id' => 'int',
    'permissions' => 'array',  // JSON 字段存储权限数组
];
```

**subusers 表结构**：

| 字段 | 类型 | 说明 |
|------|------|------|
| id | INT | 主键 |
| user_id | INT | 关联用户 ID |
| server_id | INT | 关联服务器 ID |
| permissions | JSON | 权限字符串数组，如 `["websocket.connect", "control.console"]` |
| created_at | DATETIME | 创建时间 |
| updated_at | DATETIME | 更新时间 |

### 1.3 权限获取服务

`GetUserPermissionsService` 是核心权限查询服务，根据用户身份返回不同的权限集合。

**权限判定逻辑** (`app/Services/Servers/GetUserPermissionsService.php:15-33`):
```php
public function handle(Server $server, User $user): array
{
    // 1. 根管理员或服务器所有者 → 通配符权限
    if ($user->root_admin || $user->id === $server->owner_id) {
        $permissions = ['*'];
        if ($user->root_admin) {
            $permissions[] = 'admin.websocket.errors';
            $permissions[] = 'admin.websocket.install';
            $permissions[] = 'admin.websocket.transfer';
        }
        return $permissions;
    }

    // 2. 子用户 → 从 subusers.permissions 字段读取
    $subuserPermissions = $server->subusers()->where('user_id', $user->id)->first();
    return $subuserPermissions ? $subuserPermissions->permissions : [];
}
```

**权限层级**：
- `root_admin` → 拥有所有权限（`['*']`） + 3 个管理员专属权限
- `owner_id`（服务器所有者）→ 拥有所有权限（`['*']`）
- `subuser`（子用户）→ 仅拥有分配的具体权限数组

---

## 二、权限分配时的默认注入与过滤规则

在创建或更新子用户权限时，系统会通过 `getDefaultPermissions()` 方法对用户提交的权限进行严格的过滤和处理。

### 2.1 权限处理流程总览

```
用户提交权限数组
    ↓
[白名单过滤] 只保留系统定义的有效权限
    ↓
[强制注入] 自动添加 websocket.connect 权限
    ↓
[去重处理] 移除重复的权限项
    ↓
最终保存到数据库
```

### 2.2 核心实现代码

**权限过滤与注入逻辑** (`app/Http/Controllers/Api/Client/Servers/SubuserController.php:154-168`):
```php
protected function getDefaultPermissions(Request $request): array
{
    // 1. 生成系统允许的权限白名单
    $allowed = Permission::permissions()
        ->map(function ($value, $prefix) {
            return array_map(function ($value) use ($prefix) {
                return "$prefix.$value";
            }, array_keys($value['keys']));
        })
        ->flatten()
        ->all();

    // 2. 白名单过滤：只保留用户提交中存在于白名单的权限
    $cleaned = array_intersect($request->input('permissions') ?? [], $allowed);

    // 3. 强制注入 websocket.connect 并去重
    return array_unique(array_merge($cleaned, [Permission::ACTION_WEBSOCKET_CONNECT]));
}
```

### 2.3 三步处理详解

#### 第一步：生成权限白名单

通过 `Permission::permissions()` 读取 `$permissions` 配置数组，动态生成所有合法权限的完整列表。

**白名单生成逻辑**：
```php
$allowed = Permission::permissions()
    ->map(function ($value, $prefix) {
        // 对每个类别（如 websocket），拼接类别名与权限名
        return array_map(function ($value) use ($prefix) {
            return "$prefix.$value";  // 如 "websocket.connect"
        }, array_keys($value['keys']));  // 取 keys 数组的键名
    })
    ->flatten()  // 将二维数组扁平化为一维
    ->all();
```

**白名单内容示例**（共 40 项）：
```
[
    "websocket.connect",
    "control.console",
    "control.start",
    "control.stop",
    "control.restart",
    "user.read",
    "user.create",
    "user.update",
    "user.delete",
    // ... 其余 31 个权限
]
```

**设计意图**：
- 防御性编程：防止用户提交不存在的权限字符串
- 动态生成：无需手动维护白名单，新增权限时自动生效
- 与配置保持一致：白名单直接来源于 `$permissions` 定义

#### 第二步：白名单过滤

使用 `array_intersect()` 对用户提交的权限进行过滤。

```php
$cleaned = array_intersect($request->input('permissions') ?? [], $allowed);
```

**过滤规则**：
- 用户提交的权限必须**完全匹配**白名单中的字符串
- 不区分大小写？不，PHP `array_intersect` 是区分大小写的
- 不存在于白名单的权限会被**静默移除**，不报错
- 如果用户提交空数组，`$cleaned` 也为空数组

**示例**：
```
用户提交: ["control.console", "file.read", "invalid.permission", "websocket.connect"]
白名单:   ["websocket.connect", "control.console", "file.read", ...]
过滤后:   ["control.console", "file.read", "websocket.connect"]
```

**安全意义**：
- 防止注入恶意权限字符串
- 防止越权访问未定义的权限
- 防止拼写错误导致的权限异常

#### 第三步：强制注入 websocket.connect

无论用户提交什么权限，系统都会**强制注入** `websocket.connect` 权限。

```php
return array_unique(array_merge($cleaned, [Permission::ACTION_WEBSOCKET_CONNECT]));
```

**注入逻辑**：
- 使用 `array_merge()` 将过滤后的权限与 `['websocket.connect']` 合并
- 使用 `array_unique()` 去重（防止用户已提交该权限）

**设计考量**：
1. **WebSocket 是基础功能**：子用户至少需要能查看控制台输出
2. **简化权限配置**：用户无需手动勾选这个基础权限
3. **前端依赖**：前端 UI 默认需要 WebSocket 连接才能正常工作
4. **历史迁移**：数据库迁移时也会默认注入该权限（见 `2020_03_22_163911_merge_permissions_table_into_subusers.php:92`）

**副作用**：
- 即使用户提交空权限数组，最终也至少有 `websocket.connect`
- 无法创建"完全无权限"的子用户（至少能连接 WebSocket）

### 2.4 与其他校验的配合

`getDefaultPermissions()` 是**最后一道防线**，在此之前还有三层校验：

**完整校验顺序**：
```
1. ClientApiRequest::authorize()  [通过 SubuserRequest::authorize() 调用
   → 校验当前用户有 user.create 或 user.update 权限（通过 $user->can()）

2. SubuserRequest::authorize()
   → 禁止用户编辑自己
   → validatePermissionsCanBeAssigned()：不能分配超出自身权限的权限

3. SubuserController::getDefaultPermissions()  ← 本方法
   → 白名单过滤
   → 强制注入 websocket.connect
   → 去重
```

> **真实调用顺序**：`SubuserRequest::authorize()` 首先调用 `parent::authorize()`（即 `ClientApiRequest::authorize()`），通过后才执行自己的校验逻辑，最后才到控制器的 `getDefaultPermissions()`。

### 2.5 使用场景

该方法在三个地方被调用：

1. **创建子用户** (`store()` 方法第 69 行)
2. **更新子用户权限** (`update()` 方法第 93、113 行)
   - 调用两次：一次用于比较新旧权限，一次用于实际更新
3. **权限变更检测** (`update()` 方法第 93-97 行)
   - 排序后比较，决定是否执行数据库更新和撤销任务

**权限比较逻辑** (`SubuserController.php:93-97`):
```php
$permissions = $this->getDefaultPermissions($request);
$current = $subuser->permissions;

sort($permissions);
sort($current);

// 只有权限真正变化时才执行更新和撤销
if ($permissions !== $current) {
    // ... 执行更新
}
```

---

## 三、API 层逐请求校验机制

API 请求的权限校验是一个多层防御体系，涉及路由中间件、表单请求类、策略类等多个环节。

### 3.1 校验流程总览

```
HTTP 请求
    ↓
[路由层] SubstituteClientBindings 中间件
    → 绑定 server 参数（支持 uuid/uuidShort/identifier）
    → 绑定 user 参数（限定属于该服务器的子用户）
    ↓
[中间层] AuthenticateServerAccess 中间件
    → 验证服务器存在且状态正常
    → 验证用户是所有者/管理员/子用户（否则 404）
    ↓
[请求类层] 具体请求类::authorize()  [如 UpdateSubuserRequest]
    ↓
[子类调用链]
    1. SubuserRequest::authorize()
       → 调用 parent::authorize() → ClientApiRequest::authorize()
       ↓
    2. ClientApiRequest::authorize()
       → 检查是否有 permission() 方法
       → 调用 $user->can(permission(), $server)
       ↓
    3. [策略层] ServerPolicy::before()
       → 管理员/所有者直接放行
       → 子用户检查 in_array(permission, subuser->permissions)
       ↓
    4. 返回 SubuserRequest::authorize() 继续执行
       → 禁止用户编辑自己
       → POST + 有 permissions 字段时
           → validatePermissionsCanBeAssigned()：不能分配超出自身权限的权限
    ↓
控制器执行业务逻辑
```

> **关键顺序说明**：`validatePermissionsCanBeAssigned()` 不是独立的业务逻辑层，而是在请求类 `authorize()` 方法内部，且在 `parent::authorize()` 校验通过后才执行。

### 3.2 路由层：模型绑定与参数校验

`SubstituteClientBindings` 中间件重写了 Laravel 的模型绑定逻辑，确保路由参数始终在正确的上下文中解析。

**关键绑定逻辑** (`app/Http/Middleware/Api/Client/SubstituteClientBindings.php:17-35`):
```php
// server 参数绑定：支持三种格式
$this->router->bind('server', function ($value) {
    return Server::query()
        ->when(
            str_starts_with($value, 'serv_'),
            fn ($builder) => $builder->whereIdentifier($value),
            fn ($builder) => $builder->where(strlen($value) === 8 ? 'uuidShort' : 'uuid', $value)
        )
        ->firstOrFail();
});

// user 参数绑定：必须是该服务器的子用户
$this->router->bind('user', function ($value, $route) {
    $match = $route->parameter('server')
        ->subusers()
        ->whereRelation('user', 'uuid', '=', $value)
        ->firstOrFail();
    return $match->user;
});
```

**路由配置** (`routes/api-client.php:57-129`):
```php
Route::group([
    'prefix' => '/servers/{server}',
    'middleware' => [
        ServerSubject::class,
        AuthenticateServerAccess::class,  // 基础访问校验
        ResourceBelongsToServer::class,   // 资源归属校验
    ],
], function () {
    // 子用户管理路由
    Route::group(['prefix' => '/users'], function () {
        Route::get('/', [SubuserController::class, 'index']);    // user.read
        Route::post('/', [SubuserController::class, 'store']);   // user.create
        Route::get('/{user}', [SubuserController::class, 'view']); // user.read
        Route::post('/{user}', [SubuserController::class, 'update']); // user.update
        Route::delete('/{user}', [SubuserController::class, 'delete']); // user.delete
    });
    // ... 其他服务器资源路由
});
```

### 3.3 中间件层：服务器访问认证

`AuthenticateServerAccess` 中间件执行第一道防线，确保用户至少能"看到"该服务器。

**核心校验逻辑** (`app/Http/Middleware/Api/Client/Server/AuthenticateServerAccess.php:29-67`):
```php
public function handle(Request $request, \Closure $next): mixed
{
    $user = $request->user();
    $server = $request->route()->parameter('server');

    // 基础访问校验：所有者、管理员、或子用户
    if ($user->id !== $server->owner_id && !$user->root_admin) {
        if (!$server->subusers->contains('user_id', $user->id)) {
            throw new NotFoundHttpException(); // 无权限返回 404，避免泄露服务器存在
        }
    }

    // 服务器状态校验（暂停、安装中等）
    try {
        $server->validateCurrentState();
    } catch (ServerStateConflictException $exception) {
        // 对特定路由放行（如查看服务器信息、资源使用）
    }

    return $next($request);
}
```

> **安全设计**：无权限访问时返回 `404 Not Found` 而非 `403 Forbidden`，避免攻击者通过响应状态差异探测服务器存在。

### 3.4 请求类层：具体权限校验

每个 API 端点对应一个 `FormRequest` 类，通过实现 `permission()` 方法声明所需权限。

**接口契约** (`app/Contracts/Http/ClientPermissionsRequest.php:5-12`):
```php
interface ClientPermissionsRequest
{
    public function permission(): string;
}
```

**基类校验逻辑** (`app/Http/Requests/Api/Client/ClientApiRequest.php:17-32`):
```php
public function authorize(): bool
{
    if ($this instanceof ClientPermissionsRequest || method_exists($this, 'permission')) {
        $server = $this->route()->parameter('server');
        if ($server instanceof Server) {
            return $this->user()->can($this->permission(), $server);
        }
        return false;
    }
    return true;
}
```

**示例：发送控制台命令请求** (`app/Http/Requests/Api/Client/Servers/SendCommandRequest.php:8-26`):
```php
class SendCommandRequest extends ClientApiRequest
{
    public function permission(): string
    {
        return Permission::ACTION_CONTROL_CONSOLE;
    }

    public function rules(): array
    {
        return ['command' => 'required|string|min:1'];
    }
}
```

### 3.5 策略层：ServerPolicy 权限判定

Laravel 的授权策略 `ServerPolicy` 实现最终的权限比对逻辑。

**核心判定逻辑** (`app/Policies/ServerPolicy.php:13-43`):
```php
class ServerPolicy
{
    protected function checkPermission(User $user, Server $server, string $permission): bool
    {
        $subuser = $server->subusers->where('user_id', $user->id)->first();
        if (!$subuser || empty($permission)) {
            return false;
        }
        return in_array($permission, $subuser->permissions);
    }

    public function before(User $user, string $ability, Server $server): bool
    {
        // 管理员或服务器所有者 → 直接放行
        if ($user->root_admin || $server->owner_id === $user->id) {
            return true;
        }
        // 子用户 → 检查权限数组
        return $this->checkPermission($user, $server, $ability);
    }

    // __call 魔术方法：避免 Laravel 因方法不存在而跳过 before() 检查
    public function __call(string $name, mixed $arguments)
    {
        // do nothing
    }
}
```

> **技术细节**：`__call` 魔术方法是 Laravel 授权系统的一个"hack"。因为 Laravel 会优先检查策略类是否存在与权限名对应的方法，如果不存在就不会调用 `before()`。通过 `__call` 捕获所有方法调用，确保 `before()` 始终被执行。

### 3.6 请求类层：子用户权限分配的额外校验

在 `SubuserRequest::authorize()` 中，`parent::authorize()` 校验通过后，还会执行额外的业务规则校验。

**真实执行顺序** (`app/Http/Requests/Api/Client/Servers/Subusers/SubuserRequest.php:21-44`):
```php
public function authorize(): bool
{
    // 第一步：先调用父类校验具体权限（如 user.create / user.update）
    if (!parent::authorize()) {
        return false;
    }

    // 第二步：自保护机制 - 禁止用户编辑自己的权限
    $user = $this->route()->parameter('user');
    if ($user instanceof User) {
        if ($user->uuid === $this->user()->uuid) {
            return false;
        }
    }

    // 第三步：权限分配限制（仅 POST 且有 permissions 字段时执行）
    if ($this->method() === Request::METHOD_POST && $this->has('permissions')) {
        $this->validatePermissionsCanBeAssigned(
            $this->input('permissions') ?? []
        );
    }

    return true;
}
```

**权限分配限制逻辑** (`app/Http/Requests/Api/Client/Servers/Subusers/SubuserRequest.php:52-71`):
```php
protected function validatePermissionsCanBeAssigned(array $permissions)
{
    $user = $this->user();
    $server = $this->route()->parameter('server');

    // 管理员或所有者无需校验
    if ($user->root_admin || $user->id === $server->owner_id) {
        return;
    }

    // 子用户：比较待分配权限与自身权限的差集
    $service = $this->container->make(GetUserPermissionsService::class);
    if (count(array_diff($permissions, $service->handle($server, $user))) > 0) {
        throw new HttpForbiddenException('不能分配你自己没有的权限');
    }
}
```

> **执行条件**：`validatePermissionsCanBeAssigned()` 仅在 `POST` 方法且请求包含 `permissions` 字段时执行。由于创建（`POST /users`）和更新（`POST /users/{user}`）子用户都是 POST 请求，因此这两个操作都会触发权限分配限制校验。

---

## 四、权限变更实时生效机制

权限变更的实时生效通过"数据库实时读取 + JWT 短期有效 + 主动撤销通知"三层机制实现。

### 4.1 数据库层面：无缓存，实时读取

权限数据直接存储在 `subusers.permissions` JSON 字段中，**每次请求都从数据库实时读取**，没有额外的缓存层。

**权限读取路径**：
```
API 请求
    → GetUserPermissionsService::handle()
        → $server->subusers()->where('user_id', $user->id)->first()
            → 读取 subusers.permissions 字段
```

这意味着：
- 权限变更在数据库提交后，**下一个 API 请求立即生效**
- 无需额外的缓存失效逻辑
- 牺牲了极少量的性能，换取了一致性和实现简洁性

### 4.2 API 请求层面：逐请求校验

如第三节所述，每个 API 请求都会经过完整的校验流程：
- `AuthenticateServerAccess` 中间件检查子用户身份
- `ClientApiRequest::authorize()` 调用 `ServerPolicy` 检查具体权限
- 所有校验都基于当前数据库状态

### 4.3 WebSocket 层面：JWT 短期有效性

WebSocket 连接使用短期 JWT，确保权限变更能在短时间内生效。

**JWT 生成逻辑** (`app/Http/Controllers/Api/Client/Servers/WebsocketController.php:33-72`):
```php
public function __invoke(ClientApiRequest $request, Server $server): JsonResponse
{
    // 1. 校验 websocket.connect 权限
    if ($user->cannot(Permission::ACTION_WEBSOCKET_CONNECT, $server)) {
        throw new HttpForbiddenException();
    }

    // 2. 获取当前最新权限
    $permissions = $this->permissionsService->handle($server, $user);

    // 3. 生成 JWT，有效期 10 分钟
    $token = $this->jwtService
        ->setExpiresAt(CarbonImmutable::now()->addMinutes(10))
        ->setUser($request->user())
        ->setClaims([
            'server_uuid' => $server->uuid,
            'permissions' => $permissions,  // 权限嵌入 JWT
        ])
        ->handle($node, $user->id . $server->uuid);
}
```

**JWT 结构** (`app/Services/Nodes/NodeJWTService.php:63-102`):
```php
public function handle(Node $node, ?string $identifiedBy, string $algo = 'md5'): UnencryptedToken
{
    $identifier = hash($algo, $identifiedBy);
    $config = Configuration::forSymmetricSigner(new Sha256(), InMemory::plainText($node->getDecryptedKey()));

    $builder = $config->builder()
        ->identifiedBy($identifier)           // JTI：用于撤销
        ->withHeader('jti', $identifier)
        ->expiresAt($this->expiresAt)          // 10 分钟后过期
        ->withClaim('server_uuid', $server->uuid)
        ->withClaim('permissions', $permissions)
        ->withClaim('user_uuid', $this->user->uuid)
        ->withClaim('unique_id', Str::random()) // 每次生成唯一标识
        ->getToken($config->signer(), $config->signingKey());
}
```

**WebSocket 权限校验流程**：
1. 前端调用 `/api/client/servers/{server}/websocket` 获取 JWT
2. JWT 包含当前最新的权限列表，有效期 10 分钟
3. 前端使用 JWT 连接 Wings 节点的 WebSocket
4. Wings 节点验证 JWT 签名和权限
5. JWT 过期后前端需重新获取（携带最新权限）

### 4.4 主动撤销：权限降级实时通知

当权限被**降级**（移除权限或删除子用户）时，系统会主动通知 Wings 节点断开用户连接。

**撤销触发点**：

1. **子用户权限更新** (`app/Http/Controllers/Api/Client/Servers/SubuserController.php:88-125`):
```php
public function update(UpdateSubuserRequest $request, Server $server): array
{
    $subuser = $request->attributes->get('subuser');
    $permissions = $this->getDefaultPermissions($request);
    $current = $subuser->permissions;

    sort($permissions);
    sort($current);

    // 只有权限真正变化时才执行更新和撤销
    if ($permissions !== $current) {
        $log->transaction(function () use ($request, $subuser, $server) {
            $this->repository->update($subuser->id, [
                'permissions' => $this->getDefaultPermissions($request),
            ]);
            // 分发撤销任务
            RevokeSftpAccessJob::dispatch($subuser->user->uuid, $server);
        });
    }
}
```

2. **子用户删除** (`app/Http/Controllers/Api/Client/Servers/SubuserController.php:130-146`):
```php
public function delete(DeleteSubuserRequest $request, Server $server): JsonResponse
{
    $subuser = $request->attributes->get('subuser');
    $log->transaction(function () use ($server, $subuser) {
        $subuser->delete();
        RevokeSftpAccessJob::dispatch($subuser->user->uuid, $server);
    });
}
```

3. **用户删除或密码修改** (`app/Listeners/RevocationListener.php:15-26`):
```php
public function revoke(Deleting|PasswordChanged $event): void
{
    $user = $event->user;
    Node::query()
        ->whereIn('nodes.id', $user->accessibleServers()->select('servers.node_id')->distinct())
        ->chunk(50, function (Collection $nodes) use ($user) {
            $nodes->each(fn (Node $node) => RevokeSftpAccessJob::dispatch($user->uuid, $node));
        });
}
```

**撤销任务执行** (`app/Jobs/RevokeSftpAccessJob.php:41-56`):
```php
public function handle(DaemonRevocationRepository $repository): void
{
    $node = $this->target instanceof Node ? $this->target : $this->target->node;
    try {
        $repository->setNode($node)->deauthorize(
            $this->user,
            $this->target instanceof Server ? [$this->target->uuid] : []
        );
    } catch (DaemonConnectionException) {
        $this->release($this->attempts() * 10); // 指数退避重试
    }
}
```

**Wings 节点通知** (`app/Repositories/Wings/DaemonRevocationRepository.php:17-26`):
```php
public function deauthorize(string $user, array $servers = []): void
{
    $this->getHttpClient()->post('/api/deauthorize-user', [
        'json' => ['user' => $user, 'servers' => $servers],
    ]);
}
```

### 4.5 权限变更生效时间总结

| 场景 | 生效机制 | 生效延迟 |
|------|----------|----------|
| API 请求（HTTP） | 每次请求从数据库实时读取 | 0（数据库提交后立即生效） |
| WebSocket 连接 | JWT 10 分钟后过期，需重新获取 | 最长 10 分钟 |
| 权限降级（移除权限） | 主动通知 Wings 节点断开连接 | 秒级（任务队列调度延迟） |
| 权限升级（添加权限） | 数据库实时读取，WebSocket 下次连接生效 | 0（API），最长 10 分钟（WebSocket） |

---

## 五、关键数据结构与调用关系

### 5.1 核心类关系图

```
User (1) ─── (N) Subuser (N) ─── (1) Server
                │
                └─ permissions (JSON array)
                   存储格式: ["websocket.connect", "control.console", ...]

Permission 类（仅常量定义，无数据库表）
    ├─ ACTION_WEBSOCKET_CONNECT = 'websocket.connect'
    ├─ ACTION_CONTROL_CONSOLE = 'control.console'
    └─ ... (共 40 个常量)

GetUserPermissionsService
    └─ handle(Server, User) → array

ServerPolicy
    ├─ before(User, string, Server) → bool
    └─ checkPermission(User, Server, string) → bool

ClientApiRequest
    └─ authorize() → bool
        └─ user()->can(permission(), server)
            └─ ServerPolicy::before()

SubuserRequest (extends ClientApiRequest)
    ├─ authorize() → bool
    │   ├─ 调用 parent::authorize()  [第一步]
    │   ├─ 禁止用户编辑自己          [第二步]
    │   └─ POST + 有 permissions 时
    │       └─ validatePermissionsCanBeAssigned()  [第三步]
    └─ validatePermissionsCanBeAssigned(array) → void

SubuserController
    └─ getDefaultPermissions(Request) → array
        ├─ 生成权限白名单
        ├─ 白名单过滤
        └─ 强制注入 websocket.connect
```

### 5.2 权限校验完整调用链示例

**场景**：子用户发送控制台命令

```
POST /api/client/servers/{server}/command
    Body: {"command": "say hello"}

1. SubstituteClientBindings::handle()
   → 绑定 server 对象

2. AuthenticateServerAccess::handle()
   → 检查用户是子用户（在 subusers 表中）
   → 检查服务器状态正常

3. SendCommandRequest::authorize()  [extends ClientApiRequest]
   → ClientApiRequest::authorize()
      → 检查是否有 permission() 方法
      → 调用 $user->can('control.console', $server)
         → ServerPolicy::before($user, 'control.console', $server)
             → 用户不是 admin/owner
             → 调用 checkPermission()
                 → 从 $server->subusers 查找该用户
                 → in_array('control.console', $subuser->permissions)
                 → 返回 true

4. CommandController::index()
   → 执行命令发送逻辑
```

### 5.3 权限变更完整调用链示例

**场景**：服务器所有者更新子用户权限

```
POST /api/client/servers/{server}/users/{user}
    Body: {"permissions": ["control.console", "file.read"]}

1. SubstituteClientBindings::handle()
   → 绑定 server 对象
   → 绑定 user 对象（限定为该服务器子用户）

2. AuthenticateServerAccess::handle()
   → 检查用户是所有者/管理员/子用户

3. UpdateSubuserRequest::authorize()  [extends SubuserRequest]
   → SubuserRequest::authorize()
      → 调用 parent::authorize()  [第一步]
         → ClientApiRequest::authorize()
            → $user->can('user.update', $server) → true（所有者）
      → 禁止编辑自己：通过（目标用户不是自己） [第二步]
      → POST 请求有 permissions 字段 [第三步]
         → validatePermissionsCanBeAssigned()
             → 用户是所有者，无需校验

4. SubuserController::update()
   → getDefaultPermissions() 处理权限
      a. 生成白名单（40 个权限）
      b. array_intersect 过滤无效权限
      c. 强制注入 websocket.connect
      d. array_unique 去重
   → 排序新旧权限并比较
   → 权限不同，执行事务：
      a. 更新 subusers.permissions 字段
      b. 分发 RevokeSftpAccessJob

5. RevokeSftpAccessJob 队列执行
   → DaemonRevocationRepository::deauthorize()
      → POST http://node/api/deauthorize-user
         Body: {"user": "user-uuid", "servers": ["server-uuid"]}

6. Wings 节点收到请求
   → 断开该用户的 WebSocket 连接
   → 撤销 SFTP 会话
   → 后续连接需要重新获取 JWT（携带新权限）
```

> **关键调用顺序修正**：`parent::authorize()` 是第一步调用，只有通过后才会执行后续校验。

---

## 六、安全设计亮点

1. **最小权限原则**：权限粒度细（40 个权限位，可精确控制每个操作
2. **权限不升级原则**：不能分配超出自身权限范围的权限
3. **白名单过滤**：权限分配时只保留系统定义的有效权限
4. **强制基础权限**：所有子用户默认拥有 `websocket.connect` 权限
5. **隐身安全**：无权限访问返回 404，避免信息泄露
6. **自保护机制**：禁止用户修改自己的权限
7. **JWT 短期有效**：WebSocket 权限最多 10 分钟后失效
8. **主动撤销**：权限降级时立即通知节点断开连接
9. **事务一致性**：权限更新和撤销任务在同一事务中
10. **任务重试**：节点通知失败时指数退避重试

---

## 七、代码索引

| 功能模块 | 文件路径 |
|----------|----------|
| 权限常量定义（40 个） | `app/Models/Permission.php:18-66` |
| 权限结构定义 | `app/Models/Permission.php:101-209` |
| Subuser 模型 | `app/Models/Subuser.php` |
| 权限获取服务 | `app/Services/Servers/GetUserPermissionsService.php` |
| 子用户创建服务 | `app/Services/Subusers/SubuserCreationService.php` |
| 权限过滤与注入方法 | `app/Http/Controllers/Api/Client/Servers/SubuserController.php:154-168` |
| 服务器访问中间件 | `app/Http/Middleware/Api/Client/Server/AuthenticateServerAccess.php` |
| 模型绑定中间件 | `app/Http/Middleware/Api/Client/SubstituteClientBindings.php` |
| API 请求基类 | `app/Http/Requests/Api/Client/ClientApiRequest.php` |
| 子用户请求基类 | `app/Http/Requests/Api/Client/Servers/Subusers/SubuserRequest.php` |
| 服务器授权策略 | `app/Policies/ServerPolicy.php` |
| WebSocket 控制器 | `app/Http/Controllers/Api/Client/Servers/WebsocketController.php` |
| 子用户控制器 | `app/Http/Controllers/Api/Client/Servers/SubuserController.php` |
| JWT 服务 | `app/Services/Nodes/NodeJWTService.php` |
| 撤销任务 | `app/Jobs/RevokeSftpAccessJob.php` |
| 撤销监听器 | `app/Listeners/RevocationListener.php` |
| Wings 撤销仓库 | `app/Repositories/Wings/DaemonRevocationRepository.php` |
| 子用户观察者 | `app/Observers/SubuserObserver.php` |
| 数据库迁移 | `database/migrations/2020_03_22_163911_merge_permissions_table_into_subusers.php` |
| 路由配置 | `routes/api-client.php:123-129` |
