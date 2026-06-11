# Pterodactyl Panel 通知偏好与队列投递机制分析

## 目录
1. [通知系统整体架构](#1-通知系统整体架构)
2. [核心澄清：站内信到底是什么](#2-核心澄清站内信到底是什么)
3. [渠道开关粒度分析](#3-渠道开关粒度分析)
4. [合并去重策略](#4-合并去重策略)
5. [失败重试与降级机制](#5-失败重试与降级机制)
6. [同步 vs 异步投递路径深度分析](#6-同步-vs-异步投递路径深度分析)
7. [关键服务器告警场景分析](#7-关键服务器告警场景分析)

---

## 1. 通知系统整体架构

### 1.1 两条独立的消息流水线

Pterodactyl Panel 实际上存在**两条完全独立**的消息记录流水线，这是理解"站内信收到了但邮件没到"的关键：

```
┌─────────────────────────────────────────────────────────────────┐
│                        业务事件触发                              │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ├───────────────────────────────────┐
                              │                                   │
                              ▼                                   ▼
              ┌───────────────────────────┐           ┌───────────────────────────┐
              │   通知系统 (Notification)  │           │  活动日志 (Activity Log)   │
              └───────────────────────────┘           └───────────────────────────┘
                              │                                   │
                              ▼                                   ▼
                  邮件渠道 (mail) 发送                  写入 activity_log 表
                              │                                   │
                              ▼                                   ▼
                   可能成功/失败/丢失                    必然成功写入
                              │                                   │
                              ▼                                   ▼
                   用户收到邮件通知                    用户通过前端"Activity"页面查看
```

### 1.2 核心组件位置

| 组件类型 | 文件路径 | 说明 |
|---------|---------|------|
| 通知类 | [app/Notifications/](file:///d:/fz/0508-3/solo-dogfeeding/code/203-panel/app/Notifications) | 6个通知类，均继承 `Illuminate\Notifications\Notification` |
| 用户模型 | [app/Models/User.php](file:///d:/fz/0508-3/solo-dogfeeding/code/203-panel/app/Models/User.php) | 使用 `Notifiable` trait，具备接收通知能力 |
| 事件服务提供者 | [app/Providers/EventServiceProvider.php](file:///d:/fz/0508-3/solo-dogfeeding/code/203-panel/app/Providers/EventServiceProvider.php) | 配置事件与监听器映射 |
| 活动日志服务 | [app/Services/Activity/ActivityLogService.php](file:///d:/fz/0508-3/solo-dogfeeding/code/203-panel/app/Services/Activity/ActivityLogService.php) | 活动日志核心服务 |
| 活动日志控制器 | [app/Http/Controllers/Api/Client/ActivityLogController.php](file:///d:/fz/0508-3/solo-dogfeeding/code/203-panel/app/Http/Controllers/Api/Client/ActivityLogController.php) | 账户活动日志 API |
| 活动日志前端 | [resources/scripts/components/dashboard/activity/ActivityLogContainer.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/203-panel/resources/scripts/components/dashboard/activity/ActivityLogContainer.tsx) | 活动日志展示页面 |
| 队列配置 | [config/queue.php](file:///d:/fz/0508-3/solo-dogfeeding/code/203-panel/config/queue.php) | 队列连接、失败任务配置 |
| 邮件配置 | [config/mail.php](file:///d:/fz/0508-3/solo-dogfeeding/code/203-panel/config/mail.php) | 邮件驱动、failover 配置 |
| 通知配置 | [config/pterodactyl.php](file:///d:/fz/0508-3/solo-dogfeeding/code/203-panel/config/pterodactyl.php) | 全局通知开关配置 |

### 1.3 通知类清单

| 通知类 | 触发点 | 触发方式 | 是否队列化 | 实际发送方式 |
|-------|-------|---------|-----------|-------------|
| [AccountCreated](file:///d:/fz/0508-3/solo-dogfeeding/code/203-panel/app/Notifications/AccountCreated.php) | [UserCreationService.php](file:///d:/fz/0508-3/solo-dogfeeding/code/203-panel/app/Services/Users/UserCreationService.php#L54) | `$user->notify()` | 是 | 异步队列 |
| [AddedToServer](file:///d:/fz/0508-3/solo-dogfeeding/code/203-panel/app/Notifications/AddedToServer.php) | [SubuserObserver.php](file:///d:/fz/0508-3/solo-dogfeeding/code/203-panel/app/Observers/SubuserObserver.php#L27) | `$user->notify()` | 是 | 异步队列 |
| [RemovedFromServer](file:///d:/fz/0508-3/solo-dogfeeding/code/203-panel/app/Notifications/RemovedFromServer.php) | [SubuserObserver.php](file:///d:/fz/0508-3/solo-dogfeeding/code/203-panel/app/Observers/SubuserObserver.php#L49) | `$user->notify()` | 是 | 异步队列 |
| [ServerInstalled](file:///d:/fz/0508-3/solo-dogfeeding/code/203-panel/app/Notifications/ServerInstalled.php) | [ServerInstallController.php](file:///d:/fz/0508-3/solo-dogfeeding/code/203-panel/app/Http/Controllers/Api/Remote/Servers/ServerInstallController.php#L82-L84) | 事件触发 → `sendNow()` | 是（声明） | 同步执行 |
| [SendPasswordReset](file:///d:/fz/0508-3/solo-dogfeeding/code/203-panel/app/Notifications/SendPasswordReset.php) | [User.php](file:///d:/fz/0508-3/solo-dogfeeding/code/203-panel/app/Models/User.php#L218) | `$user->notify()` | 是 | 异步队列 |
| [MailTested](file:///d:/fz/0508-3/solo-dogfeeding/code/203-panel/app/Notifications/MailTested.php) | [MailController.php](file:///d:/fz/0508-3/solo-dogfeeding/code/203-panel/app/Http/Controllers/Admin/Settings/MailController.php) | `$user->notify()` | 否 | 同步执行 |

---

## 2. 核心澄清：站内信到底是什么

### 2.1 notifications 表的实际状态

**重要发现：`notifications` 表存在但从未被读取！**

#### 2.1.1 表结构存在

[2016_09_04_182835_create_notifications_table.php](file:///d:/fz/0508-3/solo-dogfeeding/code/203-panel/database/migrations/2016_09_04_182835_create_notifications_table.php) 定义了标准的 Laravel 数据库通知表结构：

```php
Schema::create('notifications', function (Blueprint $table) {
    $table->string('id')->primary();
    $table->string('type');
    $table->morphs('notifiable');
    $table->text('data');
    $table->timestamp('read_at')->nullable();
    $table->timestamps();
});
```

#### 2.1.2 但从未被读取

通过全代码库搜索确认：

- ❌ 没有任何代码使用 `->notifications` 关系
- ❌ 没有任何代码使用 `unreadNotifications` 或 `readNotifications`
- ❌ 没有任何 API 端点返回 notifications 表数据
- ❌ 前端没有任何 `Notification` 类型定义（只有 `ActivityLog`）
- ❌ 没有任何 Blade 模板引用 notifications

**结论**：`notifications` 表是 Laravel 框架自带的迁移遗留物，在本项目中**完全未被使用**。

### 2.2 "站内信" = 活动日志系统

用户所说的"站内信收到了"，实际上是通过**活动日志系统（Activity Log）**展示的。

#### 2.2.1 活动日志完整展示链路

```
活动日志写入 → API 读取 → 前端展示
    │            │           │
    │            │           └── 前端组件链路
    │            │               │
    │            │               ├── [ActivityLogContainer.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/203-panel/resources/scripts/components/dashboard/activity/ActivityLogContainer.tsx)
    │            │               │   └── 使用 useActivityLogs hook
    │            │               ├── [activity.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/203-panel/resources/scripts/api/account/activity.ts)
    │            │               │   └── useActivityLogs() 调用 /api/client/account/activity
    │            │               └── [ActivityLogEntry.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/203-panel/resources/scripts/components/elements/activity/ActivityLogEntry.tsx)
    │            │                   └── 单条活动日志渲染
    │            │
    │            └── API 链路
    │                │
    │                ├── [ActivityLogController.php](file:///d:/fz/0508-3/solo-dogfeeding/code/203-panel/app/Http/Controllers/Api/Client/ActivityLogController.php)
    │                │   └── 读取 activity_log 表，分页返回
    │                └── [ActivityLogTransformer.php](file:///d:/fz/0508-3/solo-dogfeeding/code/203-panel/app/Transformers/Api/Client/ActivityLogTransformer.php)
    │                    └── 转换为前端可用格式
    │
    └── 写入链路
        │
        ├── [ActivityLogService.php](file:///d:/fz/0508-3/solo-dogfeeding/code/203-panel/app/Services/Activity/ActivityLogService.php)
        │   └── Activity::event()->subject()->log() 调用
        └── [ActivityLog.php](file:///d:/fz/0508-3/solo-dogfeeding/code/203-panel/app/Models/ActivityLog.php)
            └── created 事件触发 ActivityLogged 事件
```

#### 2.2.2 前端入口

在 [routes.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/203-panel/resources/scripts/routers/routes.ts#L63-L67) 中定义了活动日志页面路由：

```typescript
{
    path: '/activity',
    name: 'Activity',
    component: ActivityLogContainer,
},
```

用户可以通过 `/account/activity` 路径访问活动日志页面，这就是用户看到的"站内信"。

#### 2.2.3 活动日志 vs 通知系统对比

| 特性 | 活动日志 (Activity Log) | 通知系统 (Notification) |
|-----|------------------------|-----------------------|
| 存储表 | `activity_log` | `notifications`（未使用） |
| 触发方式 | 显式调用 `Activity::event()->log()` | 事件驱动或手动 `notify()` |
| 邮件投递 | 不发送邮件 | 发送邮件 |
| 前端展示 | `/account/activity` 页面 | 无前端入口 |
| 可靠性 | 数据库事务内写入，必然成功 | 依赖邮件服务，可能失败 |
| 失败处理 | 事务回滚时才会失败 | 记录到 `failed_jobs` 表 |
| 渠道配置 | 无渠道概念 | 仅 mail 渠道 |
| 数据读取 | 完整的 API + 前端展示 | 从未被读取 |

---

## 3. 渠道开关粒度分析

### 3.1 渠道定义方式

Pterodactyl Panel 的通知渠道采用**硬编码 + 全局配置**两层控制：

#### 3.1.1 通知级别：硬编码渠道

每个通知类通过 `via()` 方法硬编码指定投递渠道。**所有通知类均只返回 `['mail']`**，即仅通过邮件渠道发送。

```php
// app/Notifications/AccountCreated.php
public function via(): array
{
    return ['mail'];
}
```

**关键特征**：
- ❌ 无用户级别的渠道偏好设置
- ❌ 无角色级别的渠道偏好设置
- ❌ 无通知类型级别的用户自定义开关
- ✅ 所有通知硬编码仅使用 mail 渠道
- ❌ 从未使用 `'database'` 渠道（虽然表存在）

#### 3.1.2 全局级别：配置开关

在 [config/pterodactyl.php](file:///d:/fz/0508-3/solo-dogfeeding/code/203-panel/config/pterodactyl.php#L174-L179) 中，仅提供了服务器安装相关的全局邮件通知开关：

```php
'email' => [
    'send_install_notification' => env('PTERODACTYL_SEND_INSTALL_NOTIFICATION', true),
    'send_reinstall_notification' => env('PTERODACTYL_SEND_REINSTALL_NOTIFICATION', true),
],
```

**使用位置**：[ServerInstallController.php](file:///d:/fz/0508-3/solo-dogfeeding/code/203-panel/app/Http/Controllers/Api/Remote/Servers/ServerInstallController.php#L80-L85)

```php
$isInitialInstall = is_null($server->installed_at);
if ($isInitialInstall && config()->get('pterodactyl.email.send_install_notification', true)) {
    $this->eventDispatcher->dispatch(new ServerInstalled($server));
} elseif (!$isInitialInstall && config()->get('pterodactyl.email.send_reinstall_notification', true)) {
    $this->eventDispatcher->dispatch(new ServerInstalled($server));
}
```

### 3.2 渠道开关粒度总结

| 粒度级别 | 是否支持 | 实现方式 |
|---------|---------|---------|
| 全局总开关 | 部分支持 | 通过 `.env` 配置控制特定通知类型（仅服务器安装/重装） |
| 通知类型级别 | 部分支持 | 仅服务器安装/重装通知有独立开关 |
| 用户级别 | 不支持 | 无用户通知偏好表，无法按用户配置 |
| 角色级别 | 不支持 | 无角色通知偏好配置 |
| 渠道级别 | 不支持 | 所有通知硬编码仅使用 mail 渠道 |

---

## 4. 合并去重策略

### 4.1 通知层面的去重

当前代码库中，**通知系统本身没有实现合并去重机制**：

- ❌ 所有通知类均未实现 `ShouldBeUnique` 接口
- ❌ 无通知去重标识（`uniqueId` 方法）
- ❌ 相同事件多次触发会产生多封相同邮件

### 4.2 队列层面的去重

虽然通知本身没有去重，但某些非通知类 Job 实现了队列去重：

#### 4.2.1 RevokeSftpAccessJob 的去重实现

[RevokeSftpAccessJob.php](file:///d:/fz/0508-3/solo-dogfeeding/code/203-panel/app/Jobs/RevokeSftpAccessJob.php#L19-L39) 实现了 `ShouldBeUnique` 接口：

```php
class RevokeSftpAccessJob implements ShouldQueue, ShouldBeUnique
{
    use Queueable;

    public int $tries = 3;
    public int $maxExceptions = 1;

    public function uniqueId(): string
    {
        $target = $this->target instanceof Node ? "node:{$this->target->uuid}" : "server:{$this->target->uuid}";
        return "revoke-sftp:{$this->user}:{$target}";
    }
}
```

**去重规则**：
- 基于用户 + 目标（节点/服务器）的组合作为唯一标识
- 同一时刻只能有一个相同标识的 Job 在队列中
- 避免重复的 SFTP 权限撤销操作

### 4.3 活动日志的去重

活动日志系统在 [ActivityLogService.php](file:///d:/fz/0508-3/solo-dogfeeding/code/203-panel/app/Services/Activity/ActivityLogService.php#L77-L83) 中实现了 subject 去重：

```php
foreach ($this->subjects as $entry) {
    // If this subject is already tracked in our array of subjects just skip over
    // it and move on to the next one in the list.
    if ($entry->is($subject)) {
        continue 2;
    }
}
```

**去重规则**：
- 同一活动日志条目避免关联重复的 subject
- 仅在单次 log 调用内有效，不跨请求去重

### 4.4 合并去重策略总结

| 层面 | 去重机制 | 适用范围 |
|-----|---------|---------|
| 通知系统 | 无 | - |
| 队列 Job | `ShouldBeUnique` 接口 | 仅部分非通知类 Job（如 RevokeSftpAccessJob） |
| 活动日志 | Subject 列表内去重 | 单次 log 调用内 |
| 邮件渠道 | 无 | - |
| 数据库通知 | 未使用 | - |

---

## 5. 失败重试与降级机制

### 5.1 队列重试机制

#### 5.1.1 通知类的重试配置

**所有通知类均未显式配置重试参数**，使用 Laravel 默认值：

- `$tries`：未定义，使用 worker 命令行参数 `--tries` 值（默认 1）
- `$backoff`：未配置，失败后立即重试
- `retry_after`：由队列连接配置决定（redis 默认 90 秒）
- ❌ 所有通知类均未实现 `failed()` 方法

#### 5.1.2 队列连接配置

[config/queue.php](file:///d:/fz/0508-3/solo-dogfeeding/code/203-panel/config/queue.php#L64-L71) 中 Redis 队列配置：

```php
'redis' => [
    'driver' => 'redis',
    'connection' => env('REDIS_QUEUE_CONNECTION', 'default'),
    'queue' => env('REDIS_QUEUE', env('QUEUE_STANDARD', 'standard')),
    'retry_after' => (int) env('REDIS_QUEUE_RETRY_AFTER', 90),
    'block_for' => null,
    'after_commit' => false,
],
```

**关键参数 `retry_after`**：
- 90 秒后任务未完成则重新入队
- 如果邮件发送超时或 worker 异常退出，90 秒后会自动重试

### 5.2 失败任务记录

#### 5.2.1 失败任务存储

[config/queue.php](file:///d:/fz/0508-3/solo-dogfeeding/code/203-panel/config/queue.php#L87-L91) 配置：

```php
'failed' => [
    'driver' => env('QUEUE_FAILED_DRIVER', 'database-uuids'),
    'database' => env('DB_CONNECTION', 'mysql'),
    'table' => 'failed_jobs',
],
```

**失败任务记录内容**：
- 存储在 `failed_jobs` 表
- 包含异常堆栈、队列名称、连接名称、payload 等信息
- 可通过 `php artisan queue:failed` 查看
- 可通过 `php artisan queue:retry <id>` 重试

#### 5.2.2 同步 vs 异步失败落点

| 发送方式 | 失败落点 | 记录方式 |
|---------|---------|---------|
| **同步发送 (sendNow)** | 调用线程直接抛出异常 | 1. 由调用方 try-catch 处理<br>2. 未捕获则进入全局异常处理器<br>3. 记录到 `laravel.log` |
| **异步队列 (ShouldQueue)** | 队列 worker 线程处理 | 1. 按 `retry_after` 自动重试<br>2. 超过重试次数后写入 `failed_jobs` 表<br>3. 异常记录到 `laravel.log` |

### 5.3 邮件渠道的降级机制

#### 5.3.1 Failover Mailer

[config/mail.php](file:///d:/fz/0508-3/solo-dogfeeding/code/203-panel/config/mail.php#L73-L79) 配置了 failover 邮件驱动：

```php
'failover' => [
    'transport' => 'failover',
    'mailers' => [
        'smtp',
        'log',
    ],
],
```

**降级逻辑**：
- 主邮件器：smtp
- 备用邮件器：log（将邮件内容写入日志）
- 当 smtp 发送失败时，自动降级到 log 渠道

**注意**：默认 mailer 是 `smtp`（见 `config/mail.php` 第 15 行），**只有显式将 `MAIL_MAILER` 设置为 `failover` 时才会触发自动降级**。

#### 5.3.2 邮件异常处理

[Handler.php](file:///d:/fz/0508-3/solo-dogfeeding/code/203-panel/app/Exceptions/Handler.php#L86-L88) 中对邮件传输异常做了特殊处理：

```php
$this->reportable(function (TransportException $ex) {
    $ex = $this->generateCleanedExceptionStack($ex);
});
```

这个处理会清理异常堆栈中的敏感信息，避免邮件密码等凭证泄露到日志中。

### 5.4 参考：其他 Job 的重试策略

作为对比，[RevokeSftpAccessJob.php](file:///d:/fz/0508-3/solo-dogfeeding/code/203-panel/app/Jobs/RevokeSftpAccessJob.php#L50-L55) 展示了一个完整的重试模式：

```php
try {
    $repository->setNode($node)->deauthorize(
        $this->user,
        $this->target instanceof Server ? [$this->target->uuid] : []
    );
} catch (DaemonConnectionException) {
    // Keep retrying this job with a longer and longer backoff until we hit three
    // attempts at which point we stop and will assume the node is fully offline
    // and we are just wasting time.
    $this->release($this->attempts() * 10);
}
```

**重试策略**：
- 最大重试次数：3 次（`$tries = 3`）
- 退避策略：线性退避（第 N 次重试延迟 N*10 秒）
- 仅对特定异常（DaemonConnectionException）进行重试
- 其他异常直接失败

### 5.5 失败重试与降级机制总结

| 机制 | 通知系统 | 其他 Job (如 RevokeSftpAccessJob) |
|-----|---------|-----------------------------------|
| 重试次数 | 默认 1 次（未配置） | 显式配置 3 次 |
| 退避策略 | 无（立即重试） | 线性退避（N*10 秒） |
| 失败回调 | 无 `failed()` 方法 | 部分 Job 有清理逻辑 |
| 渠道降级 | mail failover 配置（默认未启用） | 不适用 |
| 失败存储 | `failed_jobs` 表（异步） | `failed_jobs` 表 |
| 异常日志 | `laravel.log` | `laravel.log` |

---

## 6. 同步 vs 异步投递路径深度分析

### 6.1 两条投递路径概览

```
通知发送
    │
    ├───────────────────────────────────────────┐
    │                                           │
    ▼                                           ▼
同步发送路径 (sendNow)                 异步发送路径 (ShouldQueue + notify())
    │                                           │
    ├─ 当前线程直接执行                        ├─ 包装为 SendQueuedNotifications Job
    ├─ MailChannel::send() 立即执行             ├─ Job 推入 Redis 队列
    ├─ 成功：返回响应                          ├─ Queue Worker 后台异步处理
    ├─ 失败：抛出异常 → 日志记录                ├─ 成功：Job 删除
    │                                           ├─ 失败：重试 → 超过次数 → failed_jobs
    ▼                                           ▼
用户收到邮件（快，但阻塞请求）            用户可能收到邮件（慢，不阻塞请求）
```

### 6.2 同步发送路径：ServerInstalled 深度分析

#### 6.2.1 为什么 ServerInstalled 是同步的？

[ServerInstalled.php](file:///d:/fz/0508-3/solo-dogfeeding/code/203-panel/app/Notifications/ServerInstalled.php) 是一个特殊的通知类：

```php
class ServerInstalled extends Notification implements ShouldQueue, ReceivesEvents
{
    use Queueable;

    public function handle(Event|Installed $event): void
    {
        $event->server->loadMissing('user');

        $this->server = $event->server;
        $this->user = $event->server->user;

        // Since we are calling this notification directly from an event listener we need to fire off the dispatcher
        // to send the email now. Don't use send() or you'll end up firing off two different events.
        Container::getInstance()->make(Dispatcher::class)->sendNow($this->user, $this);
    }

    public function via(): array
    {
        return ['mail'];
    }
}
```

**关键矛盾**：
- 类声明实现 `ShouldQueue` 接口
- 但在 `handle()` 方法中显式调用 `sendNow()` 同步发送
- 注释说明：避免重复触发事件

#### 6.2.2 完整调用链路

```
1. Wings 上报服务器安装完成
    ↓
2. [ServerInstallController.php](file:///d:/fz/0508-3/solo-dogfeeding/code/203-panel/app/Http/Controllers/Api/Remote/Servers/ServerInstallController.php#L80-L84)
   → 检查全局开关
   → 触发 ServerInstalled 事件
    ↓
3. [EventServiceProvider.php](file:///d:/fz/0508-3/solo-dogfeeding/code/203-panel/app/Providers/EventServiceProvider.php#L26)
   → 事件映射到 ServerInstalledNotification 监听器
    ↓
4. [ServerInstalled.php::handle()](file:///d:/fz/0508-3/solo-dogfeeding/code/203-panel/app/Notifications/ServerInstalled.php#L31-L41)
   → 作为事件监听器被调用
   → 显式调用 Dispatcher::sendNow()
    ↓
5. Illuminate\Notifications\ChannelManager
   → 调用 via() 获取 ['mail']
   → 解析 MailChannel
    ↓
6. Illuminate\Notifications\Channels\MailChannel::send()
   → 调用 toMail() 获取 MailMessage
   → 通过 Mailer 发送
    ↓
7. 成功：结束
   失败：抛出 TransportException → 异常处理器 → 日志记录
```

#### 6.2.3 同步发送的失败落点

如果同步发送失败：
1. `TransportException` 异常向上抛出
2. 由于是在 HTTP 请求处理中，异常会被 [Handler.php](file:///d:/fz/0508-3/solo-dogfeeding/code/203-panel/app/Exceptions/Handler.php) 捕获
3. 异常堆栈被清理（移除敏感信息）
4. 记录到 `storage/logs/laravel.log`
5. **不会**记录到 `failed_jobs` 表（因为是同步执行）
6. API 调用者收到 500 错误响应

### 6.3 异步发送路径：AddedToServer 深度分析

以 [AddedToServer.php](file:///d:/fz/0508-3/solo-dogfeeding/code/203-panel/app/Notifications/AddedToServer.php) 为例：

```php
class AddedToServer extends Notification implements ShouldQueue
{
    use Queueable;

    public function via(): array
    {
        return ['mail'];
    }

    public function toMail(): MailMessage
    {
        // ...
    }
}
```

#### 6.3.1 完整调用链路

```
1. 子用户被创建
    ↓
2. [SubuserObserver.php::created()](file:///d:/fz/0508-3/solo-dogfeeding/code/203-panel/app/Observers/SubuserObserver.php#L23-L32)
   → 触发 Subuser\Created 事件
   → 调用 $user->notify(new AddedToServer(...))
    ↓
3. Illuminate\Notifications\Notifiable::notify()
   → 解析通知类
   → 检测到 ShouldQueue 接口
   → 创建 SendQueuedNotifications Job
    ↓
4. Illuminate\Bus\Dispatcher::dispatch()
   → Job 序列化
   → 推入 Redis 队列（queue: standard）
   → HTTP 请求继续执行，立即返回响应
    ↓
======================== 以下为异步执行 ========================
    ↓
5. Queue Worker 进程
   → 从 Redis 取出 Job
   → 反序列化
   → 执行 handle()
    ↓
6. Illuminate\Notifications\ChannelManager
   → 调用 via() 获取 ['mail']
   → 解析 MailChannel
    ↓
7. Illuminate\Notifications\Channels\MailChannel::send()
   → 调用 toMail() 获取 MailMessage
   → 通过 Mailer 发送
    ↓
8. 成功：Job 从队列删除
   失败：
      → 检查重试次数
      → 未超过：release() 回队列（按 retry_after 90秒后重试）
      → 已超过：写入 failed_jobs 表 → 记录日志
```

#### 6.3.2 异步发送的失败落点

如果异步发送失败：
1. 第一次失败：90 秒后自动重试（由 `retry_after` 控制）
2. 第二次失败：由于通知类未配置 `$tries`，取决于 worker 配置
   - 如果 worker 启动时 `--tries=1`（默认）：立即写入 `failed_jobs` 表
   - 如果 worker 启动时 `--tries=3`：再重试 2 次后写入 `failed_jobs` 表
3. `failed_jobs` 表记录完整的异常堆栈和 payload
4. 异常同时记录到 `laravel.log`
5. **不会**影响原始 HTTP 请求（早已响应）

### 6.4 同步 vs 异步对比总结

| 维度 | 同步发送 (sendNow) | 异步发送 (ShouldQueue) |
|-----|-------------------|----------------------|
| 适用通知 | ServerInstalled | AccountCreated, AddedToServer, RemovedFromServer, SendPasswordReset |
| 执行时机 | 请求处理中立即执行 | 后台 worker 异步执行 |
| 阻塞请求 | 是（直到邮件发送完成或失败） | 否（Job 入队后立即返回） |
| 失败感知 | 请求立即收到错误 | 请求成功但邮件可能后续失败 |
| 重试机制 | 无（失败即结束） | retry_after 90 秒自动重试 |
| 失败记录 | laravel.log | laravel.log + failed_jobs 表 |
| 失败重试 | 需要手动重新触发整个业务流程 | php artisan queue:retry |
| 性能影响 | 慢（依赖 SMTP 响应速度） | 快（仅 Redis 操作） |
| 可靠性 | 低（SMTP 失败则用户收不到） | 中（有重试但仍可能最终失败） |

---

## 7. 关键服务器告警场景分析

### 7.1 "告警邮件未到但站内信收到"的根本原因

基于代码分析，这个现象的根本原因是：

**活动日志和邮件通知是两条独立的流水线，活动日志必然成功写入，而邮件通知可能因各种原因发送失败。**

#### 7.1.1 原因 1：活动日志 vs 通知系统分离

用户所说的"站内信"是活动日志，而"告警邮件"是通知系统的邮件。两者是完全独立的：

```
服务器告警事件触发
    │
    ├─ 写入活动日志（activity_log 表）
    │   └─ ✓ 必然成功（数据库事务内写入）
    │   └─ ✓ 用户在 Activity 页面看到
    │
    └─ 触发通知（邮件渠道）
        └─ ? 可能失败（SMTP 问题、队列问题等）
        └─ ✗ 用户收不到邮件
```

#### 7.1.2 原因 2：邮件发送失败的可能原因

邮件发送可能因为以下原因失败：

1. **SMTP 服务器不可达**：网络问题、防火墙、DNS 解析失败
2. **SMTP 认证失败**：用户名密码错误、API Key 过期
3. **邮件被拒收**：被标记为垃圾邮件、收件人地址不存在
4. **队列 Worker 未运行**：`php artisan queue:work` 没有启动
5. **邮件发送超时**：超过 `retry_after` 90 秒
6. **Worker 异常退出**：进程被 kill、内存不足等
7. **Redis 连接问题**：队列 Job 无法正常出队

#### 7.1.3 原因 3：通知偏好配置不存在

从代码来看，Pterodactyl Panel 本身：
- ❌ 没有用户级别的通知偏好设置
- ❌ 没有角色级别的通知偏好设置
- ❌ 没有渠道级别的开关配置
- ✅ 所有用户收到的通知类型和渠道完全相同

所以**不可能**是"用户关闭了邮件通知但打开了站内信"的情况。

### 7.2 邮件投递失败的排查路径

1. **检查队列状态**：
   ```bash
   # 确认 queue worker 是否在运行
   php artisan queue:work --tries=3

   # 检查队列大小
   redis-cli llen queues:standard
   ```

2. **检查失败任务**：
   ```bash
   # 列出所有失败任务
   php artisan queue:failed

   # 重试指定任务
   php artisan queue:retry <job-id>

   # 重试所有失败任务
   php artisan queue:retry all
   ```

3. **检查邮件日志**：
   ```bash
   # Laravel 日志
   tail -f storage/logs/laravel.log | grep -E "(TransportException|ERROR|email)"

   # 如果使用 log 驱动，邮件内容会直接写入日志
   ```

4. **检查 failed_jobs 表**：
   ```sql
   SELECT id, exception, failed_at FROM failed_jobs ORDER BY failed_at DESC LIMIT 10;
   ```

5. **测试邮件配置**：
   - 在 Admin → Settings → Mail 页面点击 "Send Test Email"
   - 使用 MailTested 通知类（同步发送，立即反馈结果）

### 7.3 改进建议

如果需要解决"邮件可能丢失"的问题，可以考虑以下改进：

#### 7.3.1 增加数据库通知渠道

在所有通知类的 `via()` 方法中添加 `'database'` 渠道：

```php
public function via(): array
{
    return ['mail', 'database'];
}
```

这样即使邮件发送失败，数据库通知仍然存在，可以开发前端展示页面让用户查看。

#### 7.3.2 实现通知偏好系统

1. 创建 `user_notification_preferences` 表，存储用户对每种通知类型的渠道偏好
2. 修改 `via()` 方法，根据用户偏好动态返回渠道列表
3. 提供前端界面让用户配置自己的通知偏好

#### 7.3.3 配置邮件重试策略

在所有通知类中添加重试配置：

```php
class AccountCreated extends Notification implements ShouldQueue
{
    use Queueable;

    public int $tries = 3;
    public int $backoff = 60; // 每次重试间隔 60 秒

    public function failed(\Throwable $exception)
    {
        // 记录告警、触发备用通知等
        Log::error('Notification failed', [
            'notification' => static::class,
            'user_id' => $this->user->id,
            'error' => $exception->getMessage(),
        ]);
    }
}
```

#### 7.3.4 启用 failover 邮件驱动

在 `.env` 中配置：
```env
MAIL_MAILER=failover
```

这样当 SMTP 发送失败时，会自动降级将邮件内容写入日志，至少可以事后追查。

#### 7.3.5 监控队列健康状态

1. 配置队列监控，当失败任务过多时告警
2. 定期清理 `failed_jobs` 表
3. 实现失败任务自动重试机制

---

## 附：通知发送时序图

### 异步通知（以 AddedToServer 为例）

```
[HTTP Request Thread]                          [Queue Worker Thread]
       │                                               │
       │  SubuserObserver.created()                   │
       │       → event(new Subuser\Created)            │
       │       → $user->notify(new AddedToServer)      │
       │                                               │
       │  Notifiable::notify()                         │
       │   ├─ 检测 ShouldQueue                         │
       │   └─ 分发 SendQueuedNotifications Job →→→→→→→│ 入队
       │                                               │
       │  HTTP Response 返回                           │
       │                                               │
       │                                               │  Job 出队
       │                                               │      ↓
       │                                               │  ChannelManager
       │                                               │   ├─ via() → ['mail']
       │                                               │   └─ MailChannel::send()
       │                                               │      ↓
       │                                               │  Mailer::send()
       │                                               │   ├─ 成功 → Job 删除
       │                                               │   └─ 失败 → 重试 → failed_jobs
```

### 同步通知（以 ServerInstalled 为例）

```
[HTTP Request Thread]
       │
       │  ServerInstallController::store()
       │   ├─ 更新服务器状态
       │   └─ 触发 ServerInstalled 事件
       │           ↓
       │  ServerInstalledNotification::handle()
       │   └─ Dispatcher::sendNow() （同步执行）
       │           ↓
       │  ChannelManager
       │   ├─ via() → ['mail']
       │   └─ MailChannel::send()
       │           ↓
       │  Mailer::send()
       │   ├─ 成功 → 继续执行 → HTTP 204 返回
       │   └─ 失败 → 抛出 TransportException
       │           ↓
       │  Handler::report()
       │   ├─ 清理异常堆栈
       │   └─ 记录到 laravel.log
       │           ↓
       │  HTTP 500 错误返回
```

### 活动日志链路

```
[业务代码]                              [数据库]                          [前端]
     │                                     │                                 │
     │  Activity::event('server:backup')   │                                 │
     │    → subject($server)               │                                 │
     │    → property('name', $name)        │                                 │
     │    → log()                          │                                 │
     │          │                            │                                 │
     │          └→ transaction begin        │                                 │
     │          └→ activity_logs INSERT     │                                 │
     │          └→ activity_log_subjects INSERT                             │
     │          └→ transaction commit       │                                 │
     │                                     │                                 │
     │                                     │  ActivityLog.created 事件        │
     │                                     │    → dispatch ActivityLogged     │
     │                                     │                                 │
     │                                     │                                 │  useActivityLogs()
     │                                     │                                 │    ↓
     │                                     │                                 │  GET /api/client/account/activity
     │                                     │                                 │    ↓
     │                                     │  ActivityLogController           │
     │                                     │    → paginate activity_logs     │
     │                                     │    → fractal transform          │
     │                                     │          ↓                       │
     │                                     │  JSON 响应 ←←←←←←←←←←←←←←←←←←←   │
     │                                     │                                 │    ↓
     │                                     │                                 │  ActivityLogContainer
     │                                     │                                 │    → 渲染 ActivityLogEntry 列表
```
