# 操作日志与事件监听系统梳理

## 一、整体架构概览

操作日志系统覆盖三条独立的写入路径，每条路径经过不同的中间件组，最终统一查询展示：

```
路径1: 客户端 API 请求 (/api/client, /api/application)
  中间件组: api + client-api (或 application-api)
  → TrackAPIKey 设置 apiKeyId → AccountSubject/ServerSubject 设置 actor/subject
  → 业务逻辑 → Activity::log() → 写入 activity_logs + activity_log_subjects
  → 触发 ActivityLogged 事件（当前无消费方）

路径2: Wings 守护进程批量推送 (/api/remote/activity)
  中间件组: daemon (SubstituteBindings + DaemonAuthenticate)
  → 不经 TrackAPIKey，不设 apiKeyId → ActivityProcessingController
  → 批量 insertGetId() 直接插入 activity_logs + activity_log_subjects
  → 不触发 ActivityLogged 事件（insertGetId 不触发 Eloquent 模型事件）

路径3: Wings 远程回调 (/api/remote/sftp/auth, /api/remote/backups/...)
  中间件组: daemon (SubstituteBindings + DaemonAuthenticate)
  → 不经 TrackAPIKey，不设 apiKeyId → 控制器手动设置 actor/subject
  → Activity::log() → 写入 activity_logs + activity_log_subjects
  → 触发 ActivityLogged 事件（当前无消费方）

查询展示 (仅限客户端 API):
  前端 → 客户端 API 控制器 → QueryBuilder 分页 → Transformer 转换 → 页面渲染
```

### 三条路径的关键差异

| 维度 | 路径1: 客户端 API | 路径2: Wings 批量推送 | 路径3: Wings 远程回调 |
|------|-------------------|----------------------|---------------------|
| 中间件组 | `api` + `client-api` | `daemon` | `daemon` |
| TrackAPIKey | ✅ 生效 | ❌ 不生效 | ❌ 不生效 |
| AccountSubject/ServerSubject | ✅ 生效 | ❌ 不生效 | ❌ 不生效 |
| api_key_id | 有值 | null | null |
| actor 来源 | LogTarget 自动设置 | 请求体中的 user UUID | 控制器手动设置 |
| subject 来源 | LogTarget 自动设置 | 固定为 Server | 控制器手动设置 |
| batch UUID | 可能有 | null | 可能有 |
| 触发 ActivityLogged | ✅ (save()) | ❌ (insertGetId()) | ✅ (save()) |

## 二、核心数据模型

### 2.1 ActivityLog - 活动日志主表

**文件**: `app/Models/ActivityLog.php`

**核心字段**:
| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | bigint | 主键 |
| `batch` | uuid, nullable | 批次ID，关联同一请求/事务中的多条日志 |
| `event` | string | 事件类型，如 `auth:success`, `server:backup.start` |
| `ip` | string | 操作者IP地址 |
| `description` | text, nullable | 事件描述 |
| `actor_type` / `actor_id` | nullable morph | 操作者（多态关联，通常为 User；Wings 推送的日志可能无 actor） |
| `api_key_id` | unsigned int, nullable | API 密钥 ID，仅客户端 API 路径有值，通过后续迁移 `2022_06_18_112822` 添加 |
| `properties` | json | 附加属性（ip、useragent、directory、command 等） |
| `timestamp` | timestamp | 事件发生时间 |

**关键特性**:
- 模型创建时自动触发 `ActivityLogged` 事件 (`boot()` 方法, `ActivityLog.php:149-156`)
- 支持 `MassPrunable` 自动清理旧日志 (`prunable()` 方法, `ActivityLog.php:136-143`)
- `DISABLED_EVENTS = ['server:file.upload']` 定义不在 API 响应中返回的事件（仍会记录）
- 预加载 `subjects` 关联 (`$with = ['subjects']`)

### 2.2 ActivityLogSubject - 日志主题关联表

**文件**: `app/Models/ActivityLogSubject.php`

**核心字段**:
| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | bigint | 自增主键（非典型 Pivot） |
| `activity_log_id` | bigint | 外键，关联 activity_logs，级联删除 |
| `subject_type` / `subject_id` | morph | 关联的主题（多态，如 Server、User、Backup 等） |

**设计说明**:
- 继承自 `Pivot`，但 `$incrementing = true`，拥有自增主键
- 一条日志可关联多个主题（多对多），例如备份操作同时关联 Server 和 Backup
- `subject()` 和 `actor()` 关联均支持 `withTrashed()`，确保已软删除的模型仍可查询

### 2.3 模型关联关系

```php
// User 模型 (app/Models/User.php:278)
public function activity(): MorphToMany
{
    return $this->morphToMany(ActivityLog::class, 'subject', 'activity_log_subjects');
}

// Server 模型 (app/Models/Server.php:378)
public function activity(): MorphToMany
{
    return $this->morphToMany(ActivityLog::class, 'subject', 'activity_log_subjects');
}

// ActivityLog 模型
public function actor(): MorphTo       // 多态反向关联到操作者
public function subjects(): HasMany    // 一对多关联到 ActivityLogSubject
public function apiKey(): HasOne       // 关联到 ApiKey
```

## 三、服务层设计

### 3.1 ActivityLogService - 日志写入服务

**文件**: `app/Services/Activity/ActivityLogService.php`

