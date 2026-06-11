# Pterodactyl Panel 用户角色与权限系统深度分析

## 一、角色体系总览

Pterodactyl Panel 采用**三层角色模型**，结合全局管理员开关与单服务器细粒度权限分配：

| 角色类型 | 标识字段 | 权限范围 | 典型场景 |
|---------|---------|---------|---------|
| 管理员 (Admin) | `users.root_admin = true` | 全局所有资源，不受任何限制 | 平台运维、超级管理员 |
| 服务器所有者 (Owner) | `servers.owner_id = user.id` | 单台服务器的全部权限（标记为 `*`） | 用户自己购买/创建的服务器 |
| 子用户 (Subuser) | `subusers` 表关联记录 | 指定服务器的细粒度权限白名单 | 团队协作、代运维人员 |

> **核心设计原则**：管理员和服务器所有者在单台服务器上的权限是等价的（均为 `*` 通配），管理员额外拥有 WebSocket 安装/传输等特殊通道权限。

---

## 二、数据模型与存储结构

### 2.1 User 模型 — 全局角色定义

文件：[User.php](file:///d:/fz/0508-3/solo-dogfeeding/code/205-panel/app/Models/User.php)

```php
class User extends Model
{
    public const USER_LEVEL_USER = 0;   // 普通用户
    public const USER_LEVEL_ADMIN = 1;  // 管理员

    protected $casts = [
        'root_admin' => 'boolean',  // 核心开关字段
    ];

    protected $attributes = [
        'root_admin' => false,       // 默认为普通用户
    ];

    // 获取所有可访问服务器（所有者 + 子用户身份合并）
    public function accessibleServers(): Builder
    {
        return Server::query()
            ->leftJoin('subusers', 'subusers.server_id', '=', 'servers.id')
            ->where(function (Builder $builder) {
                $builder->where('servers.owner_id', $this->id)
                    ->orWhere('subusers.user_id', $this->id);
            })
            ->groupBy('servers.id');
    }
}
```

**关键点**：
- `root_admin` 是唯一的全局权限开关，没有中间角色
- 所有用户默认 `root_admin = false`，仅可通过数据库或管理后台修改
- `accessibleServers()` 通过 LEFT JOIN 合并所有者和子用户两种身份

### 2.2 Subuser 模型 — 服务器级权限绑定

文件：[Subuser.php](file:///d:/fz/0508-3/solo-dogfeeding/code/205-panel/app/Models/Subuser.php)

```php
class Subuser extends Model
{
    protected $casts = [
        'user_id' => 'int',
        'server_id' => 'int',
        'permissions' => 'array',  // JSON 字段，存储权限字符串数组
    ];

    // 反向关联：属于某台服务器
    public function server(): BelongsTo { ... }

    // 反向关联：属于某个用户
    public function user(): BelongsTo { ... }
}
```

**数据库设计**：
- `subusers` 表为三列关系表：`user_id` + `server_id` + `permissions(JSON)`
- 同一用户对同一服务器只能有一条子用户记录（唯一约束由 Service 层校验）
- `permissions` 字段直接存储扁平化字符串数组，如 `["control.start", "file.read"]`

### 2.3 Permission 模型 — 权限字典与常量定义

文件：[Permission.php](file:///d:/fz/0508-3/solo-dogfeeding/code/205-panel/app/Models/Permission.php)

权限按 12 个功能模块组织，共约 40+ 个细粒度权限点：

| 模块 | 权限示例 | 说明 |
|-----|---------|------|
| `websocket` | `websocket.connect` | 连接控制台 WebSocket |
| `control` | `control.start/stop/restart/console` | 电源控制与控制台命令 |
| `user` | `user.read/create/update/delete` | 管理子用户 |
| `file` | `file.read/read-content/create/update/delete/archive/sftp` | 文件管理 |
| `backup` | `backup.read/create/delete/download/restore` | 备份管理 |
| `allocation` | `allocation.read/create/update/delete` | 端口分配 |
| `startup` | `startup.read/update/docker-image` | 启动参数 |
| `database` | `database.read/create/update/delete/view_password` | 数据库 |
| `schedule` | `schedule.read/create/update/delete` | 计划任务 |
| `settings` | `settings.rename/reinstall` | 服务器设置 |
| `activity` | `activity.read` | 操作日志 |

权限字典通过 `Permission::permissions()` 返回，既用于后端校验，也通过 API 暴露给前端做权限配置界面。

---

## 三、权限校验链路（从请求到响应）

### 3.1 整体架构图

```
HTTP Request
    │
    ▼
┌─────────────────────────────────┐
│  路由层中间件 (RouteServiceProvider)
│  ├─ /admin          → AdminAuthenticate (检查 root_admin)
│  ├─ /api/client     → AuthenticateServerAccess (检查 owner/admin/subuser)
│  └─ /api/application → AuthenticateApplicationUser
└─────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────┐
│  FormRequest.authorize()
│  └─ ClientApiRequest → 调用 user()->can(permission, server)
└─────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────┐
│  Gate / Policy (Laravel Authorization)
│  └─ ServerPolicy.before()
│     ├─ root_admin → true (放行)
│     ├─ owner_id匹配 → true (放行)
│     └─ 其他 → 检查 subuser.permissions 数组
└─────────────────────────────────┘
    │
    ▼
Controller Action
    │
    ▼
┌─────────────────────────────────┐
│  Transformer 层（数据裁剪）
│  └─ 如无 startup.read 权限，隐藏变量与启动命令
└─────────────────────────────────┘
```

### 3.2 管理员认证中间件

文件：[AdminAuthenticate.php](file:///d:/fz/0508-3/solo-dogfeeding/code/205-panel/app/Http/Middleware/AdminAuthenticate.php)

```php
class AdminAuthenticate
{
    public function handle(Request $request, \Closure $next): mixed
    {
        // 仅检查单一字段：root_admin
        if (!$request->user() || !$request->user()->root_admin) {
            throw new AccessDeniedHttpException();
        }

        return $next($request);
    }
}
```

**路由挂载位置**（[RouteServiceProvider.php](file:///d:/fz/0508-3/solo-dogfeeding/code/205-panel/app/Providers/RouteServiceProvider.php#L40-L45)）：

```php
// /admin 前缀的所有路由均需通过管理员认证
Route::middleware(['auth.session', RequireTwoFactorAuthentication::class, AdminAuthenticate::class])
    ->prefix('/admin')
    ->group(base_path('routes/admin.php'));
```

### 3.3 客户端服务器访问中间件

文件：[AuthenticateServerAccess.php](file:///d:/fz/0508-3/solo-dogfeeding/code/205-panel/app/Http/Middleware/Api/Client/Server/AuthenticateServerAccess.php)

这是**第一道粗粒度过滤**，只判断用户是否能"看到"这台服务器：

```php
public function handle(Request $request, \Closure $next): mixed
{
    $user = $request->user();
    $server = $request->route()->parameter('server');

    // 三层判断：所有者 → 管理员 → 子用户
    if ($user->id !== $server->owner_id && !$user->root_admin) {
        if (!$server->subusers->contains('user_id', $user->id)) {
            // 非上述三者，返回 404（避免泄露服务器存在性）
            throw new NotFoundHttpException(trans('exceptions.api.resource_not_found'));
        }
    }

    // 同时校验服务器状态（暂停/维护/安装中）
    $server->validateCurrentState();

    return $next($request);
}
```

> **安全细节**：未授权用户收到的是 404 而非 403，防止通过枚举 UUID 探测服务器存在。

### 3.4 ServerPolicy — 细粒度权限裁决

文件：[ServerPolicy.php](file:///d:/fz/0508-3/solo-dogfeeding/code/205-panel/app/Policies/ServerPolicy.php)

Laravel Policy 的 `before()` 钩子是所有权限判定的统一入口：

```php
class ServerPolicy
{
    // 所有权限检查前执行，短路返回
    public function before(User $user, string $ability, Server $server): bool
    {
        // 管理员或所有者：直接放行全部权限
        if ($user->root_admin || $server->owner_id === $user->id) {
            return true;
        }

        // 普通子用户：精确匹配权限字符串
        return $this->checkPermission($user, $server, $ability);
    }

    protected function checkPermission(User $user, Server $server, string $permission): bool
    {
        $subuser = $server->subusers->where('user_id', $user->id)->first();
        if (!$subuser || empty($permission)) {
            return false;
        }

        // permissions 字段是 JSON 解码后的数组，做 in_array 精确匹配
        return in_array($permission, $subuser->permissions);
    }
}
```

**策略注册**（[AuthServiceProvider.php](file:///d:/fz/0508-3/solo-dogfeeding/code/205-panel/app/Providers/AuthServiceProvider.php#L16-L18)）：

```php
protected $policies = [
    Server::class => ServerPolicy::class,
];
```

### 3.5 FormRequest 层 — 权限声明

每个接口通过 FormRequest 的 `permission()` 方法声明所需权限。

**示例 1：电源操作**（[SendPowerRequest.php](file:///d:/fz/0508-3/solo-dogfeeding/code/205-panel/app/Http/Requests/Api/Client/Servers/SendPowerRequest.php)）

```php
class SendPowerRequest extends ClientApiRequest
{
    public function permission(): string
    {
        return match ($this->input('signal')) {
            'start'   => Permission::ACTION_CONTROL_START,
            'stop'|'kill' => Permission::ACTION_CONTROL_STOP,
            'restart' => Permission::ACTION_CONTROL_RESTART,
            default   => '__invalid',
        };
    }
}
```

**示例 2：ClientApiRequest 基类**（[ClientApiRequest.php](file:///d:/fz/0508-3/solo-dogfeeding/code/205-panel/app/Http/Requests/Api/Client/ClientApiRequest.php)）

```php
class ClientApiRequest extends ApplicationApiRequest
{
    public function authorize(): bool
    {
        // 如果请求类实现了 permission() 方法，则自动调用 Gate 校验
        if ($this instanceof ClientPermissionsRequest || method_exists($this, 'permission')) {
            $server = $this->route()->parameter('server');
            if ($server instanceof Server) {
                return $this->user()->can($this->permission(), $server);
            }
            return false;
        }
        return true;
    }
}
```

### 3.6 Transformer 层 — 响应数据裁剪

权限不仅控制"能不能操作"，还控制"能看到什么"。

文件：[ServerTransformer.php](file:///d:/fz/0508-3/solo-dogfeeding/code/205-panel/app/Transformers/Api/Client/ServerTransformer.php)

```php
// 启动命令：无 startup.read 权限时返回空字符串
'invocation' => $service->handle($server, !$user->can(Permission::ACTION_STARTUP_READ, $server)),

// 环境变量：无 startup.read 权限时返回 null
public function includeVariables(Server $server): Collection|NullResource
{
    if (!$this->request->user()->can(Permission::ACTION_STARTUP_READ, $server)) {
        return $this->null();
    }
    return $this->collection($server->variables->where('user_viewable', true), ...);
}

// 端口分配：无 allocation.read 权限时只返回主端口且隐藏备注
public function includeAllocations(Server $server): Collection
{
    if (!$user->can(Permission::ACTION_ALLOCATION_READ, $server)) {
        $primary = clone $server->allocation;
        $primary->notes = null;
        return $this->collection([$primary], $transformer, ...);
    }
    return $this->collection($server->allocations, ...);
}
```

---

## 四、权限的授予流程

### 4.1 用户注册/创建 — 全局角色

文件：[UserCreationService.php](file:///d:/fz/0508-3/solo-dogfeeding/code/205-panel/app/Services/Users/UserCreationService.php)

```php
public function handle(array $data): User
{
    // 密码哈希、UUID 生成等...
    $user = $this->repository->create(array_merge($data, [
        'uuid' => Uuid::uuid4()->toString(),
    ]), true, true);

    // 如未设置密码，生成重置令牌并发送邮件
    $user->notify(new AccountCreated($user, $token ?? null));

    return $user;
}
```

**管理员授予**：
- 注册流程 `root_admin` 始终为 `false`（默认值）
- 只能通过管理后台手动编辑用户或直接修改数据库授予管理员权限
- CLI 命令 `php artisan p:user:make` 可交互式创建管理员

### 4.2 子用户邀请 — 服务器级权限

文件：[SubuserCreationService.php](file:///d:/fz/0508-3/solo-dogfeeding/code/205-panel/app/Services/Subusers/SubuserCreationService.php)

```php
public function handle(Server $server, string $email, array $permissions): Subuser
{
    return $this->connection->transaction(function () use ($server, $email, $permissions) {
        try {
            // 1. 邮箱已注册 → 直接使用该用户
            $user = $this->userRepository->findFirstWhere([['email', '=', $email]]);

            // 不能把服务器所有者加为子用户
            if ($server->owner_id === $user->id) {
                throw new UserIsServerOwnerException(...);
            }
            // 不能重复添加
            if ($subuserCount !== 0) {
                throw new ServerSubuserExistsException(...);
            }
        } catch (RecordNotFoundException) {
            // 2. 邮箱未注册 → 自动创建新用户（默认 root_admin=false）
            $user = $this->userCreationService->handle([
                'email' => $email,
                'username' => substr(...) . Str::random(3),
                'name_first' => 'Server',
                'name_last' => 'Subuser',
                'root_admin' => false,
            ]);
        }

        // 3. 创建子用户记录，写入权限数组
        return $this->subuserRepository->create([
            'user_id' => $user->id,
            'server_id' => $server->id,
            'permissions' => array_unique($permissions),
        ]);
    });
}
```

### 4.3 权限授予的安全约束

文件：[SubuserRequest.php](file:///d:/fz/0508-3/solo-dogfeeding/code/205-panel/app/Http/Requests/Api/Client/Servers/Subusers/SubuserRequest.php)

子用户创建/更新时，会**限制不能分配超出自身拥有的权限**：

```php
protected function validatePermissionsCanBeAssigned(array $permissions)
{
    $user = $this->user();
    $server = $this->route()->parameter('server');

    // 管理员或所有者：无限制
    if ($user->root_admin || $user->id === $server->owner_id) {
        return;
    }

    // 普通子用户：只能分配自己已有的权限子集
    $service = $this->container->make(GetUserPermissionsService::class);
    if (count(array_diff($permissions, $service->handle($server, $user))) > 0) {
        throw new HttpForbiddenException(
            'Cannot assign permissions to a subuser that your account does not actively possess.'
        );
    }
}
```

### 4.4 权限保存时的自动补充与裁剪

文件：[SubuserController.php](file:///d:/fz/0508-3/solo-dogfeeding/code/205-panel/app/Http/Controllers/Api/Client/Servers/SubuserController.php#L154-L168)

子用户权限在保存前会经过 `getDefaultPermissions()` 方法的三道处理，确保权限列表既合法又完整：

```php
protected function getDefaultPermissions(Request $request): array
{
    // 步骤一：展开权限字典，得到所有合法权限的扁平化数组
    $allowed = Permission::permissions()
        ->map(function ($value, $prefix) {
            return array_map(function ($value) use ($prefix) {
                return "$prefix.$value";
            }, array_keys($value['keys']));
        })
        ->flatten()
        ->all();

    // 步骤二：裁剪 — 仅保留在白名单中存在的权限（丢弃非法/拼写错误的权限）
    $cleaned = array_intersect($request->input('permissions') ?? [], $allowed);

    // 步骤三：自动补充 — 强制追加 websocket.connect（即使前端没传也会加上）
    return array_unique(array_merge($cleaned, [Permission::ACTION_WEBSOCKET_CONNECT]));
}
```

**处理流程详解**：

| 步骤 | 操作 | 作用 | 示例 |
|-----|------|------|------|
| 1. 展开 | 从 `Permission::permissions()` 生成全量合法权限列表 | 建立白名单基准 | `['websocket.connect', 'control.start', ...]` |
| 2. 裁剪 | `array_intersect` 取交集 | 过滤掉不存在的权限（防止注入非法权限字符串） | 传入 `['fake.perm', 'file.read']` → 只剩 `['file.read']` |
| 3. 补充 | `array_merge` 追加 `websocket.connect` | 确保子用户至少能连接控制台（即使未显式授予） | 传入 `[]` → 得到 `['websocket.connect']` |
| 4. 去重 | `array_unique` | 避免重复项 | — |

> **设计意图**：`websocket.connect` 是控制台交互的基础权限，如果没有它子用户进入服务器页面会连不上控制台，体验极差。因此系统强制保证该权限始终存在，属于一种防御性编程。

**更新时的优化**（[SubuserController.php#L93-L118](file:///d:/fz/0508-3/solo-dogfeeding/code/205-panel/app/Http/Controllers/Api/Client/Servers/SubuserController.php#L93-L118)）：

```php
// 排序后比对，只有权限真正变更时才写库 + 吊销会话
if ($permissions !== $current) {
    $log->transaction(function () use ($request, $subuser, $server) {
        $this->repository->update($subuser->id, [
            'permissions' => $this->getDefaultPermissions($request),
        ]);

        // 权限变更 → 触发 SFTP/WebSocket 吊销
        RevokeSftpAccessJob::dispatch($subuser->user->uuid, $server);
    });
}
```

---

## 五、权限的下发（后端 → 前端）

### 5.1 权限下发服务

文件：[GetUserPermissionsService.php](file:///d:/fz/0508-3/solo-dogfeeding/code/205-panel/app/Services/Servers/GetUserPermissionsService.php)

```php
public function handle(Server $server, User $user): array
{
    // 管理员或所有者 → 返回通配符 * + 管理员专属 WebSocket 权限
    if ($user->root_admin || $user->id === $server->owner_id) {
        $permissions = ['*'];

        if ($user->root_admin) {
            $permissions[] = 'admin.websocket.errors';
            $permissions[] = 'admin.websocket.install';
            $permissions[] = 'admin.websocket.transfer';
        }

        return $permissions;
    }

    // 子用户 → 返回数据库中存储的精确权限数组
    $subuserPermissions = $server->subusers()->where('user_id', $user->id)->first();
    return $subuserPermissions ? $subuserPermissions->permissions : [];
}
```

### 5.2 服务器详情接口附带权限

文件：[ServerController.php](file:///d:/fz/0508-3/solo-dogfeeding/code/205-panel/app/Http/Controllers/Api/Client/Servers/ServerController.php)

```php
public function index(GetServerRequest $request, Server $server): array
{
    return $this->fractal->item($server)
        ->transformWith($this->getTransformer(ServerTransformer::class))
        ->addMeta([
            'is_server_owner'  => $request->user()->id === $server->owner_id,
            'user_permissions' => $this->permissionsService->handle($server, $request->user()),
        ])
        ->toArray();
}
```

### 5.3 前端接收与存储

文件：[getServer.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/205-panel/resources/scripts/api/server/getServer.ts#L94-L105)

```typescript
// 前端解析响应 meta
http.get(`/api/client/servers/${uuid}`).then(({ data }) =>
    resolve([
        rawDataToServerObject(data),
        // 所有者标记为 ['*']，否则使用具体权限列表
        data.meta?.is_server_owner ? ['*'] : data.meta?.user_permissions || [],
    ])
);
```

前端权限存入 [ServerContext](file:///d:/fz/0508-3/solo-dogfeeding/code/205-panel/resources/scripts/state/server/index.ts#L13-L65)（Easy Peasy Store）：

```typescript
interface ServerDataStore {
    permissions: string[];
    setPermissions: Action<ServerDataStore, string[]>;
}
```

### 5.4 前端权限组件

文件：[Can.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/205-panel/resources/scripts/components/elements/Can.tsx) 和 [usePermissions.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/205-panel/resources/scripts/plugins/usePermissions.ts)

```tsx
// 用法示例：<Can action="file.read">...</Can>
const Can = ({ action, matchAny = false, renderOnError, children }: Props) => {
    const can = usePermissions(action);
    return (
        <>
            {(matchAny && can.filter(p => p).length > 0) || (!matchAny && can.every(p => p))
                ? children
                : renderOnError}
        </>
    );
};

// 权限判定 Hook
export const usePermissions = (action: string | string[]): boolean[] => {
    const userPermissions = ServerContext.useStoreState(s => s.server.permissions);

    return useDeepCompareMemo(() => {
        // 通配符：管理员或所有者拥有全部权限
        if (userPermissions[0] === '*') {
            return Array(Array.isArray(action) ? action.length : 1).fill(true);
        }

        return (Array.isArray(action) ? action : [action]).map(
            permission =>
                // 前缀通配：如 'file.*' 匹配所有 file.xxx 权限
                (permission.endsWith('.*') &&
                    userPermissions.filter(p => p.startsWith(permission.split('.')[0])).length > 0) ||
                // 精确匹配
                userPermissions.indexOf(permission) >= 0
        );
    }, [action, userPermissions]);
};
```

> **前端优化**：支持 `file.*` 这种模块级通配检测，但后端只存储精确权限字符串（或 `*`）。

### 5.5 系统权限字典接口

前端还会通过 `/api/client/permissions` 获取完整权限字典用于渲染权限配置面板：

文件：[ClientController.php](file:///d:/fz/0508-3/solo-dogfeeding/code/205-panel/app/Http/Controllers/Api/Client/ClientController.php#L72-L80)

```php
public function permissions(): array
{
    return [
        'object' => 'system_permissions',
        'attributes' => [
            'permissions' => Permission::permissions(),
        ],
    ];
}
```

---

## 六、管理员 vs 服务器所有者 — 逐场景判断分支剖析

虽然管理员和服务器所有者在 `ServerPolicy` 中都被短路返回 `true`，但两者在多个场景下存在实质性区别。以下逐场景追踪代码中的每一条判断分支。

### 6.1 管理后台访问权

最根本的差异：**只有管理员能进入 `/admin` 管理后台**。

- 管理员：可访问管理后台全部功能（用户、节点、服务器、Nest/Egg、设置等）
- 服务器所有者：无权访问 `/admin`，只能使用客户端界面

管理后台路由文件：[admin.php](file:///d:/fz/0508-3/solo-dogfeeding/code/205-panel/routes/admin.php)，全部由 `AdminAuthenticate` 中间件统一保护，内部不做进一步细粒度区分——所有管理员权限等价。

### 6.2 服务器异常状态下的访问 — 完整判断分支

文件：[AuthenticateServerAccess.php](file:///d:/fz/0508-3/solo-dogfeeding/code/205-panel/app/Http/Middleware/Api/Client/Server/AuthenticateServerAccess.php)

这是理解管理员与所有者差异最关键的一段代码。中间件执行两步：第一步检查身份，第二步检查服务器状态。我们逐步拆解。

#### 第一步：身份检查（第 42-47 行）

```php
if ($user->id !== $server->owner_id && !$user->root_admin) {
    if (!$server->subusers->contains('user_id', $user->id)) {
        throw new NotFoundHttpException(...);  // 404，不泄露服务器存在
    }
}
```

三种角色在此分路：
- 服务器所有者 → `owner_id` 匹配，跳过
- 管理员 → `root_admin` 为 true，跳过
- 子用户 → 需要在 `subusers` 集合中找到记录

#### 第二步：服务器状态检查（第 49-62 行）

先调用 `Server::validateCurrentState()`，该方法（[Server.php#L390-L401](file:///d:/fz/0508-3/solo-dogfeeding/code/205-panel/app/Models/Server.php#L390-L401)）在以下任一条件为真时抛出 `ServerStateConflictException`：

```php
public function validateCurrentState()
{
    if (
        $this->isSuspended()                          // status === 'suspended'
        || $this->node->isUnderMaintenance()          // 节点维护中
        || !$this->isInstalled()                      // status === 'installing' || 'install_failed'
        || $this->status === self::STATUS_RESTORING_BACKUP  // 正在恢复备份
        || !is_null($this->transfer)                  // 正在迁移
    ) {
        throw new ServerStateConflictException($this);
    }
}
```

异常被捕获后，进入三级判断分支：

```php
catch (ServerStateConflictException $exception) {
    // 分支 A：查看服务器详情的请求 → 所有角色放行
    if (!$request->routeIs('api:client:server.view')) {

        // 分支 B：暂停或节点维护时 → 仅资源查询路由可能放行
        if (($server->isSuspended() || $server->node->isUnderMaintenance()) 
            && !$request->routeIs('api:client:server.resources')) {
            throw $exception;   // B1: 不是资源查询 → 全部拦截
        }

        // 分支 C：安装中/恢复备份/迁移中 → 仅管理员访问 WebSocket 可能放行
        if (!$user->root_admin || !$request->routeIs($this->except)) {
            throw $exception;   // C1: 非管理员 或 非例外路由 → 拦截
        }
        // C2: 管理员 + 例外路由（websocket）→ 放行
    }
    // A1: 查看详情 → 放行（所有角色）
}
```

其中 `$except` 数组定义为：

```php
protected array $except = [
    'api:client:server.ws',   // WebSocket 令牌获取接口
];
```

**完整的决策树（5 种服务器状态 × 3 种角色 × 3 类路由）**：

```
服务器状态异常（validateCurrentState 抛异常）
│
├─ 请求路由 = api:client:server.view（查看服务器详情）
│   └─ ✅ 所有角色放行（所有人可看到服务器基本信息）
│
├─ 请求路由 = api:client:server.resources（资源利用率）
│   ├─ 服务器暂停 或 节点维护中
│   │   └─ ✅ 所有角色放行（可看到 CPU/内存占用）
│   └─ 安装中 / 恢复备份 / 迁移中
│       ├─ 管理员 → ✅ 进入下一级判断
│       │   └─ 请求路由 = api:client:server.ws → ✅ 放行
│       │       （其他路由 → ❌ 拦截）
│       └─ 所有者 / 子用户 → ❌ 拦截
│
├─ 请求路由 = api:client:server.ws（WebSocket 令牌）
│   ├─ 服务器暂停 或 节点维护中
│   │   └─ ❌ 所有角色拦截（WebSocket 不是 resources 路由）
│   └─ 安装中 / 恢复备份 / 迁移中
│       ├─ 管理员 → ✅ 放行（可看安装进度/迁移日志）
│       └─ 所有者 / 子用户 → ❌ 拦截
│
└─ 其他所有路由
    └─ ❌ 所有角色拦截
```

**精简总结表**：

| 服务器状态 | 查看详情 | 资源利用率 | WebSocket 令牌 | 其他操作 |
|-----------|---------|-----------|--------------|---------|
| **暂停 (suspended)** | ✅ 全部 | ✅ 全部 | ❌ 全部 | ❌ 全部 |
| **节点维护** | ✅ 全部 | ✅ 全部 | ❌ 全部 | ❌ 全部 |
| **安装中** | ✅ 全部 | ❌ 仅管理员 | ✅ 仅管理员 | ❌ 全部 |
| **安装失败** | ✅ 全部 | ❌ 仅管理员 | ✅ 仅管理员 | ❌ 全部 |
| **恢复备份** | ✅ 全部 | ❌ 仅管理员 | ✅ 仅管理员 | ❌ 全部 |
| **迁移中** | ✅ 全部 | ❌ 仅管理员 | ✅ 仅管理员 | ❌ 全部 |

> **关键洞察**：暂停和节点维护时，管理员也**不能**获取 WebSocket 令牌——这是因为暂停意味着 Wings 端服务器进程已被冻结，WebSocket 连不上。而安装/迁移时管理员能获取 WebSocket，是为了监听安装进度和迁移日志这两个管理员专属通道。

### 6.3 WebSocket 控制台连接 — 管理员专属通道

文件：[WebsocketController.php](file:///d:/fz/0508-3/solo-dogfeeding/code/205-panel/app/Http/Controllers/Api/Client/Servers/WebsocketController.php) 和 [GetUserPermissionsService.php](file:///d:/fz/0508-3/solo-dogfeeding/code/205-panel/app/Services/Servers/GetUserPermissionsService.php#L15-L33)

#### 判断分支 1：基本连接权限

```php
if ($user->cannot(Permission::ACTION_WEBSOCKET_CONNECT, $server)) {
    throw new HttpForbiddenException(...);
}
```

- 管理员 → `ServerPolicy.before()` 短路返回 true → 放行
- 所有者 → `ServerPolicy.before()` 短路返回 true → 放行
- 子用户 → 检查 `websocket.connect` 是否在权限数组中

#### 判断分支 2：迁移状态检查

```php
if (!is_null($server->transfer)) {
    // 分支 A：无 admin.websocket.transfer 权限 → 拒绝
    if (!in_array('admin.websocket.transfer', $permissions)) {
        throw new HttpForbiddenException(...);
    }
    // 分支 B：已归档 → 重定向到新节点的 WebSocket
    if ($server->transfer->archived) {
        $node = $server->transfer->newNode;
    }
}
```

`$permissions` 由 `GetUserPermissionsService` 生成，三种角色差异如下：

```php
// 管理员
['*', 'admin.websocket.errors', 'admin.websocket.install', 'admin.websocket.transfer']

// 服务器所有者
['*']

// 子用户
['websocket.connect', 'control.start', ...]  // 仅实际分配的权限
```

**迁移中的 WebSocket 连接决策**：

| 角色 | `admin.websocket.transfer` 在权限中？ | 结果 |
|-----|--------------------------------------|------|
| 管理员 | ✅（显式追加） | 放行 + 可能重定向到新节点 |
| 服务器所有者 | ❌（`*` 不包含 `admin.websocket.transfer`） | **拒绝** |
| 子用户 | ❌ | **拒绝** |

> **`*` 通配符不匹配 `admin.websocket.*`**：`in_array('admin.websocket.transfer', ['*'])` 返回 `false`。这是 `WebsocketController` 直接用 `in_array` 而非 `Gate::check` 的结果——所有者的 `*` 通配在 Policy 层有效，但在这里的显式检查中无效。

#### 判断分支 3：安装进度通道

安装中的服务器，中间件层面管理员能通过（见 6.2 的分支 C2）。到达 `WebsocketController` 后，JWT 中的 `permissions` 包含 `admin.websocket.install`，Wings 端据此开放安装日志流。

**各通道的权限控制**：

| WebSocket 通道 | 管理员 | 所有者 | 子用户 |
|---------------|-------|-------|-------|
| 控制台输出 (console) | ✅ | ✅ | ⚠️ 需 `control.console` |
| 服务器状态 (status) | ✅ | ✅ | ✅ 连接即可见 |
| 错误日志 (admin errors) | ✅ `admin.websocket.errors` | ❌ | ❌ |
| 安装进度 (install logs) | ✅ `admin.websocket.install` | ❌ | ❌ |
| 迁移进度 (transfer logs) | ✅ `admin.websocket.transfer` | ❌ | ❌ |

### 6.4 服务器操作权限

管理后台的服务器操作全部走管理员专属接口，不经过 `ServerPolicy`：

| 操作 | 管理员（管理后台） | 服务器所有者（客户端） |
|-----|-------------------|---------------------|
| 修改所有者 | ✅ | ❌ |
| 修改配置（内存/CPU/磁盘） | ✅ | ❌ |
| 切换节点（迁移） | ✅ | ❌ |
| 暂停/恢复服务器 | ✅ [SuspensionService](file:///d:/fz/0508-3/solo-dogfeeding/code/205-panel/app/Services/Servers/SuspensionService.php#L28-L60) | ❌ |
| 删除服务器 | ✅ | ❌ |
| 切换 Nest/Egg | ✅ | ❌ |
| 强制重装系统 | ✅ | ⚠️ 仅可触发（受 `settings.reinstall` 限制） |

### 6.5 SFTP 鉴权差异

文件：[SftpAuthenticationController.php#L141-L154](file:///d:/fz/0508-3/solo-dogfeeding/code/205-panel/app/Http/Controllers/Api/Remote/SftpAuthenticationController.php#L141-L154)

```php
protected function validateSftpAccess(User $user, Server $server): void
{
    if (!$user->root_admin && $server->owner_id !== $user->id) {
        $permissions = $this->permissions->handle($server, $user);
        if (!in_array(Permission::ACTION_FILE_SFTP, $permissions)) {
            throw new HttpForbiddenException(...);
        }
    }
    $server->validateCurrentState();  // ← 注意：SFTP 也检查服务器状态！
}
```

**SFTP 在异常状态下同样被阻断**：`validateCurrentState()` 会在暂停、维护、安装等状态下拒绝 SFTP 登录，且这里没有中间件层面那种管理员例外放行逻辑。即**SFTP 不区分角色，异常状态下所有角色都不能登录**。

| 角色 | SFTP 访问条件 | 异常状态下 |
|-----|--------------|-----------|
| 管理员 | 直接通过（不检查 `file.sftp`） | ❌ 被阻断 |
| 服务器所有者 | 直接通过（不检查 `file.sftp`） | ❌ 被阻断 |
| 子用户 | 必须有 `file.sftp` 权限 | ❌ 被阻断 |

### 6.6 差异速查表

| 维度 | 管理员 | 服务器所有者 |
|-----|-------|-------------|
| 管理后台访问 | ✅ | ❌ |
| 全局服务器访问 | ✅ 全部 | ✅ 仅自己的 |
| 暂停/维护时查看资源 | ✅ | ✅ |
| 安装/迁移时获取 WebSocket | ✅ | ❌ |
| 安装/迁移时查看资源 | ✅ | ❌ |
| 迁移中 WebSocket 连接 | ✅ 含迁移日志 | ❌ 被拒绝 |
| 暂停后 SFTP 登录 | ❌ | ❌ |
| 子用户权限分配范围 | 全局无限制 | 本服务器无限制 |
| 修改服务器配置/迁移 | ✅ | ❌ |

---

## 七、管理后台 vs 客户端界面的权限隔离

### 7.1 管理后台（/admin）

路由文件：[admin.php](file:///d:/fz/0508-3/solo-dogfeeding/code/205-panel/routes/admin.php)

| 模块 | 功能 | 权限条件 |
|-----|------|---------|
| 仪表盘 | `/admin` 首页 | `root_admin = true` |
| 用户 | 列表/创建/编辑/删除 | `root_admin = true` |
| 服务器 | 列表/创建/构建/启停/重装/删除 | `root_admin = true` |
| 节点 | 列表/创建/配置/分配管理 | `root_admin = true` |
| 位置 | 列表/创建/编辑 | `root_admin = true` |
| Nest/Egg | 列表/创建/导入导出/变量 | `root_admin = true` |
| 数据库主机 | 列表/创建/编辑 | `root_admin = true` |
| 挂载 | 列表/创建/关联节点与 Egg | `root_admin = true` |
| 设置 | 常规/邮件/高级设置 | `root_admin = true` |
| API 密钥 | 管理员 API Key 管理 | `root_admin = true` |

**全部由 `AdminAuthenticate` 中间件统一保护**，管理后台内部不做进一步细粒度区分——所有管理员权限等价。

### 7.2 客户端界面（/）

路由文件：[api-client.php](file:///d:/fz/0508-3/solo-dogfeeding/code/205-panel/routes/api-client.php)

客户端功能全部受 `AuthenticateServerAccess` 中间件 + `ServerPolicy` 细粒度控制：

| 功能模块 | 必需权限 | 说明 |
|---------|---------|------|
| 控制台查看 | `websocket.connect` | 连接 WebSocket |
| 发送命令 | `control.console` | 向服务器输入命令 |
| 开机/关机/重启 | `control.start/stop/restart` | 电源控制 |
| 文件列表 | `file.read` | 只能看目录结构 |
| 文件内容/下载 | `file.read-content` | 查看和下载文件 |
| 文件上传/创建/修改/删除 | `file.create/update/delete` | 文件写入操作 |
| 备份查看/创建/下载/恢复 | `backup.read/create/download/restore` | 备份全生命周期 |
| 数据库管理 | `database.*` | 含 view_password 特殊权限 |
| 计划任务 | `schedule.*` | Cron 定时任务 |
| 子用户管理 | `user.read/create/update/delete` | 只能分配自身已有权限 |
| 启动参数 | `startup.read/update/docker-image` | 含 Docker 镜像切换 |
| 服务器设置 | `settings.rename/reinstall` | 重命名与重装 |
| 操作日志 | `activity.read` | 审计日志查看 |

### 7.3 Application API（/api/application）

这是**管理员的程序化接口**，供外部系统集成使用。权限机制与 Web 管理后台不同：

文件：[AdminAcl.php](file:///d:/fz/0508-3/solo-dogfeeding/code/205-panel/app/Services/Acl/Api/AdminAcl.php)

```php
class AdminAcl
{
    public const NONE = 0;
    public const READ = 1;   // 位 0
    public const WRITE = 2;  // 位 1

    // 可独立授权的资源
    public const RESOURCE_SERVERS = 'servers';
    public const RESOURCE_NODES = 'nodes';
    public const RESOURCE_USERS = 'users';
    public const RESOURCE_NESTS = 'nests';
    public const RESOURCE_EGGS = 'eggs';
    // ... 共 9 种资源

    // 按位与判断权限
    public static function can(int $permission, int $action = self::READ): bool
    {
        return (bool)($permission & $action);
    }

    // 读取 api_keys 表的 r_{resource} 字段
    public static function check(ApiKey $key, string $resource, int $action = self::READ): bool
    {
        return self::can(data_get($key, self::COLUMN_IDENTIFIER . $resource, self::NONE), $action);
    }
}
```

> Application API Key 按资源维度独立授权（读/写/无），与 Web 会话的管理员权限互不影响。即使是管理员，创建的 API Key 默认也是 NONE，需手动勾选资源权限。

---

## 八、WebSocket 完整生命周期 — 签发、校验、续期、吊销与重授权

### 8.1 JWT 令牌签发 — 完整数据流

文件：[WebsocketController.php](file:///d:/fz/0508-3/solo-dogfeeding/code/205-panel/app/Http/Controllers/Api/Client/Servers/WebsocketController.php#L33-L72) 和 [NodeJWTService.php](file:///d:/fz/0508-3/solo-dogfeeding/code/205-panel/app/Services/Nodes/NodeJWTService.php#L60-L102)

```
前端 getWebsocketToken(uuid)
    │  GET /api/client/servers/{uuid}/websocket
    │
    ▼
AuthenticateServerAccess 中间件
    ├─ 身份检查：owner / root_admin / subuser
    └─ 状态检查（参见 6.2 完整决策树）
        ├─ 暂停/维护 → ❌ 拦截（WebSocket 不在例外路由中）
        └─ 安装/迁移 → ✅ 仅管理员放行
    │
    ▼
WebsocketController.__invoke()
    ├─ Gate::check(websocket.connect, server)
    │   ├─ 管理员 → true
    │   ├─ 所有者 → true
    │   └─ 子用户 → in_array('websocket.connect', permissions)
    │
    ├─ GetUserPermissionsService.handle(server, user)
    │   ├─ 管理员 → ['*', 'admin.websocket.errors', 'admin.websocket.install', 'admin.websocket.transfer']
    │   ├─ 所有者 → ['*']
    │   └─ 子用户 → ['websocket.connect', 'control.start', ...]
    │
    ├─ 迁移检查
    │   ├─ server.transfer 为 null → 跳过
    │   └─ server.transfer 非 null
    │       ├─ in_array('admin.websocket.transfer', permissions) → false → ❌ 403
    │       └─ true → 继续
    │           └─ transfer.archived → 重定向到 newNode
    │
    └─ NodeJWTService 签发 JWT
        ├─ 签名密钥：节点的 getDecryptedKey()
        ├─ jti: md5(user_id + server_uuid)   ← 吊销时按此标识
        ├─ exp: 当前时间 + 10 分钟
        ├─ nbf: 当前时间 - 5 分钟（容忍时钟偏差）
        ├─ claims:
        │   ├─ server_uuid: 服务器 UUID
        │   ├─ permissions: 权限数组（快照！）
        │   └─ user_uuid: 用户 UUID
        └─ unique_id: Str::random()（每次签发唯一）
    │
    ▼
返回 { token: JWT字符串, socket: wss://node:port/api/servers/uuid/ws }
```

### 8.2 WebSocket 连接建立 — 从前端到 Wings

文件：[Websocket.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/205-panel/resources/scripts/plugins/Websocket.ts) 和 [WebsocketHandler.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/205-panel/resources/scripts/components/server/WebsocketHandler.tsx)

```
前端 WebsocketHandler 组件挂载
    │
    ▼
connect(uuid)
    │
    ├─ new Websocket()
    │   └─ 注册事件处理器（auth success / token expiring / jwt error / ...）
    │
    ├─ getWebsocketToken(uuid)  ← 第一次请求 /websocket
    │   └─ 返回 { token, socket }
    │
    └─ socket.setToken(token).connect(socketUrl)
        │
        ▼
    WebSocket 连接到 wss://node:port/api/servers/uuid/ws
        │
        ▼
    onopen 触发 → authenticate()
        │  发送 { event: "auth", args: [JWT_TOKEN] }
        │
        ▼
    Wings 验证 JWT
        ├─ 检查签名（使用节点密钥）
        ├─ 检查 exp（未过期）
        ├─ 检查 jti（不在 denylist 中）
        ├─ 检查 nbf ≤ 当前时间
        └─ 读取 permissions 建立会话权限上下文
        │
        ▼
    Wings 返回 { event: "auth success", args: [...] }
        │
        ▼
    前端 setConnectionState(true) → 连接成功
```

### 8.3 JWT 自动续期 — 不中断的令牌轮换

文件：[WebsocketHandler.tsx#L20-L30](file:///d:/fz/0508-3/solo-dogfeeding/code/205-panel/resources/scripts/components/server/WebsocketHandler.tsx#L20-L30) 和 [Websocket.ts#L62-L76](file:///d:/fz/0508-3/solo-dogfeeding/code/205-panel/resources/scripts/plugins/Websocket.ts#L62-L76)

```
JWT 生命周期（10 分钟有效期）
    │
    ├─ T+0min  签发
    ├─ T+7min  Wings 推送 'token expiring' 事件（提前 3 分钟警告）
    ├─ T+10min JWT 过期
    │
    ▼
续期触发条件（任一满足即触发 updateToken）：

1. 'token expiring'  → Wings 主动提醒
2. 'token expired'   → JWT 已过期
3. 'jwt error' 且错误消息包含以下之一：
   ├─ 'jwt: exp claim is invalid'         ← 过期
   └─ 'jwt: created too far in past (denylist)'  ← 被吊销

续期流程：
    │
    ▼
updateToken(uuid, socket)
    │  if (updatingToken) return;   ← 防止并发续期
    │  updatingToken = true;
    │
    ├─ getWebsocketToken(uuid)      ← 重新请求 /api/client/servers/{uuid}/websocket
    │   │
    │   ├─ Panel 重新鉴权（中间件 + Gate + 权限服务）
    │   │   ├─ 权限已被撤销 → ❌ 403 → .catch(error)
    │   │   └─ 权限仍在 → ✅ 返回新 JWT（含最新权限快照）
    │   │
    │   └─ 新 JWT 的 jti 与旧 JWT 相同（md5(user_id+server_uuid)）
    │       ← 这意味着旧 JWT 被吊销后，新 JWT 的 jti 也会
    │         被加入 denylist，导致"吊销-重签-又被吊销"死循环
    │
    └─ socket.setToken(newToken, isUpdate=true)
        │  this.token = newToken;
        └─ this.authenticate()
           发送 { event: "auth", args: [NEW_JWT] }
           │
           ▼
        Wings 验证新 JWT → 'auth success' 或 'jwt error'
```

> **关于 jti 的关键细节**：JWT 的 jti 由 `md5(user_id + server_uuid)` 计算得出，**同一用户对同一服务器的所有 JWT 共享同一 jti**。这意味着当 Wings 收到 `/api/deauthorize-user` 请求后，会将该 jti 加入 denylist，**后续用同一 jti 签发的新 JWT 也会被拒绝**。实际上 Wings 的吊销机制是按 user_uuid 而非 jti 运作的——收到 deauthorize 请求后，Wings 会清除该用户的全部会话和 jti 记录，因此重签的新 JWT（即使 jti 相同）可以重新通过验证。

### 8.4 JWT 被吊销后的重连与重授权路径

当 Panel 调用 `RevokeSftpAccessJob` 后，以下链路触发：

```
Panel 端
    │
    ├─ SubuserController.update() → RevokeSftpAccessJob::dispatch(user_uuid, server)
    ├─ SubuserController.delete() → RevokeSftpAccessJob::dispatch(user_uuid, server)
    └─ RevocationListener.revoke() → RevokeSftpAccessJob::dispatch(user_uuid, node)
    │
    ▼
RevokeSftpAccessJob（队列异步执行，最多重试 3 次）
    │
    ▼
DaemonRevocationRepository.deauthorize(user_uuid, servers)
    │  POST /api/deauthorize-user
    │  Body: { "user": "user_uuid", "servers": ["server_uuid"] 或 [] }
    │
    ▼
Wings 端收到 deauthorize-user 请求
    │
    ├─ 立即断开该用户在此节点上的所有 SFTP 会话
    ├─ 将该用户的所有 JWT 加入 denylist（按 user_uuid 匹配）
    └─ 对活跃的 WebSocket 连接推送 'jwt error' 事件
        │  消息：'jwt: created too far in past (denylist)'
        │
        ▼
    前端收到 'jwt error' 事件
        │
        ▼
    WebsocketHandler 判断错误类型
        │
        ├─ 错误包含 'denylist' → 匹配 reconnectErrors
        │   │
        │   ▼
        │   updateToken(uuid, socket)
        │       │
        │       ├─ GET /api/client/servers/{uuid}/websocket
        │       │   │
        │       │   ├─ 场景 A：子用户权限被修改（未删除）
        │       │   │   → 中间件放行 → WebsocketController 签发新 JWT
        │       │   │   → 新 JWT 含最新权限 → Wings 接受 → ✅ 重连成功
        │       │   │
        │       │   ├─ 场景 B：子用户被删除 / websocket.connect 被撤销
        │       │   │   → Gate::check 返回 false → ❌ 403
        │       │   │   → getWebsocketToken catch(error) → console.error
        │       │   │   → 更新 updatingToken = false → 不会再尝试
        │       │   │   → WebSocket 保持断开状态
        │       │   │
        │       │   └─ 场景 C：服务器被暂停
        │       │       → AuthenticateServerAccess 拦截 → ❌ ServerStateConflictException
        │       │       → getWebsocketToken catch(error) → 同上
        │       │
        │       └─ 结果：
        │           ├─ 权限仍在 → ✅ 无缝重连（新 JWT 含最新权限）
        │           └─ 权限已撤销 → ❌ 永久断开（前端不再尝试）
        │
        └─ 错误不包含 denylist
            → setError('凭证验证错误，请刷新页面') → 用户需手动操作
```

### 8.5 服务器暂停后的 WebSocket 行为

服务器被暂停（[SuspensionService](file:///d:/fz/0508-3/solo-dogfeeding/code/205-panel/app/Services/Servers/SuspensionService.php#L28-L60)）后，WebSocket 的完整行为链路：

```
管理员在管理后台暂停服务器
    │
    ├─ Panel DB: server.status = 'suspended'
    └─ Wings: daemonServerRepository.sync() → Wings 冻结服务器进程
    │
    ▼
Wings 主动关闭该服务器的所有 WebSocket 连接
    │  关闭码: 4409（Suspended）
    │
    ▼
前端 Websocket.ts onreconnect 处理
    │  evt.code === 4409 → this.close(1000)  ← 不再自动重连
    │
    ▼
前端显示红色提示条：'连接中断...'

此时如果用户尝试刷新页面：
    │
    ▼
前端 getWebsocketToken(uuid)
    │  GET /api/client/servers/{uuid}/websocket
    │
    ▼
AuthenticateServerAccess 中间件
    ├─ validateCurrentState() → isSuspended() = true → 抛异常
    └─ 捕获后检查路由
        ├─ 路由 = api:client:server.ws（在 $except 中）
        ├─ 但 isSuspended() = true → 进入分支 B
        │   └─ 路由不是 resources → ❌ 拦截
        └─ 结果：所有角色（包括管理员）都被拦截
```

> **暂停 vs 安装/迁移的关键区别**：暂停时管理员也不能获取 WebSocket 令牌，因为 Wings 端服务器进程已冻结，WebSocket 服务不可用。而安装/迁移时 Wings 仍运行着 WebSocket 服务，只是频道内容不同。

### 8.6 迁移过程中的 WebSocket 重连

文件：[WebsocketHandler.tsx#L65-L77](file:///d:/fz/0508-3/solo-dogfeeding/code/205-panel/resources/scripts/components/server/WebsocketHandler.tsx#L65-L77)

迁移是一个多阶段过程，WebSocket 需要跨节点重连：

```
管理员发起迁移（管理后台 ServerTransferController）
    │
    ├─ 创建 server_transfers 记录
    ├─ 通知源节点开始传输
    └─ 源节点向目标节点推送数据
    │
    ▼
前端收到 'transfer status' 事件
    │
    ├─ status = 'starting' → 忽略（迁移刚开始）
    ├─ status = 'success' → 忽略（迁移完成）
    └─ status = 'archived' 或其他 → 触发重连
        │
        ▼
    socket.close()           ← 关闭旧连接
    setInstance(null)        ← 清空实例
    connect(uuid)            ← 重新建立连接
        │
        ▼
    getWebsocketToken(uuid)  ← 获取新令牌
        │
        ├─ WebsocketController 检查 transfer.archived
        │   └─ node = server.transfer.newNode  ← 切换到目标节点
        │
        └─ 返回新节点的 WebSocket 地址和令牌
            │
            ▼
        连接到新节点的 WebSocket
            └─ 继续接收迁移日志（管理员专属）
```

---

## 九、SFTP 完整生命周期 — 认证、会话与吊销

### 9.1 SFTP 认证流程

文件：[SftpAuthenticationController.php](file:///d:/fz/0508-3/solo-dogfeeding/code/205-panel/app/Http/Controllers/Api/Remote/SftpAuthenticationController.php)

```
用户使用 SFTP 客户端连接（如 FileZilla）
    │  用户名格式：username.server_uuid
    │  密码：用户密码 或 SSH 密钥
    │
    ▼
Wings 收到 SFTP 连接请求
    │  调用 Panel 的 /api/remote/sftp/auth 接口
    │
    ▼
SftpAuthenticationController.__invoke()
    │
    ├─ parseUsername(username)
    │   │  倒序分割 '.' → 避免用户名含点号的问题
    │   │  例：'john.doe.abc123' → username='john.doe', server='abc123'
    │   └─ 返回 { username, server }
    │
    ├─ hasTooManyLoginAttempts(request)
    │   └─ 节流检查（按 username|IP 维度）
    │
    ├─ getUser(request, username)
    │   └─ User::where('username', $username) → 找不到则 reject
    │
    ├─ getServer(request, server_uuid)
    │   └─ Server::where(uuid/uuidShort, $uuid)->where(node_id, $node->id)
    │       └─ 必须属于当前节点（防止跨节点伪造）
    │
    ├─ 验证凭据
    │   ├─ 密码模式：password_verify(input, user.password)
    │   └─ 公钥模式：比对 SSH Key 指纹
    │
    ├─ validateSftpAccess(user, server)
    │   ├─ 管理员或所有者 → 跳过权限检查
    │   ├─ 子用户 → 检查 file.sftp 权限
    │   └─ validateCurrentState() → 检查服务器状态
    │       ├─ 暂停 → ❌ 拒绝（无管理员例外！）
    │       ├─ 节点维护 → ❌ 拒绝
    │       ├─ 安装中 → ❌ 拒绝
    │       ├─ 恢复备份 → ❌ 拒绝
    │       └─ 迁移中 → ❌ 拒绝
    │
    └─ 返回 JSON
        { "user": "user_uuid", "server": "server_uuid", "permissions": [...] }
        │
        ▼
    Wings 建立SFTP 会话
        ├─ 基于 permissions 限制文件操作范围
        └─ 会话持续到客户端断开或被主动吊销
```

### 9.2 SFTP 会话期间权限变更

SFTP 会话一旦建立，**Wings 不会中途重新查询 Panel 的权限**。如果权限在会话期间变更：

```
子用户正在使用 SFTP
    │
    ▼
管理员修改了该子用户的权限
    │  SubuserController.update()
    ├─ 更新 subusers.permissions
    └─ RevokeSftpAccessJob::dispatch(user_uuid, server)
        │
        ▼
    Wings 收到 /api/deauthorize-user
        │
        ├─ 立即断开该用户的 SFTP 会话
        └─ SFTP 客户端收到连接重置
            │
            ▼
        用户需要重新登录 SFTP
            │  重新走完整认证流程（使用最新权限）
```

### 9.3 SFTP 吊销的两种范围

`RevokeSftpAccessJob` 有两种构造方式，对应不同的吊销范围：

| 构造方式 | 调用场景 | servers 参数 | 吊销范围 |
|---------|---------|-------------|---------|
| `dispatch(user_uuid, $server)` | 子用户权限变更/删除 | `[$server->uuid]` | 仅指定服务器 |
| `dispatch(user_uuid, $node)` | 用户删除/密码修改 | `[]`（空） | 该节点上全部服务器 |

```php
// RevokeSftpAccessJob.handle()
$repository->setNode($node)->deauthorize(
    $this->user,    // user_uuid
    $this->target instanceof Server ? [$this->target->uuid] : []  // 空数组=全部
);
```

> 当 `servers` 为空数组时，Wings 吊销该用户在此节点上的**所有** SFTP 会话和 WebSocket JWT。

### 9.4 SFTP 重连后的权限生效

| 权限变更类型 | SFTP 当前会话 | SFTP 重新登录 |
|-----------|-------------|-------------|
| 权限增加 | ❌ 不生效（旧会话） | ✅ 生效（重新查询） |
| 权限减少 | ❌ 不生效（旧会话） | ✅ 生效（重新查询） |
| file.sftp 被撤销 | ✅ 会话被断开 | ❌ 无法登录 |
| 子用户被删除 | ✅ 会话被断开 | ❌ 无法登录 |
| 服务器被暂停 | ✅ 会话被断开 | ❌ 无法登录 |

---

## 十、权限变更后的会话同步 — 全场景总结

### 10.1 完整的吊销触发矩阵

| 触发事件 | 触发代码位置 | 吊销方式 | 影响范围 |
|---------|------------|---------|---------|
| 子用户权限变更 | [SubuserController.update()](file:///d:/fz/0508-3/solo-dogfeeding/code/205-panel/app/Http/Controllers/Api/Client/Servers/SubuserController.php#L110-L117) | `RevokeSftpAccessJob(user, server)` | 单服务器 |
| 子用户删除 | [SubuserController.delete()](file:///d:/fz/0508-3/solo-dogfeeding/code/205-panel/app/Http/Controllers/Api/Client/Servers/SubuserController.php#L140-L144) | `RevokeSftpAccessJob(user, server)` | 单服务器 |
| 用户删除 | [RevocationListener.revoke()](file:///d:/fz/0508-3/solo-dogfeeding/code/205-panel/app/Listeners/RevocationListener.php) | `RevokeSftpAccessJob(user, node)` × N | 全部节点 |
| 密码修改 | [RevocationListener.revoke()](file:///d:/fz/0508-3/solo-dogfeeding/code/205-panel/app/Listeners/RevocationListener.php) | `RevokeSftpAccessJob(user, node)` × N | 全部节点 |
| 管理员身份变更 | 无 | — | **不触发吊销** |
| 服务器所有者变更 | 无 | — | **不触发吊销** |
| 服务器暂停 | [SuspensionService.toggle()](file:///d:/fz/0508-3/solo-dogfeeding/code/205-panel/app/Services/Servers/SuspensionService.php#L28-L60) | Wings sync() | Wings 端主动断开（码 4409） |

### 10.2 各通道的生效时序

| 变更类型 | HTTP API | WebSocket | SFTP | 主动吊销？ |
|---------|---------|-----------|------|-----------|
| 子用户权限变更 | 立即生效 | denylist → jwt error → 重签（几秒内） | 会话断开 → 需重新登录 | ✅ |
| 子用户删除 | 立即生效 | denylist → 重签失败 → 永久断开 | 会话断开 → 登录被拒 | ✅ |
| 用户删除 | 立即生效 | 全部节点 denylist | 全部节点断开 | ✅ |
| 密码修改 | 不变 | 全部节点 denylist | 全部节点断开 | ✅ |
| 管理员身份变更 | 下次请求生效 | 下次续期生效（≤10分钟） | 下次登录生效 | ❌ |
| 服务器所有者变更 | 下次请求生效 | 下次续期生效 | 下次登录生效 | ❌ |
| 服务器暂停 | 立即生效 | Wings 4409 断开（不重连） | 会话断开 | ✅ Wings 端 |
| 服务器恢复 | 立即生效 | 需刷新页面重新连接 | 可重新登录 | — |

### 10.3 前端状态重置

文件：[server/index.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/205-panel/resources/scripts/state/server/index.ts#L99-L115)

前端切换服务器时会清除整个 Store：

```typescript
clearServerState: action((state) => {
    state.server.data = undefined;
    state.server.permissions = [];   // 清空权限数组
    if (state.socket.instance) {
        state.socket.instance.removeAllListeners();
        state.socket.instance.close();  // 关闭 WebSocket 连接
    }
    state.socket.instance = null;
    state.socket.connected = false;
}),
```

每次进入新服务器页面都会重新调用 `getServer()` 获取该服务器的权限数组，确保权限始终与后端同步。

---

## 十一、关键代码速查表

| 功能 | 文件 | 关键位置 |
|-----|------|---------|
| 管理员全局开关 | [User.php](file:///d:/fz/0508-3/solo-dogfeeding/code/205-panel/app/Models/User.php#L39-L103) | `root_admin` 字段与常量 |
| 权限字典定义 | [Permission.php](file:///d:/fz/0508-3/solo-dogfeeding/code/205-panel/app/Models/Permission.php#L18-L218) | 全部权限常量与 `$permissions` 数组 |
| 权限最终裁决 | [ServerPolicy.php](file:///d:/fz/0508-3/solo-dogfeeding/code/205-panel/app/Policies/ServerPolicy.php#L13-L43) | `before()` 和 `checkPermission()` |
| 管理员中间件 | [AdminAuthenticate.php](file:///d:/fz/0508-3/solo-dogfeeding/code/205-panel/app/Http/Middleware/AdminAuthenticate.php#L15-L22) | `handle()` 方法 |
| 服务器状态+身份检查 | [AuthenticateServerAccess.php](file:///d:/fz/0508-3/solo-dogfeeding/code/205-panel/app/Http/Middleware/Api/Client/Server/AuthenticateServerAccess.php#L29-L67) | `handle()` 三级判断分支 |
| 服务器状态校验 | [Server.php](file:///d:/fz/0508-3/solo-dogfeeding/code/205-panel/app/Models/Server.php#L390-L401) | `validateCurrentState()` |
| 迁移前状态校验 | [Server.php](file:///d:/fz/0508-3/solo-dogfeeding/code/205-panel/app/Models/Server.php#L409-L418) | `validateTransferState()` |
| 权限下发给前端 | [GetUserPermissionsService.php](file:///d:/fz/0508-3/solo-dogfeeding/code/205-panel/app/Services/Servers/GetUserPermissionsService.php#L15-L33) | `handle()` — 管理员额外追加 3 个 WebSocket 权限 |
| WebSocket JWT 签发 | [WebsocketController.php](file:///d:/fz/0508-3/solo-dogfeeding/code/205-panel/app/Http/Controllers/Api/Client/Servers/WebsocketController.php#L33-L72) | `__invoke()` — 迁移检查 + 令牌签发 |
| JWT 生成服务 | [NodeJWTService.php](file:///d:/fz/0508-3/solo-dogfeeding/code/205-panel/app/Services/Nodes/NodeJWTService.php#L60-L102) | `handle()` — jti/exp/claims |
| 前端 WebSocket 处理 | [WebsocketHandler.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/205-panel/resources/scripts/components/server/WebsocketHandler.tsx) | `updateToken()` + `connect()` + 事件处理 |
| 前端 WebSocket 传输层 | [Websocket.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/205-panel/resources/scripts/plugins/Websocket.ts) | `authenticate()` + 4409 暂停处理 |
| SFTP 鉴权接口 | [SftpAuthenticationController.php](file:///d:/fz/0508-3/solo-dogfeeding/code/205-panel/app/Http/Controllers/Api/Remote/SftpAuthenticationController.php#L34-L83) | `__invoke()` + `validateSftpAccess()` |
| 会话吊销 Job | [RevokeSftpAccessJob.php](file:///d:/fz/0508-3/solo-dogfeeding/code/205-panel/app/Jobs/RevokeSftpAccessJob.php#L18-L57) | `handle()` — 含重试与唯一 ID |
| Wings 吊销仓库 | [DaemonRevocationRepository.php](file:///d:/fz/0508-3/solo-dogfeeding/code/205-panel/app/Repositories/Wings/DaemonRevocationRepository.php#L17-L26) | `deauthorize()` — POST /api/deauthorize-user |
| 全局会话吊销监听 | [RevocationListener.php](file:///d:/fz/0508-3/solo-dogfeeding/code/205-panel/app/Listeners/RevocationListener.php#L15-L32) | `revoke()` — 用户删除/密码修改 |
| 服务器暂停服务 | [SuspensionService.php](file:///d:/fz/0508-3/solo-dogfeeding/code/205-panel/app/Services/Servers/SuspensionService.php#L28-L60) | `toggle()` — sync Wings |
| 服务器迁移 | [ServerTransferController.php](file:///d:/fz/0508-3/solo-dogfeeding/code/205-panel/app/Http/Controllers/Admin/Servers/ServerTransferController.php#L38-L92) | `transfer()` — 管理员专属 |
| 权限保存前的裁剪与补充 | [SubuserController.php](file:///d:/fz/0508-3/solo-dogfeeding/code/205-panel/app/Http/Controllers/Api/Client/Servers/SubuserController.php#L154-L168) | `getDefaultPermissions()` |
| 权限分配安全校验 | [SubuserRequest.php](file:///d:/fz/0508-3/solo-dogfeeding/code/205-panel/app/Http/Requests/Api/Client/Servers/Subusers/SubuserRequest.php#L52-L71) | `validatePermissionsCanBeAssigned()` |
| API 层权限声明 | [ClientApiRequest.php](file:///d:/fz/0508-3/solo-dogfeeding/code/205-panel/app/Http/Requests/Api/Client/ClientApiRequest.php#L17-L32) | `authorize()` |
| 前端权限判定 Hook | [usePermissions.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/205-panel/resources/scripts/plugins/usePermissions.ts#L4-L21) | `usePermissions()` |
| 前端权限组件 | [Can.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/205-panel/resources/scripts/components/elements/Can.tsx#L12-L22) | `Can` 组件 |
| 路由中间件挂载 | [RouteServiceProvider.php](file:///d:/fz/0508-3/solo-dogfeeding/code/205-panel/app/Providers/RouteServiceProvider.php#L38-L66) | `boot()` → `routes()` |
| 管理员 API Key ACL | [AdminAcl.php](file:///d:/fz/0508-3/solo-dogfeeding/code/205-panel/app/Services/Acl/Api/AdminAcl.php#L40-L56) | `can()` 和 `check()` 方法 |
