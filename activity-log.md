# 操作日志与事件监听系统梳理

## 一、整体架构概览

操作日志系统采用分层设计，核心流程如下：

```
用户请求 → 中间件(设置上下文) → 业务逻辑 → Activity::log() 写入日志
                                                          ↓
                                              ActivityLog 模型 created 事件
                                                          ↓
                                              触发 ActivityLogged 事件
                                                          ↓
                                        前端 API 查询 → Transformer 转换 → 页面展示
```

## 二、核心数据模型

### 2.1 ActivityLog - 活动日志主表

**文件**: `app/Models/ActivityLog.php`

**核心字段**:
| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | bigint | 主键 |
| `batch` | uuid | 批次ID，用于关联同一请求/事务中的多条日志 |
| `event` | string | 事件类型，如 `auth:success`, `server:backup.start` |
| `ip` | string | 操作者IP地址 |
| `description` | text | 事件描述（可选） |
| `actor_type` / `actor_id` | morph | 操作者（多态关联，支持用户、系统等） |
| `api_key_id` | int | API密钥ID（如果是通过API调用） |
| `properties` | json | 附加属性，如useragent、目录路径等 |
| `timestamp` | timestamp | 事件发生时间 |

**关键特性**:
- 模型创建时自动触发 `ActivityLogged` 事件 (`boot()` 方法, 第149-156行)
- 支持 `MassPrunable` 自动清理旧日志 (`prunable()` 方法, 第136-143行)
- `DISABLED_EVENTS` 常量定义了不展示的事件类型
- 预加载 `subjects` 关联关系 (`$with = ['subjects']`)

### 2.2 ActivityLogSubject - 日志主题关联表

**文件**: `app/Models/ActivityLogSubject.php`

**核心字段**:
| 字段 | 类型 | 说明 |
|------|------|------|
| `activity_log_id` | bigint | 外键，关联 activity_logs |
| `subject_type` / `subject_id` | morph | 关联的主题（多态，如Server、User等） |

**设计说明**:
- 继承自 `Pivot`，作为中间表连接日志和主题
- 一条日志可以关联多个主题（多对多）
- 支持软删除模型查询 (`withTrashed()`)

### 2.3 模型关联关系

```php
// User 或 Server 模型中定义 (app/Models/User.php:278, Server.php:378)
public function activity(): MorphToMany
{
    return $this->morphToMany(ActivityLog::class, 'subject', 'activity_log_subjects');
}
```

## 三、服务层设计

### 3.1 ActivityLogService - 日志写入服务

**文件**: `app/Services/Activity/ActivityLogService.php`

**核心方法链**:
```php
Activity::event('server:backup.start')        // 设置事件类型
        ->subject($server, $backup)            // 设置关联主题
        ->actor($user)                         // 设置操作者（可选，自动从Auth获取）
        ->property('directory', '/data/')      // 设置附加属性
        ->withRequestMetadata()                // 自动附加IP和User-Agent
        ->log('启动备份任务');                  // 写入日志
```

**关键实现**:
- `getActivity()` (第196-220行): 延迟创建 ActivityLog 实例，自动填充:
  - IP 地址 (`Request::ip()`)
  - Batch UUID (从 `ActivityLogBatchService` 获取)
  - API Key ID (从 `ActivityLogTargetableService` 获取)
  - 主题和操作者 (从 `ActivityLogTargetableService` 或 Auth 获取)

- `save()` (第227-252行): 事务写入日志和主题关联
- `transaction()` (第173-182行): 支持数据库事务，仅当事务成功才写入日志
- `log()` (第133-153行): 非生产环境抛出异常，生产环境仅记录错误日志

### 3.2 ActivityLogBatchService - 批次管理服务

**文件**: `app/Services/Activity/ActivityLogBatchService.php`

**设计目的**: 将同一请求/事务中的多条日志关联在一起，便于追踪。

**核心机制**:
- 使用 `scoped` 绑定，每个请求一个实例
- 支持嵌套事务，引用计数管理 (`$transaction`)
- 当 `$transaction > 0` 时生成并保持 UUID
- `transaction()` 方法包装回调，自动管理批次生命周期

**使用示例**:
```php
app(ActivityLogBatchService::class)->transaction(function ($batchUuid) {
    // 这里产生的所有 activity log 都会带有相同的 batch uuid
    Activity::event('event1')->log();
    Activity::event('event2')->log();
});
```

### 3.3 ActivityLogTargetableService - 上下文目标服务

**文件**: `app/Services/Activity/ActivityLogTargetableService.php`

**设计目的**: 在请求生命周期中存储日志上下文信息，避免重复传递。

**可存储信息**:
- `actor`: 操作者模型
- `subject`: 关联主题模型
- `apiKeyId`: API密钥ID