**Facade 入口**: `Pterodactyl\Facades\Activity` → 解析为 `ActivityLogService`

**核心方法链**:
```php
Activity::event('server:backup.start')        // 设置事件类型（必须）
        ->subject($server, $backup)            // 设置关联主题（可多个）
        ->actor($user)                         // 设置操作者（可选，自动从 Auth/LogTarget 获取）
        ->property('directory', '/data/')      // 设置单个附加属性
        ->property(['ip' => $ip, 'key' => $v]) // 或批量设置附加属性
        ->withRequestMetadata()                // 自动附加 ip 和 useragent
        ->log('启动备份任务');                  // 写入日志并返回 ActivityLog 实例
```

**关键实现细节**:

- `getActivity()` (`ActivityLogService.php:196-220`): 延迟创建 ActivityLog 实例，自动填充:
  - IP 地址: `Request::ip()`
  - Batch UUID: 从 `ActivityLogBatchService::uuid()` 获取（可能为 null）
  - API Key ID: 从 `ActivityLogTargetableService::apiKeyId()` 获取（仅客户端 API 路径有值，因为仅该路径经 TrackAPIKey 中间件设置）
  - Subject: 从 `ActivityLogTargetableService::subject()` 获取（仅客户端 API 路径由中间件预设）
  - Actor: 优先使用 `ActivityLogTargetableService::actor()`（中间件预设），其次回退到 `AuthManager->guard()->user()`

- `save()` (`ActivityLogService.php:227-252`): 在数据库事务中:
  1. 保存 `activity_logs` 记录
  2. 批量插入 `activity_log_subjects` 关联
  3. 写入完成后重置 `$this->activity` 和 `$this->subjects`

- `log()` (`ActivityLogService.php:133-153`): 写入日志，异常处理策略:
  - 非生产环境：直接抛出异常
  - 生产环境：仅写入 `Log::error()`，不中断业务流程

- `transaction()` (`ActivityLogService.php:173-182`): 在数据库事务中执行回调，事务成功后自动保存日志

- `clone()` (`ActivityLogService.php:159-162`): 返回服务实例的克隆，用于基于一个模板创建多条日志

### 3.2 ActivityLogBatchService - 批次管理服务

**文件**: `app/Services/Activity/ActivityLogBatchService.php`

**设计目的**: 将同一请求/事务中的多条日志关联到同一个 UUID，便于追踪。

**核心机制**:
- 通过 `ActivityLogServiceProvider` 以 `scoped` 方式注册，每个请求生命周期共享一个实例
- 内部维护 `$transaction` 计数器，支持嵌套调用
- `$transaction > 0` 时持有 UUID，等于 0 时清空
- `start()`: 计数器 +1，首次进入时生成 UUID
- `end()`: 计数器 -1（最小为 0），归零时清空 UUID
- `transaction(Closure)`: 包装 start/end，自动管理批次生命周期

**使用场景**:
```php
// 典型场景：BackupController::store 中
Activity::event('server:backup.start')->transaction(function ($log) use (...) {
    // 回调中的所有 activity log 共享同一个 batch UUID
    // 事务成功才写入日志
    return $action->handle($server, $name);
});
```

**注意**: 当 `ActivityLogService::getActivity()` 被调用时，`$this->batch->uuid()` 可能为 null（不在批次事务中），此时 `activity_logs.batch` 列为 null。

### 3.3 ActivityLogTargetableService - 请求级上下文服务

**文件**: `app/Services/Activity/ActivityLogTargetableService.php`

**Facade 入口**: `Pterodactyl\Facades\LogTarget` → 解析为 `ActivityLogTargetableService`

**设计目的**: 在请求生命周期中存储日志上下文（actor、subject、apiKeyId），避免在每个控制器中重复传递。

**可存储信息**:
- `actor`: 操作者模型（由客户端 API 路径的中间件通过 `LogTarget::setActor()` 预设）
- `subject`: 关联主题模型（由客户端 API 路径的中间件通过 `LogTarget::setSubject()` 预设）
- `apiKeyId`: API 密钥 ID（由客户端 API 路径的 TrackAPIKey 中间件通过 `LogTarget::setApiKeyId()` 预设）

**注意**: remote 路径使用 `daemon` 中间件组，不经过 TrackAPIKey/AccountSubject/ServerSubject，因此 `ActivityLogTargetableService` 中的 actor/subject/apiKeyId 均不会被中间件预设，需要控制器手动设置或回退到其他默认值。

**服务注册** (`app/Providers/ActivityLogServiceProvider.php:15-19`):
```php
$this->app->scoped(ActivityLogBatchService::class);
$this->app->scoped(ActivityLogTargetableService::class);
```

## 四、中间件与路由分组

### 4.0 路由与中间件组的绑定关系

**文件**: `app/Providers/RouteServiceProvider.php:38-66`

Laravel 的中间件按组分配，不同的路由前缀使用不同的中间件组，这决定了日志上下文的自动注入能力：

```php
// RouteServiceProvider::routes() 的关键结构

// 1. /api/client 和 /api/application → api 中间件组
Route::middleware(['api', RequireTwoFactorAuthentication::class])->group(function () {
    Route::middleware(['application-api', 'throttle:api.application'])
        ->prefix('/api/application')
        ->group(base_path('routes/api-application.php'));

    Route::middleware(['client-api', 'throttle:api.client'])
        ->prefix('/api/client')
        ->group(base_path('routes/api-client.php'));
});

// 2. /api/remote → daemon 中间件组（不是 api 中间件组）
Route::middleware('daemon')
    ->prefix('/api/remote')
    ->scopeBindings()
    ->group(base_path('routes/api-remote.php'));
```

