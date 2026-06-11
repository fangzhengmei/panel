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

## 六、管理后台 vs 客户端界面的权限隔离

### 6.1 管理后台（/admin）

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

### 6.2 客户端界面（/）

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

### 6.3 Application API（/api/application）

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

## 七、会话与权限同步机制

### 7.1 权限变更的实时性

**核心结论：权限是实时查询的，不做缓存。**

每次请求：
1. `AuthenticateServerAccess` 中间件从 `$server->subusers` 关系读取
2. `ServerPolicy.before()` 再次从 `$server->subusers` 读取
3. `GetUserPermissionsService` 直接查 `subusers()` 关联

**没有任何 Redis/文件缓存层**，修改子用户权限后下一次请求立即生效。

### 7.2 权限变更事件

文件：[SubuserObserver.php](file:///d:/fz/0508-3/solo-dogfeeding/code/205-panel/app/Observers/SubuserObserver.php)

```php
class SubuserObserver
{
    public function created(Subuser $subuser): void
    {
        event(new Events\Subuser\Created($subuser));
        // 发送通知邮件给被邀请用户
        $subuser->user->notify(new AddedToServer([...]));
    }

    public function deleted(Subuser $subuser): void
    {
        event(new Events\Subuser\Deleted($subuser));
        // 发送通知邮件给被移除用户
        $subuser->user->notify(new RemovedFromServer([...]));
    }
}
```

### 7.3 会话吊销机制

文件：[RevocationListener.php](file:///d:/fz/0508-3/solo-dogfeeding/code/205-panel/app/Listeners/RevocationListener.php)

当用户被删除或修改密码时，主动断开其所有节点的 SFTP 和 WebSocket 连接：

```php
class RevocationListener implements SubscribesToEvents
{
    public function revoke(Deleting|PasswordChanged $event): void
    {
        $user = $event->user;

        // 查找用户可访问的所有服务器所在节点
        Node::query()
            ->whereIn('nodes.id', $user->accessibleServers()->select('servers.node_id')->distinct())
            ->chunk(50, function (Collection $nodes) use ($user) {
                // 向每个 Wings 节点派发任务，吊销 user.uuid 的访问
                $nodes->each(fn (Node $node) => RevokeSftpAccessJob::dispatch($user->uuid, $node));
            });
    }
}
```

**触发时机**：
- `User\Deleting` 事件：用户被删除时
- `User\PasswordChanged` 事件：用户修改密码时

> **管理员身份变更**：`root_admin` 字段变更后不会立即吊销现有会话，用户下次请求时新的权限才生效。当前会话的登录状态不会失效（Laravel Session 机制决定），但管理员专属接口会在下一次请求时重新检查 `root_admin`。

### 7.4 前端状态重置

前端切换服务器时会清除整个 Store：

文件：[server/index.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/205-panel/resources/scripts/state/server/index.ts#L99-L115)

```typescript
clearServerState: action((state) => {
    state.server.data = undefined;
    state.server.permissions = [];   // 清空权限数组
    // ... 清空其他子模块状态
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

## 八、关键代码速查表

| 功能 | 文件 | 关键位置 |
|-----|------|---------|
| 管理员全局开关 | [User.php](file:///d:/fz/0508-3/solo-dogfeeding/code/205-panel/app/Models/User.php#L39-L103) | `root_admin` 字段与常量 |
| 权限字典定义 | [Permission.php](file:///d:/fz/0508-3/solo-dogfeeding/code/205-panel/app/Models/Permission.php#L18-L218) | 全部权限常量与 `$permissions` 数组 |
| 权限最终裁决 | [ServerPolicy.php](file:///d:/fz/0508-3/solo-dogfeeding/code/205-panel/app/Policies/ServerPolicy.php#L13-L43) | `before()` 和 `checkPermission()` |
| 管理员中间件 | [AdminAuthenticate.php](file:///d:/fz/0508-3/solo-dogfeeding/code/205-panel/app/Http/Middleware/AdminAuthenticate.php#L15-L22) | `handle()` 方法 |
| 服务器粗粒度鉴权 | [AuthenticateServerAccess.php](file:///d:/fz/0508-3/solo-dogfeeding/code/205-panel/app/Http/Middleware/Api/Client/Server/AuthenticateServerAccess.php#L29-L67) | `handle()` 方法 |
| 权限下发给前端 | [GetUserPermissionsService.php](file:///d:/fz/0508-3/solo-dogfeeding/code/205-panel/app/Services/Servers/GetUserPermissionsService.php#L15-L33) | `handle()` 方法 |
| 子用户创建 | [SubuserCreationService.php](file:///d:/fz/0508-3/solo-dogfeeding/code/205-panel/app/Services/Subusers/SubuserCreationService.php#L39-L73) | `handle()` 方法 |
| 权限分配安全校验 | [SubuserRequest.php](file:///d:/fz/0508-3/solo-dogfeeding/code/205-panel/app/Http/Requests/Api/Client/Servers/Subusers/SubuserRequest.php#L52-L71) | `validatePermissionsCanBeAssigned()` |
| API 层权限声明 | [ClientApiRequest.php](file:///d:/fz/0508-3/solo-dogfeeding/code/205-panel/app/Http/Requests/Api/Client/ClientApiRequest.php#L17-L32) | `authorize()` 方法 |
| 前端权限判定 Hook | [usePermissions.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/205-panel/resources/scripts/plugins/usePermissions.ts#L4-L21) | `usePermissions()` 函数 |
| 前端权限组件 | [Can.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/205-panel/resources/scripts/components/elements/Can.tsx#L12-L22) | `Can` 组件 |
| 会话吊销 | [RevocationListener.php](file:///d:/fz/0508-3/solo-dogfeeding/code/205-panel/app/Listeners/RevocationListener.php#L15-L32) | `revoke()` 方法 |
| 路由中间件挂载 | [RouteServiceProvider.php](file:///d:/fz/0508-3/solo-dogfeeding/code/205-panel/app/Providers/RouteServiceProvider.php#L38-L66) | `boot()` → `routes()` |
| 管理员 API Key ACL | [AdminAcl.php](file:///d:/fz/0508-3/solo-dogfeeding/code/205-panel/app/Services/Acl/Api/AdminAcl.php#L40-L56) | `can()` 和 `check()` 方法 |