**服务注册** (`app/Providers/ActivityLogServiceProvider.php:15-19`):
```php
$this->app->scoped(ActivityLogBatchService::class);
$this->app->scoped(ActivityLogTargetableService::class);
```

**Facade 访问**:
- `Activity` → `ActivityLogService` (写入日志)
- `LogTarget` → `ActivityLogTargetableService` (设置上下文)

## 四、中间件 - 自动上下文注入

### 4.1 AccountSubject - 账户上下文

**文件**: `app/Http/Middleware/Activity/AccountSubject.php`

**作用**: 将当前登录用户同时设为 actor 和 subject，适用于账户相关操作。

### 4.2 ServerSubject - 服务器上下文

**文件**: `app/Http/Middleware/Activity/ServerSubject.php`

**作用**: 当路由参数包含 `server` 时，自动设置:
- actor: 当前用户
- subject: Server 模型实例

### 4.3 TrackAPIKey - API密钥追踪

**文件**: `app/Http/Middleware/Activity/TrackAPIKey.php`

**作用**: 检测请求是否使用 API Key 认证，如果是则记录 `api_key_id`。

**路由注册** (`routes/api-client.php:6-7`):
```php
Route::middleware([AccountSubject::class])->group(function () { ... });
Route::middleware([ServerSubject::class, TrackAPIKey::class])->group(function () { ... });
```

## 五、事件监听机制

### 5.1 ActivityLogged 事件

**文件**: `app/Events/ActivityLogged.php`

**触发时机**: ActivityLog 模型创建后 (`ActivityLog.php:153-155`)
```php
static::created(function (self $model) {
    Event::dispatch(new ActivityLogged($model));
});
```

**辅助方法**:
- `is(string $event)`: 判断事件类型
- `isServerEvent()`: 是否为 `server:` 前缀的事件
- `isSystem()`: 是否为系统操作（无操作者）
- `actor()`: 获取操作者模型

### 5.2 AuthenticationListener - 认证事件监听

**文件**: `app/Listeners/AuthenticationListener.php`

**订阅事件** (`subscribe()` 方法, 第39-44行):
- `Illuminate\Auth\Events\Failed` → 登录失败 → `auth:fail`
- `Pterodactyl\Events\Auth\DirectLogin` → 登录成功 → `auth:success`
- `Illuminate\Auth\Events\PasswordReset` → 密码重置 → `event:password-reset`

**处理逻辑示例** (`login()` 方法, 第18-32行):
```php
public function login(Failed|DirectLogin $event): void
{
    $activity = Activity::withRequestMetadata()->subject($event->user);
    // 登录失败时记录凭证信息
    $activity->event($event instanceof Failed ? 'auth:fail' : 'auth:success')->log();
}
```

## 六、日志写入流程示例

### 6.1 控制器中直接使用

**文件**: `app/Http/Controllers/Api/Client/Servers/BackupController.php:80`
```php
$backup = Activity::event('server:backup.start')->transaction(function ($log) use ($action, $server, $request) {
    $log->subject($server);
    $log->property('is_locked', $request->boolean('is_locked'));
    
    return $action->handle($server, $request->input('name'));
});
```

**流程说明**:
1. `event()` 设置事件类型
2. `transaction()` 开启批次和数据库事务
3. 回调中执行业务逻辑
4. 事务成功后自动写入日志

### 6.2 Wings 节点远程日志

**文件**: `app/Http/Controllers/Api/Remote\ActivityProcessingController.php`

**作用**: 接收来自 Wings 守护进程的活动日志（如服务器启动、停止、控制台命令等）。

**处理流程**:
1. 批量接收多个服务器的日志数据
2. 验证服务器和用户存在性
3. 解析时间戳（处理时区转换）
4. 批量插入 `activity_logs` 和 `activity_log_subjects`

**路由**: `POST /api/remote/activity` (`routes/api-remote.php:11`)

## 七、查询与展示

### 7.1 账户活动日志 API

**文件**: `app/Http/Controllers/Api/Client/ActivityLogController.php`

**路由**: `GET /api/client/account/activity`

**查询逻辑**:
```php
QueryBuilder::for($request->user()->activity())
    ->with('actor')
    ->allowedFilters([AllowedFilter::partial('event')])
    ->allowedSorts(['timestamp'])
    ->whereNotIn('activity_logs.event', ActivityLog::DISABLED_EVENTS)
    ->paginate(25);
```

### 7.2 服务器活动日志 API

**文件**: `app/Http/Controllers/Api/Client/Servers/ActivityLogController.php`

**路由**: `GET /api/client/servers/{server}/activity`

**安全特性**:
- 权限检查: `Permission::ACTION_ACTIVITY_READ`
- 可配置隐藏管理员活动 (`activity.hide_admin_activity`)
- 过滤非服务器子用户的管理员操作

### 7.3 数据转换层

**文件**: `app/Transformers/Api/Client/ActivityLogTransformer.php`