**对应 Kernel.php 中的中间件组定义** (`app/Http/Kernel.php:60-90`):

```php
protected $middlewareGroups = [
    'api' => [                         // ← /api/client 和 /api/application 使用此组
        EnsureStatefulRequests::class,
        'auth:sanctum',
        IsValidJson::class,
        TrackAPIKey::class,            // ← 仅在此组中，作用于客户端 API
        RequireTwoFactorAuthentication::class,
        AuthenticateIPAccess::class,
    ],
    'client-api' => [                  // ← /api/client 额外使用此组
        SubstituteClientBindings::class,
        RequireClientApiKey::class,
    ],
    'application-api' => [             // ← /api/application 额外使用此组
        SubstituteBindings::class,
        AuthenticateApplicationUser::class,
    ],
    'daemon' => [                      // ← /api/remote 使用此组（不含 TrackAPIKey）
        SubstituteBindings::class,
        DaemonAuthenticate::class,
    ],
];
```

**核心结论**: `/api/remote` 路由使用 `daemon` 中间件组，不包含 `TrackAPIKey`，也不包含 `auth:sanctum`。因此 remote 路径中：
- `TrackAPIKey` 不会执行 → `api_key_id` 始终为 null
- `$request->user()` 返回的不是 User 模型 → AuthManager 回退取不到用户
- `AccountSubject`/`ServerSubject` 不在中间件栈中 → actor/subject 不会被自动设置

### 4.1 TrackAPIKey - API 密钥追踪

**文件**: `app/Http/Middleware/Activity/TrackAPIKey.php`

**注入位置**: `app/Http/Kernel.php:74`，在 `api` 中间件组中注册。

**作用域**: 仅作用于使用 `api` 中间件组的路由，即 `/api/client` 和 `/api/application`。**不作用于** `/api/remote`（该路径使用 `daemon` 中间件组）。

**实现逻辑** (`TrackAPIKey.php:17-26`):
- 检查 `$request->user()` 是否存在
- 获取 `$request->user()->currentAccessToken()`
- 若 token 是 `ApiKey` 实例，记录其 `id` 到 `LogTarget::setApiKeyId()`
- 若为 `TransientToken`（会话认证），则设置 `api_key_id` 为 null

### 4.2 AccountSubject - 账户上下文

**文件**: `app/Http/Middleware/Activity/AccountSubject.php`

**注入位置**: `routes/api-client.php:23`，仅作用于 `/api/client/account` 路由组：

```php
Route::prefix('/account')->middleware(AccountSubject::class)->group(function () {
    // 账户相关路由...
});
```

**作用**: 将当前登录用户同时设为 `LogTarget::setActor()` 和 `LogTarget::setSubject()`，适用于账户自身操作（改密码、2FA、API Key 管理等）。

### 4.3 ServerSubject - 服务器上下文

**文件**: `app/Http/Middleware/Activity/ServerSubject.php`

**注入位置**: `routes/api-client.php:57-64`，作用于 `/api/client/servers/{server}` 路由组：

```php
Route::group([
    'prefix' => '/servers/{server}',
    'middleware' => [
        ServerSubject::class,
        AuthenticateServerAccess::class,
        ResourceBelongsToServer::class,
    ],
], function () {
    // 服务器相关路由...
});
```

**作用**: 当路由参数包含 `server`（类型为 `Server` 模型）时:
- `LogTarget::setActor($request->user())` — 设置操作者
- `LogTarget::setSubject($server)` — 设置关联主题为服务器

**注意**: `ServerSubject` 不会覆盖 `TrackAPIKey` 已设置的 `apiKeyId`，二者协作：TrackAPIKey 在 `api` 组中设置 apiKeyId，ServerSubject 在路由级设置 actor 和 subject。

### 4.4 中间件协作时序

#### 客户端 API 请求

```
/api/client/servers/{server}/...
  │
  ├─ 1. api 中间件组 (Kernel.php) ← TrackAPIKey 在此层
  │     ├─ EnsureStatefulRequests
  │     ├─ auth:sanctum             → 认证 User，设置 $request->user()
  │     ├─ IsValidJson
  │     ├─ TrackAPIKey              → LogTarget::setApiKeyId(...)
  │     ├─ RequireTwoFactorAuthentication
  │     └─ AuthenticateIPAccess     → Activity::event('auth:ip-blocked') (可选)
  │
  ├─ 2. client-api 中间件组 (Kernel.php)
  │     ├─ SubstituteClientBindings
  │     └─ RequireClientApiKey
  │
  ├─ 3. 路由级中间件 (api-client.php) ← AccountSubject/ServerSubject 在此层
  │     ├─ ServerSubject            → LogTarget::setActor(user) + LogTarget::setSubject(server)
  │     ├─ AuthenticateServerAccess
  │     └─ ResourceBelongsToServer
  │
  └─ 4. 控制器
       └─ Activity::event('server:xxx')->log()
            → 从 LogTarget 读取预设的 actor / subject / apiKeyId
```

#### Remote API 请求 (Wings 守护进程)

```
/api/remote/...
  │
  ├─ 1. daemon 中间件组 (Kernel.php) ← 注意：不含 TrackAPIKey
  │     ├─ SubstituteBindings
  │     └─ DaemonAuthenticate        → 认证 Node，设置 $request->attributes->get('node')
  │                                    （不设置 $request->user() 为 User 模型）
  │
  ├─ 2. 控制器
  │     ├─ LogTarget::apiKeyId()     → null (未经 TrackAPIKey)
  │     ├─ LogTarget::actor()        → null (未经 AccountSubject/ServerSubject)
  │     ├─ LogTarget::subject()      → null (未经 AccountSubject/ServerSubject)
  │     │
  │     ├─ 方式A: 手动设置 actor/subject
  │     │   Activity::event('...')->actor($user)->subject($server)->log()
  │     │   (SftpAuthenticationController, BackupStatusController 等)
  │     │
  │     └─ 方式B: 直接 insertGetId，完全绕过 Activity Facade
  │         (ActivityProcessingController)
  └─
```

## 五、事件监听机制

### 5.1 ActivityLogged 事件 — 当前无消费链路

**文件**: `app/Events/ActivityLogged.php`

**触发时机**: ActivityLog 模型的 `created` 事件 (`ActivityLog.php:149-156`):
```php
protected static function boot()
{
    parent::boot();

    static::created(function (self $model) {
        Event::dispatch(new ActivityLogged($model));
    });
}
```

**重要**: 仅通过 Eloquent `save()` 方法创建的记录会触发此事件。`ActivityProcessingController` 使用 `insertGetId()` 直接插入，绕过了 Eloquent 模型事件，因此 Wings 批量推送的日志不触发此事件。

**事件辅助方法**:
- `is(string $event)`: 判断事件类型是否匹配
- `isServerEvent()`: 判断是否为 `server:` 前缀的事件
- `isSystem()`: 判断是否为系统操作（`actor_id` 为 null）
- `actor()`: 返回操作者模型，系统操作返回 null

**当前状态**: 经核实 `app/Providers/EventServiceProvider.php`，`ActivityLogged` 事件 **没有注册任何 Listener**。`EventServiceProvider::$listen` 和 `$subscribe` 中均无此事件的订阅。

```php
// app/Providers/EventServiceProvider.php — 完整的事件注册
protected $listen = [
    ServerInstalledEvent::class => [ServerInstalledNotification::class],
];

protected $subscribe = [
    AuthenticationListener::class,
    RevocationListener::class,
    TwoFactorListener::class,
];
```

全代码库搜索 `ActivityLogged` 的使用，仅在以下位置出现:
- `ActivityLog.php:154` — 触发点 (`Event::dispatch`)
- `tests/` 目录 — 测试中通过 `Event::fake(ActivityLogged::class)` 进行断言

**结论**: `ActivityLogged` 事件当前是"发射后不管"的状态，没有实际的消费链路。这是一个扩展点设计，为将来添加事件驱动的后续处理（如通知、WebSocket 推送等）预留了接口。

### 5.2 认证相关事件监听（写入日志，非消费 ActivityLogged）

以下 Listener 订阅的是 Laravel 内置认证事件和自定义认证事件，它们是**日志的写入方**，而非 `ActivityLogged` 的消费方：

#### AuthenticationListener

**文件**: `app/Listeners/AuthenticationListener.php`

**注册方式**: `EventServiceProvider::$subscribe` (订阅者模式)

**订阅事件**:
| 事件 | 处理方法 | 写入的 activity event |
|------|----------|----------------------|
| `Illuminate\Auth\Events\Failed` | `login()` | `auth:fail` |
| `Pterodactyl\Events\Auth\DirectLogin` | `login()` | `auth:success` |
| `Illuminate\Auth\Events\PasswordReset` | `reset()` | `event:password-reset` |

**处理逻辑** (`login()` 方法):
```php
public function login(Failed|DirectLogin $event): void
{
    $activity = Activity::withRequestMetadata();
    if ($event->user) {
        $activity = $activity->subject($event->user);
    }
    if ($event instanceof Failed) {
        foreach ($event->credentials as $key => $value) {
            $activity = $activity->property($key, $value);
        }
    }
    $activity->event($event instanceof Failed ? 'auth:fail' : 'auth:success')->log();
}
```

#### TwoFactorListener

**文件**: `app/Listeners/TwoFactorListener.php`

**注册方式**: `EventServiceProvider::$subscribe`

**订阅事件**:
| 事件 | 处理方法 | 写入的 activity event |
|------|----------|----------------------|
| `ProvidedAuthenticationToken` | `__invoke()` | `auth:recovery-token` 或 `auth:token` |

**处理逻辑**:
```php
public function __invoke(ProvidedAuthenticationToken $event): void
{
    Activity::event($event->recovery ? 'auth:recovery-token' : 'auth:token')
        ->withRequestMetadata()
        ->subject($event->user)
        ->log();
}
```

### 5.3 非监听器的日志写入点

以下位置直接在业务代码中调用 `Activity::` 写入日志，不经过事件监听机制：

**Web 认证流程** (走 `web` 中间件组，不经过 TrackAPIKey):
| 文件 | activity event | 说明 |
|------|---------------|------|
| `LoginController.php:60` | `auth:checkpoint` | 2FA 验证检查点 |