**转换规则**:
- `id`: 使用 `sha1($model->id)` 生成前端唯一标识（非安全用途）
- `is_api`: 判断是否为 API 调用
- `ip`: 仅本人或管理员可见
- `properties`: 
  - 数组类型转换为 `{key}_count` 字段
  - 非本人的 IP 地址显示为 `[hidden]`
  - `directory` 字段自动规范化路径格式
- `has_additional_metadata`: 智能判断是否有未在描述中展示的额外属性

**国际化处理**:
- 通过 `trans('activity.' . str_replace(':', '.', $event))` 获取事件描述
- 解析描述字符串中的 `:key` 占位符，判断哪些属性已在描述中展示

### 7.4 前端展示

**API Hooks**:
- `resources/scripts/api/account/activity.ts` - 账户活动日志
- `resources/scripts/api/server/activity.ts` - 服务器活动日志

**组件**:
- `ActivityLogContainer.tsx` - 列表容器，支持分页和筛选
- `ActivityLogEntry.tsx` - 单条日志展示

**技术栈**:
- `useSWR` 数据获取与缓存
- 支持按 `event`、`ip` 筛选
- 支持按 `timestamp` 排序
- URL Hash 同步筛选条件

## 八、配置与清理

### 8.1 配置文件

**文件**: `config/activity.php`
```php
return [
    'prune_days' => env('APP_ACTIVITY_PRUNE_DAYS', 90),      // 日志保留天数
    'hide_admin_activity' => env('APP_ACTIVITY_HIDE_ADMIN', false),  // 隐藏管理员活动
];
```

### 8.2 自动清理

**实现**: `ActivityLog::prunable()` (第136-143行)
- Laravel 内置的 `MassPrunable` trait
- 配合 `model:prune` 命令使用
- 清理超过 `prune_days` 配置的日志记录

### 8.3 禁用事件

**常量**: `ActivityLog::DISABLED_EVENTS = ['server:file.upload']`
- 这些事件仍会被记录，但不会在 API 响应中返回

## 九、关键设计模式

### 9.1 Scoped 容器绑定

```php
$this->app->scoped(ActivityLogBatchService::class);
```
- 每个请求生命周期内共享同一实例
- 确保同一请求中的日志共享批次信息

### 9.2 链式调用 (Fluent Interface)

```php
Activity::event('...')->subject(...)->property(...)->log();
```
- 提高可读性，减少临时变量
- 每个方法返回 `$this`

### 9.3 延迟加载

`getActivity()` 方法仅在首次调用时创建 ActivityLog 实例，避免不必要的对象创建。

### 9.4 多态关联

使用 `morphTo` 和 `morphToMany` 支持多种模型类型作为 actor 和 subject，无需为每种类型创建单独字段。

## 十、数据流总图

```
┌─────────────────────────────────────────────────────────────────┐
│                    请求进入 (Request)                            │
└─────────────────────────────┬───────────────────────────────────┘
                              │
┌─────────────────────────────▼───────────────────────────────────┐
│  中间件层 (Middleware)                                           │
│  ├─ AccountSubject → LogTarget::setActor/user                   │
│  ├─ ServerSubject  → LogTarget::setActor/user + setSubject/server│
│  └─ TrackAPIKey    → LogTarget::setApiKeyId                     │
└─────────────────────────────┬───────────────────────────────────┘
                              │
┌─────────────────────────────▼───────────────────────────────────┐
│  业务逻辑层 (Controller/Service)                                │
│  ├─ Activity::event('type')                                     │
│  ├─ Activity::subject(...)                                      │
│  ├─ Activity::property(...)                                     │
│  └─ Activity::log() / transaction()                             │
└─────────────────────────────┬───────────────────────────────────┘
                              │
┌─────────────────────────────▼───────────────────────────────────┐
│  ActivityLogService::save()                                     │
│  ├─ 事务插入 activity_logs                                      │
│  └─ 批量插入 activity_log_subjects                              │
└─────────────────────────────┬───────────────────────────────────┘
                              │
┌─────────────────────────────▼───────────────────────────────────┐
│  ActivityLog::created 模型事件                                   │
│  └─ Event::dispatch(new ActivityLogged($model))                 │
└─────────────────────────────┬───────────────────────────────────┘
                              │
┌─────────────────────────────▼───────────────────────────────────┐
│  事件监听 (可选)                                                 │
│  └─ 监听 ActivityLogged 事件的 Listener                          │
└─────────────────────────────┬───────────────────────────────────┘
                              │
┌─────────────────────────────▼───────────────────────────────────┐
│  查询展示                                                        │
│  ├─ API Controller → QueryBuilder 分页查询                       │
│  ├─ Transformer → 数据转换与权限控制                             │
│  └─ 前端组件 → SWR 缓存 + 分页展示                               │
└─────────────────────────────────────────────────────────────────┘
```