**客户端 API 安全拦截** (走 `api` 中间件组，经过 TrackAPIKey):
| 文件 | activity event | 说明 |
|------|---------------|------|
| `AuthenticateIPAccess.php:40` | `auth:ip-blocked` | API Key 的 IP 白名单拦截 |

**Remote API (Wings)** (走 `daemon` 中间件组，不经过 TrackAPIKey):
| 文件 | activity event | 说明 |
|------|---------------|------|
| `SftpAuthenticationController.php:52` | `auth:sftp.fail` | SFTP 密码认证失败 |
| `SftpAuthenticationController.php:147` | `server:sftp.denied` | SFTP 权限不足 |
| `BackupStatusController.php:56` | `server:backup.complete` / `server:backup.fail` | 备份完成/失败 |
| `BackupStatusController.php:105` | `server:backup.restore-complete` / `server.backup.restore-failed` | 恢复完成/失败 |
| `ServerDetailsController.php:118` | `server:backup.restore-failed` | 节点重置时恢复失败 |
| `ActivityProcessingController.php` | 动态 `server:*` 事件 | Wings 推送的服务器活动日志 |

## 六、日志写入完整路径

### 6.1 路径一：客户端 API — 通过 Activity Facade

**覆盖范围**: 所有 `/api/client/` 和 `/api/application/` 路由下的控制器

**中间件栈**: `api` 组（含 TrackAPIKey）→ `client-api` 或 `application-api` 组 → 路由级中间件（AccountSubject / ServerSubject）

**自动注入的上下文**:
- `api_key_id`: 由 TrackAPIKey 设置（API Key 认证时有值，会话认证时为 null）
- `actor`: 由 AccountSubject/ServerSubject 设置，或回退到 AuthManager 的当前用户
- `subject`: 由 AccountSubject/ServerSubject 设置

**完整事件清单**:

**账户类事件** (`/api/client/account`, 中间件: AccountSubject):
| 事件名 | 触发位置 |
|--------|----------|
| `user:account.email-changed` | `AccountController.php:58` |
| `user:account.password-changed` | `AccountController.php:74` |
| `user:two-factor.create` | `TwoFactorController.php:67` |
| `user:two-factor.delete` | `TwoFactorController.php:97` |
| `user:api-key.create` | `ApiKeyController.php:41` |
| `user:api-key.delete` | `ApiKeyController.php:63` |
| `user:ssh-key.create` | `SSHKeyController.php:35` |
| `user:ssh-key.delete` | `SSHKeyController.php:59` |
| `auth:reset-password` | `User.php:213` |

**服务器类事件** (`/api/client/servers/{server}`, 中间件: ServerSubject):
| 事件名 | 触发位置 |
|--------|----------|
| `server:power.start/stop/restart/kill` | `PowerController.php:31` |
| `server:console.command` | `CommandController.php:46` |
| `server:backup.start` | `BackupController.php:80` |
| `server:backup.lock` / `server:backup.unlock` | `BackupController.php:114` |
| `server:backup.delete` | `BackupController.php:151` |
| `server:backup.download` | `BackupController.php:179` |
| `server:backup.restore` | `BackupController.php:210` |
| `server:database.create` | `DatabaseController.php:52` |
| `server:database.rotate-password` | `DatabaseController.php:78` |
| `server:database.delete` | `DatabaseController.php:98` |
| `server:file.read/download/write/create-directory/rename/copy/compress/decompress/delete/pull` | `FileController.php` |
| `server:schedule.create/update/execute/delete` | `ScheduleController.php` |
| `server:task.create/update/delete` | `ScheduleTaskController.php` |
| `server:allocation.notes/primary/create/delete` | `NetworkAllocationController.php` |
| `server:subuser.create/update/delete` | `SubuserController.php` |
| `server:settings.rename` | `SettingsController.php:45` |
| `server:settings.description` | `SettingsController.php:51` |
| `server:reinstall` | `SettingsController.php:68` |
| `server:startup.image` | `SettingsController.php:88` |
| `server:startup.edit` | `StartupController.php:81` |

### 6.2 路径二：Wings 守护进程推送 — 批量直接插入

**文件**: `app/Http/Controllers/Api/Remote/ActivityProcessingController.php`

**路由**: `POST /api/remote/activity` (`routes/api-remote.php:11`)

**中间件栈**: `daemon` 组（SubstituteBindings + DaemonAuthenticate）— 不含 TrackAPIKey

**请求格式** (通过 `ActivityEventRequest` 验证):
```json
{
  "data": [
    {
      "server": "uuid",
      "user": "uuid (nullable)",
      "event": "server:console.command",
      "metadata": { "command": "ls" },
      "ip": "127.0.0.1",
      "timestamp": "2022-05-28T12:00:00Z"
    }
  ]
}
```

**处理流程**:
1. 从请求中提取所有唯一的 server UUID 和 user UUID
2. 批量查询 Server 和 User 模型（减少数据库查询）
3. 遍历每条数据:
   - 验证 server 存在且 event 以 `server:` 开头
   - 解析 RFC3339 时间戳，失败则使用当前时间并记录原始时间戳到 metadata
   - 构建 `activity_logs` 记录（ip、event、properties、timestamp、actor_id/actor_type）
4. 按服务器分组，批量插入 `activity_logs`（使用 `insertGetId`）
5. 批量插入 `activity_log_subjects` 关联（subject 均为对应的 Server）

**此路径的两个关键特征**:

1. **不经 Activity Facade**: 使用 `ActivityLog::insertGetId()` 直接操作数据库，绕过了 `ActivityLogService` 的全部逻辑
2. **经 `daemon` 中间件组**: 不含 TrackAPIKey，即使经过 Activity Facade 也无法获得 api_key_id

**由此产生的差异**:
- `batch` 列为 null（未使用 `ActivityLogBatchService`）
- `api_key_id` 列为 null（双重原因：不经 TrackAPIKey 且不经 ActivityLogService）
- 不触发 `ActivityLogged` 事件（`insertGetId` 不触发 Eloquent 模型事件）
- actor 信息从请求体中的 user UUID 解析，而非从 `$request->user()` 获取

### 6.3 路径三：Wings 远程回调 — 通过 Activity Facade

**中间件栈**: `daemon` 组（SubstituteBindings + DaemonAuthenticate）— 不含 TrackAPIKey

| 控制器 | 路由 | 事件 | 说明 |
|--------|------|------|------|
| `BackupStatusController::index` | `POST /api/remote/backups/{backup}` | `server:backup.complete` / `server:backup.fail` | 备份状态上报 |
| `BackupStatusController::restore` | `POST /api/remote/backups/{backup}/restore` | `server:backup.restore-complete` / `server.backup.restore-failed` | 恢复状态上报 |
| `SftpAuthenticationController` | `POST /api/remote/sftp/auth` | `auth:sftp.fail` / `server:sftp.denied` | SFTP 认证 |
| `ServerDetailsController::resetState` | `POST /api/remote/servers/reset` | `server:backup.restore-failed` | 节点重置时标记恢复失败 |

**此路径的关键特征**:

1. **经 Activity Facade**: 使用 `Activity::event()...->log()`，走 `ActivityLogService::save()`
2. **经 `daemon` 中间件组**: 不含 TrackAPIKey，`ActivityLogTargetableService` 中的 actor/subject/apiKeyId 均未被中间件预设

**由此产生的差异**:
- `api_key_id` 列为 null（不经 TrackAPIKey，`ActivityLogTargetableService::apiKeyId()` 返回 null）
- actor 由控制器手动设置（如 `SftpAuthenticationController` 中 `Activity::event('...')->actor($user)->subject($server)->log()`），或回退到 `AuthManager->guard()->user()`（在 daemon 认证下通常为 null）
- subject 由控制器手动设置
- 触发 `ActivityLogged` 事件（使用 Eloquent `save()`）

## 七、查询与展示

### 7.1 账户活动日志 API

**文件**: `app/Http/Controllers/Api/Client/ActivityLogController.php`

**路由**: `GET /api/client/account/activity` (`routes/api-client.php:36`)

**中间件**: `api` 组（含 TrackAPIKey）→ `client-api` 组 → AccountSubject

**查询逻辑**:
```php
QueryBuilder::for($request->user()->activity())  // 查询以当前用户为 subject 的日志
    ->with('actor')
    ->allowedFilters([AllowedFilter::partial('event')])
    ->allowedSorts(['timestamp'])
    ->whereNotIn('activity_logs.event', ActivityLog::DISABLED_EVENTS)  // 排除禁用事件
    ->paginate(min($request->query('per_page', 25), 100))
    ->appends($request->query());
```

**注意**: `$request->user()->activity()` 使用的是 `MorphToMany` 关联，通过 `activity_log_subjects` 表查找以当前用户为 subject 的日志，而非以用户为 actor 的日志。

### 7.2 服务器活动日志 API

**文件**: `app/Http/Controllers/Api/Client/Servers/ActivityLogController.php`

**路由**: `GET /api/client/servers/{server}/activity` (`routes/api-client.php:70`)

**中间件**: `api` 组（含 TrackAPIKey）→ `client-api` 组 → ServerSubject + AuthenticateServerAccess + ResourceBelongsToServer

**安全特性**:
- 权限检查: `$this->authorize(Permission::ACTION_ACTIVITY_READ, $server)`
- 可配置隐藏管理员活动: 当 `config('activity.hide_admin_activity')` 为 true 时:
  - 查出服务器的 subuser ID 列表 + owner_id
  - 左连接 `users` 表
  - 排除: 非用户操作 + root_admin 且非服务器成员的操作
- 排除 `DISABLED_EVENTS` 中定义的事件类型

### 7.3 数据转换层

**文件**: `app/Transformers/Api/Client/ActivityLogTransformer.php`

**输出字段**:
| 字段 | 转换规则 |
|------|----------|
| `id` | `sha1($model->id)` — 前端唯一渲染 key，非安全用途 |
| `batch` | 原值 |
| `event` | 原值 |
| `is_api` | `!is_null($model->api_key_id)` — 仅客户端 API 路径产生的日志此字段为 true |
| `ip` | 仅本人或 root_admin 可见，否则 null |
| `description` | 原值 |
| `properties` | 经规范化处理（见下方） |
| `has_additional_metadata` | 智能判断是否有未在描述中展示的额外属性 |
| `timestamp` | Atom 格式字符串 |

**properties 规范化规则**:
- 非本人的 `ip` 属性显示为 `[hidden]`
- 数组类型属性自动生成 `{key}_count` 字段
- 若只有一个 `_count` 字段，重命名为 `count` 并移除原字段
- `directory` 属性自动规范化为 `/path/` 格式

**has_additional_metadata 判断逻辑**:
1. 获取 `trans('activity.' . str_replace(':', '.', $event))` 对应的语言字符串
2. 解析字符串中的 `:key` 占位符
3. 将已展示的 key 加上 `ip`、`useragent`、`using_sftp` 组成排除列表
4. 若 properties 中存在不在排除列表中的 key，则返回 true

**actor 关联** (available include):
- 仅当 actor 为 `User` 实例时输出，否则返回 null

### 7.4 前端展示

**API Hooks**:
| 文件 | 请求路径 | 说明 |
|------|----------|------|
| `resources/scripts/api/account/activity.ts` | `GET /api/client/account/activity` | 账户活动日志 |
| `resources/scripts/api/server/activity.ts` | `GET /api/client/servers/{uuid}/activity` | 服务器活动日志 |

**组件**:
- `resources/scripts/components/dashboard/activity/ActivityLogContainer.tsx` — 账户日志列表容器
- `resources/scripts/components/server/ServerActivityLogContainer.tsx` — 服务器日志列表容器
- `resources/scripts/components/elements/activity/ActivityLogEntry.tsx` — 单条日志展示

**前端技术栈**:
- `useSWR` 数据获取与缓存，key 包含用户/服务器 ID + 筛选参数
- 支持 `event`、`ip` 筛选（通过 `withQueryBuilderParams` 传递）
- 支持 `timestamp` 排序
- URL Hash 同步筛选条件（`useLocationHash`）
- `revalidateOnMount: false`（账户日志）/ `revalidateOnMount: true`（服务器日志首次进入时加载）

## 八、配置与清理

### 8.1 配置文件

**文件**: `config/activity.php`
```php
return [
    'prune_days' => env('APP_ACTIVITY_PRUNE_DAYS', 90),
    'hide_admin_activity' => env('APP_ACTIVITY_HIDE_ADMIN', false),
];
```

### 8.2 自动清理

**实现**: `ActivityLog::prunable()` (`ActivityLog.php:136-143`)
- Laravel 内置 `MassPrunable` trait
- 配合 `php artisan model:prune` 命令使用
- 清理 `timestamp` 超过 `prune_days` 天的记录
- 若 `prune_days` 配置为 null，抛出 `LogicException`

### 8.3 禁用事件

**常量**: `ActivityLog::DISABLED_EVENTS = ['server:file.upload']`
- 这些事件仍会被写入数据库，但查询 API 通过 `whereNotIn` 排除
- 不影响 Wings 推送或远程回调中的写入

### 8.4 api_key_id 追踪

**迁移**: `database/migrations/2022_06_18_112822_track_api_key_usage_for_activity_events.php`
- 在 `activity_logs` 表添加 `api_key_id` unsignedInteger nullable 列
- 仅客户端 API 路径（`/api/client`, `/api/application`）通过 `TrackAPIKey` 中间件自动填充
- Remote 路径（`/api/remote`）使用 `daemon` 中间件组，不含 TrackAPIKey，因此 `api_key_id` 始终为 null
- 在 Transformer 中通过 `is_api` 字段暴露给前端

## 九、关键设计模式

### 9.1 Scoped 容器绑定

```php
// ActivityLogServiceProvider.php
$this->app->scoped(ActivityLogBatchService::class);
$this->app->scoped(ActivityLogTargetableService::class);
```
- 每个请求生命周期内共享同一实例（等效于 singleton 但作用域限定在单次请求）
- 确保同一请求中的日志共享批次信息和上下文
- 在 remote 路径中，虽然实例存在，但因为不经 TrackAPIKey/AccountSubject/ServerSubject 中间件，actor/subject/apiKeyId 均为 null

### 9.2 链式调用 (Fluent Interface)

```php
Activity::event('...')->subject(...)->property(...)->log();
```
- 每个方法返回 `$this`，支持流畅调用
- `log()` 和 `transaction()` 为终端操作

### 9.3 延迟加载

`getActivity()` 方法仅在首次调用时创建 ActivityLog 实例:
- 未调用 `log()` 前不产生任何数据库操作
- 自动从 LogTarget 和 AuthManager 填充上下文
- 在 remote 路径中，LogTarget 的值全为 null，AuthManager 的 guard user 也为 null（daemon 认证不设置 User），因此 actor 和 subject 需要控制器手动指定

### 9.4 多态关联

- `actor` 使用 `nullableMorphs`，支持 User 等多种操作者类型
- `subject` 使用 `morphToMany`，支持 Server、User、Backup 等多种主题
- 所有 morphTo 关联均使用 `withTrashed()`，确保已删除记录可追溯

### 9.5 三条写入路径的权衡

- **Facade 路径（路径1和3）**: 通过 `ActivityLogService` → `save()` → 触发 `ActivityLogged` 事件。路径1有完整的中间件上下文，路径3需要手动设置 actor/subject。
- **直接插入路径（路径2）**: `ActivityProcessingController` → `insertGetId` → 不触发模型事件，不设置 batch/api_key_id。牺牲了模型事件触发和数据完整性，换取了批量写入性能。

## 十、数据流总图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           日志写入路径                                    │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  路径1: 客户端 API (/api/client, /api/application)                      │
│  ┌──────────┐   ┌───────────────────────┐   ┌────────────────────┐     │
│  │ 请求进入  │→ │ api 中间件组            │→ │ 路由级中间件         │     │
│  │          │   │ TrackAPIKey            │   │ AccountSubject     │     │
│  │          │   │ → LogTarget::apiKeyId  │   │ ServerSubject      │     │
│  │          │   │ AuthenticateIPAccess   │   │ → LogTarget::actor │     │
│  │          │   │   (auth:ip-blocked)    │   │ → LogTarget::subject│    │
│  └──────────┘   └───────────────────────┘   └────────┬───────────┘     │
│                                                     │                   │
│                                            ┌────────▼───────────┐     │
│                                            │ 控制器             │     │
│                                            │ Activity::event()  │     │
│                                            │   ->log()          │     │
│                                            └────────┬───────────┘     │
│                                                     │                 │
│                                                     │ api_key_id: ✓   │
│                                                     │ actor: ✓        │
│                                                     │ subject: ✓      │
│                                                     │ batch: 可能有    │
│                                                     ▼                 │
│                                            ┌────────────────┐         │
│                                            │ ActivityLog    │         │
│                                            │ Service::save()│         │
│                                            └────────┬───────┘         │
│                                                     │                 │
│                                                     ▼                 │
│                                            ┌────────────────┐         │
│                                            │ ActivityLogged │         │
│                                            │ 事件触发 ✓     │         │
│                                            └────────────────┘         │
│                                                                         │
│  路径2: Wings 批量推送 (/api/remote/activity)                            │
│  ┌──────────┐   ┌───────────────────┐                                   │
│  │ Wings    │→ │ daemon 中间件组     │                                   │
│  │ 守护进程 │   │ SubstituteBindings │                                   │
│  │          │   │ DaemonAuthenticate │                                   │
│  │          │   │ (无 TrackAPIKey)   │                                   │
│  └──────────┘   └────────┬──────────┘                                   │
│                          │                                               │
│                 ┌────────▼───────────────┐                               │
│                 │ ActivityProcessing     │                               │
│                 │ Controller             │                               │
│                 │ insertGetId() 直接插入  │                               │
│                 └────────┬───────────────┘                               │
│                          │                                               │
│                          │ api_key_id: ✗ (null)                          │
│                          │ actor: 从请求体 user UUID 解析                 │
│                          │ subject: 固定为 Server                        │
│                          │ batch: ✗ (null)                               │
│                          │ ActivityLogged 事件: ✗ (不触发)               │
│                          ▼                                               │
│                 ┌────────────────────┐                                   │
│                 │ 直接写入数据库       │                                   │
│                 │ (不经 Eloquent save)│                                   │
│                 └────────────────────┘                                   │
│                                                                         │
│  路径3: Wings 远程回调 (/api/remote/sftp/auth, /api/remote/backups/...)  │
│  ┌──────────┐   ┌───────────────────┐                                   │
│  │ Wings    │→ │ daemon 中间件组     │                                   │
│  │ 守护进程 │   │ SubstituteBindings │                                   │
│  │          │   │ DaemonAuthenticate │                                   │
│  │          │   │ (无 TrackAPIKey)   │                                   │
│  └──────────┘   └────────┬──────────┘                                   │
│                          │                                               │
│                 ┌────────▼───────────────┐                               │
│                 │ 控制器                  │                               │
│                 │ Activity::event()      │                               │
│                 │   ->actor($user)       │  ← 手动设置                    │
│                 │   ->subject($server)   │  ← 手动设置                    │
│                 │   ->log()              │                               │
│                 └────────┬───────────────┘                               │
│                          │                                               │
│                          │ api_key_id: ✗ (null, 不经 TrackAPIKey)        │
│                          │ actor: 手动设置                                │
│                          │ subject: 手动设置                              │
│                          │ batch: 可能有                                  │
│                          ▼                                               │
│                 ┌────────────────┐                                       │
│                 │ ActivityLog    │                                       │
│                 │ Service::save()│                                       │
│                 └────────┬───────┘                                       │
│                          │                                               │
│                          ▼                                               │
│                 ┌────────────────┐                                       │
│                 │ ActivityLogged │                                       │
│                 │ 事件触发 ✓     │                                       │
│                 └────────────────┘                                       │
│                                                                         │
├─────────────────────────────────────────────────────────────────────────┤
│                           查询展示路径                                    │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌──────────────────┐   ┌──────────────────┐   ┌───────────────┐       │
│  │ 前端 SWR Hook    │→ │ API Controller    │→ │ QueryBuilder  │       │
│  │ useActivityLogs  │   │ 分页 + 过滤       │   │ 排除禁用事件   │       │
│  └──────────────────┘   └────────┬─────────┘   │ 隐藏管理员     │       │
│                                  │              └───────┬───────┘       │
│                          ┌───────▼───────┐              │               │
│                          │ Transformer   │←─────────────┘               │
│                          │ IP 可见性控制  │                              │
│                          │ is_api 判断    │                              │
│                          │ 属性规范化     │                              │
│                          └───────┬───────┘                              │
│                          ┌───────▼───────┐                              │
│                          │ React 组件     │                              │
│                          │ ActivityLogEntry│                             │
│                          │ 分页 + 筛选    │                              │
│                          └───────────────┘                              │
└─────────────────────────────────────────────────────────────────────────┘
```
