# Pterodactyl Panel 通知偏好与队列投递机制分析

## 目录
1. [通知系统整体架构](#1-通知系统整体架构)
2. [核心澄清：站内信到底是什么](#2-核心澄清站内信到底是什么)
3. [渠道开关粒度分析](#3-渠道开关粒度分析)
4. [合并去重策略](#4-合并去重策略)
5. [失败重试与降级机制](#5-失败重试与降级机制)
6. [同步 vs 异步投递路径深度分析](#6-同步-vs-异步投递路径深度分析)
7. [管理员测试邮件：MailTested 独立分析](#7-管理员测试邮件mailtested-独立分析)
8. [关键服务器告警场景分析](#8-关键服务器告警场景分析)

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
                  邮件渠道 (mail) 发送                 写入 activity_logs 表
                              │                                   │
                              ▼                                   ▼
                   可能成功/失败/丢失               生产环境吞异常，不保证成功
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
| 活动日志主表模型 | [app/Models/ActivityLog.php](file:///d:/fz/0508-3/solo-dogfeeding/code/203-panel/app/Models/ActivityLog.php) | 对应 `activity_logs` 表 |
| 活动日志关联模型 | [app/Models/ActivityLogSubject.php](file:///d:/fz/0508-3/solo-dogfeeding/code/203-panel/app/Models/ActivityLogSubject.php) | 对应 `activity_log_subjects` 表 |
| 活动日志 API 入口 | [app/Http/Controllers/Api/Client/ActivityLogController.php](file:///d:/fz/0508-3/solo-dogfeeding/code/203-panel/app/Http/Controllers/Api/Client/ActivityLogController.php) | 单动作控制器，`__invoke()` 方法处理 `/api/client/account/activity` |
| 服务器活动日志 API 入口 | [app/Http/Controllers/Api/Client/Servers/ActivityLogController.php](file:///d:/fz/0508-3/solo-dogfeeding/code/203-panel/app/Http/Controllers/Api/Client/Servers/ActivityLogController.php) | 单动作控制器，`__invoke()` 方法处理 `/api/client/servers/{server}/activity` |
| 活动日志前端容器 | [resources/scripts/components/dashboard/activity/ActivityLogContainer.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/203-panel/resources/scripts/components/dashboard/activity/ActivityLogContainer.tsx) | 活动日志展示页面 |
| 活动日志 API 封装 | [resources/scripts/api/account/activity.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/203-panel/resources/scripts/api/account/activity.ts) | `useActivityLogs()` Hook |
| 队列配置 | [config/queue.php](file:///d:/fz/0508-3/solo-dogfeeding/code/203-panel/config/queue.php) | 队列连接、失败任务配置 |
| 邮件配置 | [config/mail.php](file:///d:/fz/0508-3/solo-dogfeeding/code/203-panel/config/mail.php) | 邮件驱动、failover 配置 |
| 通知配置 | [config/pterodactyl.php](file:///d:/fz/0508-3/solo-dogfeeding/code/203-panel/config/pterodactyl.php) | 全局通知开关配置 |

### 1.3 通知类清单（分两类：普通用户通知 vs 管理员测试邮件）

#### 1.3.1 普通用户业务通知（5 个）

| 通知类 | 触发点 | 触发方式 | 是否队列化 | 实际发送方式 |
|-------|-------|---------|-----------|-------------|
| [AccountCreated](file:///d:/fz/0508-3/solo-dogfeeding/code/203-panel/app/Notifications/AccountCreated.php) | [UserCreationService.php](file:///d:/fz/0508-3/solo-dogfeeding/code/203-panel/app/Services/Users/UserCreationService.php#L54) | `$user->notify()` | 是 | 异步队列 |
| [AddedToServer](file:///d:/fz/0508-3/solo-dogfeeding/code/203-panel/app/Notifications/AddedToServer.php) | [SubuserObserver.php](file:///d:/fz/0508-3/solo-dogfeeding/code/203-panel/app/Observers/SubuserObserver.php#L27) | `$user->notify()` | 是 | 异步队列 |
| [RemovedFromServer](file:///d:/fz/0508-3/solo-dogfeeding/code/203-panel/app/Notifications/RemovedFromServer.php) | [SubuserObserver.php](file:///d:/fz/0508-3/solo-dogfeeding/code/203-panel/app/Observers/SubuserObserver.php#L49) | `$user->notify()` | 是 | 异步队列 |
| [ServerInstalled](file:///d:/fz/0508-3/solo-dogfeeding/code/203-panel/app/Notifications/ServerInstalled.php) | [ServerInstallController.php](file:///d:/fz/0508-3/solo-dogfeeding/code/203-panel/app/Http/Controllers/Api/Remote/Servers/ServerInstallController.php#L82-L84) | 事件触发 → 内联 `sendNow()` | 是（声明） | 同步执行 |
| [SendPasswordReset](file:///d:/fz/0508-3/solo-dogfeeding/code/203-panel/app/Notifications/SendPasswordReset.php) | [User.php](file:///d:/fz/0508-3/solo-dogfeeding/code/203-panel/app/Models/User.php#L218) | `$user->notify()` | 是 | 异步队列 |

#### 1.3.2 管理员测试邮件（1 个，独立流程）

| 通知类 | 触发点 | 触发方式 | 是否队列化 | 实际发送方式 |
|-------|-------|---------|-----------|-------------|
| [MailTested](file:///d:/fz/0508-3/solo-dogfeeding/code/203-panel/app/Notifications/MailTested.php) | [MailController::test()](file:///d:/fz/0508-3/solo-dogfeeding/code/203-panel/app/Http/Controllers/Admin/Settings/MailController.php#L77-L87) | `Notification::route()` 匿名通知路由 | 否 | 同步执行（带 try-catch） |

> **MailTested 的特殊性**：不通过 `$user->notify()` 触发，而是使用 Laravel 的 On-demand Notifications（匿名通知路由），调用方自带 try-catch 捕获异常，失败直接返回 500 错误消息给前端。详见第 7 节。

---

## 2. 核心澄清：站内信到底是什么

### 2.1 notifications 表的实际状态

**重要发现：`notifications` 表存在但从未被读取！**

#### 2.1.1 表结构存在（Laravel 框架迁移遗留）

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

#### 2.1.2 但从未被读取，也从未被写入

通过全代码库搜索确认：

- ❌ 没有任何代码使用 `->notifications` 关系读取数据
- ❌ 没有任何代码使用 `unreadNotifications` 或 `readNotifications`
- ❌ 没有任何通知类在 `via()` 中返回 `'database'` 渠道
- ❌ 没有任何 API 端点返回 notifications 表数据
- ❌ 前端 TypeScript 类型中没有 `Notification` 类型定义（只有 `ActivityLog`）
- ❌ 没有任何 Blade 模板引用 notifications

**结论**：`notifications` 表是 Laravel 框架自带的迁移遗留物，在本项目中**完全未被使用**——既没有写入，也没有读取。

### 2.2 "站内信" = 活动日志系统（Activity Log）

用户所说的"站内信收到了"，实际上是通过**活动日志系统**展示的。

#### 2.2.1 活动日志的实际表名

活动日志使用**两张表**（注意表名都是复数）：

| 表名 | 迁移文件 | 模型 | 用途 |
|-----|---------|------|------|
| `activity_logs` | [2022_05_28_135717_create_activity_logs_table.php](file:///d:/fz/0508-3/solo-dogfeeding/code/203-panel/database/migrations/2022_05_28_135717_create_activity_logs_table.php) | [ActivityLog.php](file:///d:/fz/0508-3/solo-dogfeeding/code/203-panel/app/Models/ActivityLog.php) | 存储活动日志主记录 |
| `activity_log_subjects` | [2022_05_29_140349_create_activity_log_actors_table.php](file:///d:/fz/0508-3/solo-dogfeeding/code/203-panel/database/migrations/2022_05_29_140349_create_activity_log_actors_table.php)（注意：迁移文件名写的是 actors，但实际创建的表名是 subjects） | [ActivityLogSubject.php](file:///d:/fz/0508-3/solo-dogfeeding/code/203-panel/app/Models/ActivityLogSubject.php) | 关联活动日志与关联对象（如 Server、User） |

> ⚠️ **命名陷阱**：第二个迁移文件名叫 `create_activity_log_actors_table`，但它实际执行 `Schema::create('activity_log_subjects', ...)`，真实表名是 `activity_log_subjects`，不要被文件名误导。

#### 2.2.2 活动日志写入链路（含异常处理）

写入入口：[ActivityLogService::log()](file:///d:/fz/0508-3/solo-dogfeeding/code/203-panel/app/Services/Activity/ActivityLogService.php#L133-L153)

```php
public function log(?string $description = null): ActivityLog
{
    $activity = $this->getActivity();

    if (!is_null($description)) {
        $activity->description = $description;
    }

    try {
        return $this->save();
    } catch (\Throwable $exception) {
        if (config('app.env') !== 'production') {
            /* @noinspection PhpUnhandledExceptionInspection */
            throw $exception;
        }

        Log::error($exception);
    }

    return $activity;
}
```

关键逻辑：
1. 调用 `save()` 方法执行实际写入（内部有数据库事务）
2. **用 try-catch 包裹了写入操作**
3. 非生产环境：写入失败直接抛出异常（方便调试）
4. **生产环境**：写入失败只记录 `Log::error($exception)`，**不抛出异常**，返回内存中的 ActivityLog 对象（此时对象 `id` 为 null，并未真的写入数据库）
5. 调用方永远不会感知到生产环境的写入失败

实际写入操作在 [ActivityLogService::save()](file:///d:/fz/0508-3/solo-dogfeeding/code/203-panel/app/Services/Activity/ActivityLogService.php#L227-L252)：

```php
protected function save(): ActivityLog
{
    Assert::notNull($this->activity);

    $response = $this->connection->transaction(function () {
        $this->activity->save();  // 写入 activity_logs

        $subjects = Collection::make($this->subjects)
            ->map(fn (Model $subject) => [
                'activity_log_id' => $this->activity->id,
                'subject_id' => $subject->getKey(),
                'subject_type' => $subject->getMorphClass(),
            ])
            ->values()
            ->toArray();

        ActivityLogSubject::insert($subjects);  // 写入 activity_log_subjects

        return $this->activity;
    });

    $this->activity = null;
    $this->subjects = [];

    return $response;
}
```

写入步骤：
1. **数据库事务内**先插入 `activity_logs` 表主记录
2. 再批量插入 `activity_log_subjects` 表关联记录
3. 事务提交，返回已保存的模型
4. 任何一步失败 → 事务回滚 → 抛出异常 → 被 `log()` 外层的 try-catch 捕获

**写入可靠性结论**：
- ✅ `save()` 方法本身是原子的（事务），要么全成功要么全失败
- ⚠️ 但 `log()` 在生产环境**吞掉了异常**，调用方无法感知失败
- ❌ 因此活动日志写入**不保证一定成功**，生产环境下数据库写入失败只会静默记一条日志，用户看不到这条活动
- 但由于写入操作比较简单（纯 INSERT，无复杂业务逻辑），实际失败概率很低

#### 2.2.3 活动日志读取 API 入口

存在**两个**查询入口，分别对应账户维度和服务器维度：

**入口 1：账户活动日志（所有服务器）**
- 路由：`GET /api/client/account/activity`
- 控制器：[ActivityLogController::index()](file:///d:/fz/0508-3/solo-dogfeeding/code/203-panel/app/Http/Controllers/Api/Client/ActivityLogController.php)
- 查询逻辑：基于 actor 过滤，只返回当前用户作为执行者的活动
- 分页参数：支持 `page`、`per_page`

**入口 2：单个服务器的活动日志**
- 路由：`GET /api/client/servers/{server}/activity`
- 控制器：[Servers/ActivityLogController::index()](file:///d:/fz/0508-3/solo-dogfeeding/code/203-panel/app/Http/Controllers/Api/Client/Servers/ActivityLogController.php)
- 查询逻辑：通过 `activity_log_subjects` 关联表过滤指定 server 的活动
- 分页参数：同上

查询后通过 [ActivityLogTransformer.php](file:///d:/fz/0508-3/solo-dogfeeding/code/203-panel/app/Transformers/Api/Client/ActivityLogTransformer.php) 做 fractal 转换，输出标准化 JSON。

#### 2.2.4 活动日志前端展示链路

```
用户访问 /account/activity
    │
    ├── [routes.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/203-panel/resources/scripts/routers/routes.ts#L63-L67)
    │   → 路由 → ActivityLogContainer 组件
    │
    └── [ActivityLogContainer.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/203-panel/resources/scripts/components/dashboard/activity/ActivityLogContainer.tsx)
        │
        └── 调用 useActivityLogs() Hook
            │
            └── [activity.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/203-panel/resources/scripts/api/account/activity.ts)
                │
                └── 发起 HTTP 请求 GET /api/client/account/activity
                    │
                    └── 返回 JSON（ActivityLog 数组）
                        │
                        └── [ActivityLogEntry.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/203-panel/resources/scripts/components/elements/activity/ActivityLogEntry.tsx)
                            → 渲染单条活动日志条目
```

#### 2.2.5 活动日志 vs 通知系统对比（校正版）

| 特性 | 活动日志 (Activity Log) | 通知系统 (Notification) |
|-----|------------------------|-----------------------|
| 存储表 | `activity_logs` + `activity_log_subjects` | `notifications`（未使用） |
| 触发方式 | 显式调用 `Activity::event()->subject()->log()` | 事件驱动或手动 `notify()` |
| 邮件投递 | 不发送邮件 | 发送邮件 |
| 前端入口 | `/account/activity` 页面 | 无前端入口 |
| 数据读取 | 2 个 API 端点 + 完整前端展示 | 从未被读取 |
| 写入可靠性 | 事务内写入，但生产环境吞异常，**不保证成功** | 依赖邮件服务，可能失败 |
| 失败处理 | 生产环境：静默记日志，返回内存对象 | 异步：`failed_jobs` 表；同步：调用方处理 |
| 失败后用户感知 | 无感知，就是页面看不到这条活动 | 无感知（异步）或 500 错误（同步） |
| 渠道配置 | 无渠道概念 | 仅 mail 渠道（硬编码） |

---

## 3. 渠道开关粒度分析

### 3.1 渠道定义方式

Pterodactyl Panel 的通知渠道采用**硬编码 + 全局配置**两层控制：

#### 3.1.1 通知级别：硬编码渠道

每个通知类通过 `via()` 方法硬编码指定投递渠道。**所有 6 个通知类均只返回 `['mail']`**，即仅通过邮件渠道发送。

```php
// 所有通知类的 via() 方法完全一致
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
- ❌ 从未使用 `'database'` 渠道（虽然 `notifications` 表存在）

#### 3.1.2 全局级别：配置开关（仅服务器安装类通知）

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

**其他 4 个普通通知 + 1 个测试邮件**：均无独立的全局开关，业务流程触发即发送。

### 3.2 渠道开关粒度总结

| 粒度级别 | 是否支持 | 实现方式 |
|---------|---------|---------|
| 全局总开关 | 部分支持 | 通过 `.env` 配置控制特定通知类型（仅服务器安装/重装通知） |
| 通知类型级别 | 部分支持 | 仅服务器安装/重装通知有独立开关；其余通知无独立开关 |
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

虽然通知本身没有去重，但某些**非通知类**的 Job 实现了队列去重：

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

活动日志系统在 [ActivityLogService::subject()](file:///d:/fz/0508-3/solo-dogfeeding/code/203-panel/app/Services/Activity/ActivityLogService.php#L77-L83) 中实现了 subject 去重：

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
- 同一活动日志条目避免关联重复的 subject 对象
- 仅在**单次 `log()` 调用内**有效，不跨请求去重
- 影响的是 `activity_log_subjects` 表的关联记录，不影响主表去重

### 4.4 合并去重策略总结

| 层面 | 去重机制 | 适用范围 |
|-----|---------|---------|
| 通知系统 | 无 | - |
| 队列 Job | `ShouldBeUnique` 接口 | 仅部分非通知类 Job（如 RevokeSftpAccessJob） |
| 活动日志 | Subject 列表内去重 | 单次 log 调用内，仅影响关联表 |
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

#### 5.2.1 异步队列的失败存储

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

#### 5.2.2 活动日志的失败记录（生产环境静默）

活动日志写入失败时：
- 非生产环境：抛出异常，业务中断
- 生产环境：`Log::error($exception)` 记录到 `laravel.log`，无其他痕迹，不进入 `failed_jobs`

#### 5.2.3 不同发送方式的失败落点对比

| 发送方式 | 适用通知 | 失败落点 | 记录方式 |
|---------|---------|---------|---------|
| **异步队列** | AccountCreated, AddedToServer, RemovedFromServer, SendPasswordReset | 队列 worker 线程处理 | 1. 按 `retry_after` 90 秒自动重试<br>2. 超过重试次数后写入 `failed_jobs` 表<br>3. 异常记录到 `laravel.log` |
| **同步 sendNow** | ServerInstalled | 调用线程直接抛出异常 | 1. 未被业务代码捕获<br>2. 进入全局异常处理器 Handler<br>3. 记录到 `laravel.log`<br>4. **不进入** `failed_jobs` 表 |
| **同步 + try-catch** | MailTested | 调用线程直接抛出异常 → 调用方捕获 | 1. 被 `MailController::test()` 的 try-catch 捕获<br>2. 返回 HTTP 500，响应体带异常消息<br>3. **不进入** `failed_jobs` 表<br>4. 不额外记录日志（由调用方决定） |
| **活动日志 log()** | 不涉及通知 | 事务回滚 → try-catch 捕获 | 1. 生产环境：`Log::error()` 记日志<br>2. 非生产环境：重新抛出异常<br>3. **不进入** `failed_jobs` 表 |

### 5.3 邮件渠道的降级机制

#### 5.3.1 Failover Mailer（默认未启用）

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
- 备用邮件器：log（将邮件内容写入日志文件）
- 当 smtp 发送失败时，自动降级到 log 渠道

**注意**：默认 mailer 是 `smtp`（见 `config/mail.php` 第 15 行），**只有显式将 `MAIL_MAILER` 设置为 `failover` 时才会触发自动降级**。默认配置下 smtp 失败就是失败，不会自动写日志兜底。

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

| 机制 | 通知系统 | 其他 Job (如 RevokeSftpAccessJob) | 活动日志 |
|-----|---------|-----------------------------------|---------|
| 重试次数 | 默认 1 次（未配置） | 显式配置 3 次 | 无（一次性写入） |
| 退避策略 | 无（立即重试） | 线性退避（N*10 秒） | 不适用 |
| 失败回调 | 无 `failed()` 方法 | 部分 Job 有清理逻辑 | 生产环境吞异常 |
| 渠道降级 | mail failover 配置（默认未启用） | 不适用 | 不适用 |
| 失败存储 | `failed_jobs` 表（仅异步） | `failed_jobs` 表 | `laravel.log`（生产环境） |
| 异常日志 | `laravel.log` | `laravel.log` | `laravel.log`（生产环境） |

---

## 6. 同步 vs 异步投递路径深度分析

### 6.1 两条投递路径概览

```
通知发送
    │
    ├───────────────────────────────────────────────┐
    │                                               │
    ▼                                               ▼
同步发送路径 (sendNow)                     异步发送路径 (ShouldQueue + notify())
    │                                               │
    ├─ 当前线程直接执行                             ├─ 包装为 SendQueuedNotifications Job
    ├─ MailChannel::send() 立即执行                 ├─ Job 推入 Redis 队列
    ├─ 成功：返回响应                               ├─ Queue Worker 后台异步处理
    ├─ 失败：抛出异常 → 调用方/全局处理              ├─ 成功：Job 删除
    │                                               ├─ 失败：重试 → 超过次数 → failed_jobs
    ▼                                               ▼
阻塞请求，立即反馈结果                        不阻塞请求，失败无感知
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
- 注释说明：避免重复触发事件（如果用 `send()`，Laravel 会再包一层 Job 推入队列，导致同一个邮件被发送两次）

#### 6.2.2 完整调用链路

```
1. Wings 守护进程上报服务器安装完成
    ↓
2. [ServerInstallController.php::store()](file:///d:/fz/0508-3/solo-dogfeeding/code/203-panel/app/Http/Controllers/Api/Remote/Servers/ServerInstallController.php#L80-L84)
   → 检查全局开关（send_install_notification / send_reinstall_notification）
   → 开关开启 → 触发 ServerInstalled 事件
    ↓
3. [EventServiceProvider.php](file:///d:/fz/0508-3/solo-dogfeeding/code/203-panel/app/Providers/EventServiceProvider.php#L26)
   → 事件映射到 ServerInstalledNotification 监听器
   → （注意：监听类的 FQCN 就是 ServerInstalled 自己，因为它实现了 ReceivesEvents）
    ↓
4. [ServerInstalled::handle()](file:///d:/fz/0508-3/solo-dogfeeding/code/203-panel/app/Notifications/ServerInstalled.php#L31-L41)
   → 作为事件监听器被同步调用
   → 显式调用 Dispatcher::sendNow()（强制同步，不走队列）
    ↓
5. Illuminate\Notifications\ChannelManager
   → 调用 via() 获取 ['mail']
   → 解析 MailChannel
    ↓
6. Illuminate\Notifications\Channels\MailChannel::send()
   → 调用 toMail() 获取 MailMessage
   → 通过配置的 Mailer 发送
    ↓
7. 成功：执行完毕，继续 HTTP 响应流程
   失败：抛出 TransportException
         → 未被业务代码捕获
         → 进入全局异常 Handler
         → 记录到 laravel.log
         → 返回 HTTP 500 给 Wings
```

#### 6.2.3 同步发送的失败落点

如果同步发送失败：
1. `TransportException` 异常向上抛出
2. 由于是在 HTTP 请求处理中，异常会被 [Handler.php](file:///d:/fz/0508-3/solo-dogfeeding/code/203-panel/app/Exceptions/Handler.php) 捕获
3. 异常堆栈被清理（移除敏感信息）
4. 记录到 `storage/logs/laravel.log`
5. **不会**记录到 `failed_jobs` 表（因为是同步执行，根本没经过队列）
6. API 调用者（Wings）收到 500 错误响应

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
1. 子用户被创建（Subuser 模型 created 事件）
    ↓
2. [SubuserObserver::created()](file:///d:/fz/0508-3/solo-dogfeeding/code/203-panel/app/Observers/SubuserObserver.php#L23-L32)
   → 触发 Subuser\Created 事件（供其他监听器使用）
   → 调用 $user->notify(new AddedToServer(...))
    ↓
3. Illuminate\Notifications\Notifiable::notify()
   → 解析通知类
   → 检测到 ShouldQueue 接口
   → 创建 SendQueuedNotifications Job（Laravel 内部 Job 类）
    ↓
4. Illuminate\Bus\Dispatcher::dispatch()
   → Job 序列化
   → 推入 Redis 队列（queue: standard）
   → HTTP 请求继续执行，立即返回响应给用户
    ↓
======================== 以下为异步执行（Queue Worker 进程）========================
    ↓
5. Queue Worker 进程
   → 从 Redis 取出 Job（BRPOP 阻塞等待）
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
8. 成功：Job 从队列删除（ACK）
   失败：
      → 检查重试次数（tries，默认 1）
      → 未超过：release() 回队列（按 retry_after 90 秒后重试）
      → 已超过：写入 failed_jobs 表 + 记录 laravel.log
```

#### 6.3.2 异步发送的失败落点

如果异步发送失败：
1. 第一次失败：90 秒后自动重试（由 `retry_after` 控制）
2. 第二次失败：由于通知类未配置 `$tries`，取决于 worker 启动参数
   - 如果 worker 启动时 `--tries=1`（默认）：立即写入 `failed_jobs` 表
   - 如果 worker 启动时 `--tries=3`：再重试 2 次后写入 `failed_jobs` 表
3. `failed_jobs` 表记录完整的异常堆栈和 payload
4. 异常同时记录到 `laravel.log`
5. **不会**影响原始 HTTP 请求（早已响应给用户，用户无感知）

### 6.4 同步 vs 异步对比总结

| 维度 | 同步 (sendNow) - ServerInstalled | 异步 (ShouldQueue) - 普通通知 |
|-----|---------------------------------|------------------------------|
| 执行时机 | HTTP 请求处理中立即执行 | 后台 worker 异步执行 |
| 阻塞请求 | 是（直到邮件发送完成或失败） | 否（Job 入队后立即返回） |
| 失败感知 | Wings 收到 HTTP 500，可重试请求 | 请求成功但邮件可能后续失败，无感知 |
| 重试机制 | 无（失败即结束，依赖 Wings 重试请求） | retry_after 90 秒自动重试 + --tries 参数 |
| 失败记录 | laravel.log | laravel.log + failed_jobs 表 |
| 失败后重试方式 | Wings 重新上报安装完成事件，或手动重发整个流程 | `php artisan queue:retry <id>` |
| 性能影响 | 慢（依赖 SMTP 响应速度） | 快（仅 Redis 操作） |
| 可靠性 | 低（SMTP 失败则用户收不到，需 Wings 重上报） | 中（有重试但仍可能最终失败） |

---

## 7. 管理员测试邮件：MailTested 独立分析

MailTested 是一个**完全独立的通知流程**，专门用于管理员测试邮件配置，不参与任何业务事件驱动，不面向普通用户。

### 7.1 触发方式：On-demand Notifications（匿名通知路由）

**触发入口**：[MailController::test()](file:///d:/fz/0508-3/solo-dogfeeding/code/203-panel/app/Http/Controllers/Admin/Settings/MailController.php#L77-L87)

```php
public function test(Request $request): Response
{
    try {
        Notification::route('mail', $request->user()->email)
            ->notify(new MailTested($request->user()));
    } catch (\Exception $exception) {
        return response($exception->getMessage(), 500);
    }

    return response('', 204);
}
```

**关键特点**（与普通用户通知完全不同）：

1. **不通过 `$user->notify()`**：使用 `Notification::route('mail', $email)->notify(...)`，即 Laravel 的 On-demand Notifications（匿名通知路由）
2. **通知对象不是 User 模型**：是一个 `AnonymousNotifiable` 匿名对象，只携带邮箱地址
3. **不带 ShouldQueue**：[MailTested.php](file:///d:/fz/0508-3/solo-dogfeeding/code/203-panel/app/Notifications/MailTested.php) 没有实现 `ShouldQueue` 接口，必然同步执行
4. **调用方自带 try-catch**：任何异常直接被捕获，以异常消息文本作为响应体返回 HTTP 500
5. **面向管理员**：仅后台 Admin → Settings → Mail 页面的 "Send Test Email" 按钮触发

### 7.2 MailTested 通知类本身

[MailTested.php](file:///d:/fz/0508-3/solo-dogfeeding/code/203-panel/app/Notifications/MailTested.php)：

```php
class MailTested extends Notification
{
    public function __construct(private User $user)
    {
    }

    public function via(): array
    {
        return ['mail'];
    }

    public function toMail(): MailMessage
    {
        return (new MailMessage())
            ->subject('Pterodactyl Test Message')
            ->greeting('Hello ' . $this->user->name . '!')
            ->line('This is a test of the Pterodactyl mail system. You\'re good to go!');
    }
}
```

注意：`$user` 只是用来获取用户名显示在邮件正文中，通知本身并不发送给这个 User 模型，而是发送给 `Notification::route()` 指定的邮箱（虽然两者邮箱地址相同，但机制不同）。

### 7.3 完整投递链路

```
管理员点击 "Send Test Email" 按钮
    ↓
POST /admin/settings/mail/test
    ↓
[MailController::test()](file:///d:/fz/0508-3/solo-dogfeeding/code/203-panel/app/Http/Controllers/Admin/Settings/MailController.php#L77-L87)
    │
    ├─ 进入 try 块
    │
    ├─ Notification::route('mail', $email)
    │   → 创建 AnonymousNotifiable 对象
    │   → 设置 mail 渠道路由地址 = 当前管理员邮箱
    │
    ├─ → notify(new MailTested($user))
    │      │
    │      ├─ 解析通知类
    │      ├─ MailTested 不带 ShouldQueue → 同步执行
    │      ├─ via() → ['mail']
    │      ├─ MailChannel::send()
    │      ├─ toMail() → 构建 MailMessage
    │      └─ Mailer::send() → SMTP 发送
    │
    ├─ ✅ 成功：返回 HTTP 204（无响应体）
    │
    └─ ❌ 失败：catch (\Exception $exception)
           → 返回 HTTP 500，响应体 = $exception->getMessage()
           → 前端弹窗显示错误信息
           → 不进入 failed_jobs（未经过队列）
           → 不额外记录 laravel.log（由调用方不主动调用 Log）
```

### 7.4 失败落点和用户体验

MailTested 的失败处理是所有通知中**对用户最友好**的：
- 失败时直接在前端页面弹出错误消息（比如 "Connection could not be established with host smtp.example.com"）
- 管理员可以**立即知道**邮件配置是否正确
- 不需要去查 `failed_jobs` 表或 `laravel.log`
- 同步执行意味着无需等待队列 worker

### 7.5 与普通业务通知的对比

| 对比项 | 管理员测试邮件 MailTested | 普通业务通知（如 AddedToServer） |
|-------|------------------------|-------------------------------|
| 触发方式 | `Notification::route()` 匿名路由 | `$user->notify()` 或事件 |
| 通知对象 | AnonymousNotifiable | User 模型（Notifiable trait） |
| ShouldQueue | ❌ 不带 | ✅ 带 |
| 实际发送方式 | 同步 | 异步队列（ServerInstalled 除外） |
| 异常捕获 | 调用方 try-catch | 全局 Handler 或队列系统 |
| 失败反馈 | 前端立即看到错误消息 | 无感知（异步）或 500（同步） |
| 失败记录方式 | HTTP 500 响应体 | laravel.log + failed_jobs |
| 面向用户 | 管理员 | 普通用户 |
| 触发场景 | 手动点击测试按钮 | 业务事件自动触发 |

---

## 8. 关键服务器告警场景分析

### 8.1 "告警邮件未到但站内信收到"的根本原因

基于代码分析，这个现象的根本原因是：

**活动日志和邮件通知是两条独立的流水线。活动日志走数据库写入（生产环境吞异常但通常成功），邮件通知走 SMTP 发送（可能因多种原因失败），两者之间没有任何依赖或降级关系。**

#### 8.1.1 原因 1：活动日志 vs 通知系统分离

用户所说的"站内信"是活动日志（activity_logs 表），而"告警邮件"是通知系统的 mail 渠道。两者是完全独立的：

```
服务器告警事件触发（假设是服务器安装完成）
    │
    ├─ 写入活动日志（ActivityLogService::log()）
    │   └─ save() 在事务内写入 activity_logs + activity_log_subjects
    │   └─ 生产环境：写入失败只记日志，不抛异常
    │   └─ 写入成功：用户在 Activity 页面看到 ✅
    │
    └─ 触发 ServerInstalled 事件（如开关开启）
        └─ ServerInstalled::handle() 内调用 sendNow()
        └─ MailChannel::send() → SMTP 发送
        └─ 发送成功：用户收到邮件 ✅
        └─ 发送失败：记录 laravel.log，用户收不到邮件 ❌
```

注意：并不是所有业务事件都会同时产生活动日志和通知。实际代码中，活动日志和通知的触发点是独立的——有些事件会同时触发两条流水线，有些事件只会触发其中一条。

#### 8.1.2 原因 2：邮件发送失败的可能原因

邮件发送可能因为以下原因失败：

| 原因 | 影响范围 | 排查方式 |
|-----|---------|---------|
| **SMTP 服务器不可达** | 所有邮件 | `telnet smtp.example.com 587` |
| **SMTP 认证失败** | 所有邮件 | 检查用户名密码/API Key |
| **邮件被拒收/进垃圾箱** | 单封邮件 | 查看收件人垃圾箱、SMTP 返回码 |
| **队列 Worker 未运行** | 异步通知（4 个普通通知） | `ps aux | grep queue:work` |
| **邮件发送超时** | 所有邮件 | 检查 `retry_after`（默认 90 秒） |
| **Worker 异常退出** | 异步通知 | 查看 supervisor/systemd 日志 |
| **Redis 连接问题** | 异步通知 | 检查队列是否能入队/出队 |
| **MAIL_MAILER 不是 failover** | 所有邮件 | 失败不自动降级，邮件直接丢失 |

对于 ServerInstalled（同步发送）：
- 失败直接返回 500 给 Wings，Wings 可能会在后续重试上报
- 但最终是否重发取决于 Wings 的重试逻辑

对于 4 个普通通知（异步）：
- 如果 queue worker 没运行，邮件会一直堆在 Redis 队列里
- 即使 worker 后来恢复，过期太久的通知可能已经失去时效性

#### 8.1.3 原因 3：通知偏好配置不存在（不可能是配置问题）

从代码来看，Pterodactyl Panel 本身：
- ❌ 没有用户级别的通知偏好设置
- ❌ 没有角色级别的通知偏好设置
- ❌ 没有渠道级别的开关配置
- ✅ 所有用户收到的通知类型和渠道完全相同

所以**不可能**是"用户关闭了邮件通知但打开了站内信"的情况。不存在这种配置界面。

### 8.2 邮件投递失败的排查路径

#### 8.2.1 快速排查清单

1. **区分是同步还是异步通知失败**
   - 如果是服务器安装/重装通知：检查 `laravel.log` 有没有 TransportException
   - 如果是其他通知（账户创建、子用户增减、密码重置）：检查 `failed_jobs` 表

2. **检查队列状态**（针对异步通知）
   ```bash
   # 确认 queue worker 是否在运行
   ps aux | grep queue:work
   # 或用 supervisor 管理
   supervisorctl status pterodactyl-worker:*

   # 检查 Redis 队列堆积量
   redis-cli llen queues:standard
   ```

3. **检查失败任务**（针对异步通知）
   ```bash
   # 列出所有失败任务
   php artisan queue:failed

   # 重试指定任务
   php artisan queue:retry <job-id>

   # 重试所有失败任务
   php artisan queue:retry all
   ```

4. **检查邮件日志**（针对所有通知）
   ```bash
   # 搜索邮件相关异常
   grep -E "(TransportException|Swift_|EMAIL|mail)" storage/logs/laravel.log | tail -20
   ```

5. **检查 failed_jobs 表**（针对异步通知）
   ```sql
   SELECT id, exception, failed_at
   FROM failed_jobs
   WHERE payload LIKE '%SendQueuedNotifications%'
   ORDER BY failed_at DESC
   LIMIT 10;
   ```

6. **验证邮件配置是否正确**
   - 在 Admin → Settings → Mail 页面点击 "Send Test Email"
   - 这个会触发 MailTested（同步 + try-catch），前端立即反馈成功或失败
   - 如果测试邮件都发不出去，说明全局邮件配置有问题

#### 8.2.2 失败记录位置速查表

| 通知类型 | 发送方式 | 失败记录位置 |
|---------|---------|-------------|
| AccountCreated | 异步 | `failed_jobs` 表 + `laravel.log` |
| AddedToServer | 异步 | `failed_jobs` 表 + `laravel.log` |
| RemovedFromServer | 异步 | `failed_jobs` 表 + `laravel.log` |
| SendPasswordReset | 异步 | `failed_jobs` 表 + `laravel.log` |
| ServerInstalled | 同步 | 仅 `laravel.log`（不进 failed_jobs） |
| MailTested | 同步+捕获 | HTTP 500 响应体（不写日志，不进 failed_jobs） |

### 8.3 改进建议

如果需要解决"邮件可能丢失"的问题，可以考虑以下改进：

#### 8.3.1 增加数据库通知渠道

在所有通知类的 `via()` 方法中添加 `'database'` 渠道：

```php
public function via(): array
{
    return ['mail', 'database'];
}
```

这样即使邮件发送失败，数据库通知仍然存在（写入 `notifications` 表），再开发前端展示页面即可让用户看到。

#### 8.3.2 实现通知偏好系统

1. 创建 `user_notification_preferences` 表，存储用户对每种通知类型的渠道偏好
2. 修改 `via()` 方法，根据用户偏好动态返回渠道列表
3. 提供前端界面让用户配置自己的通知偏好

#### 8.3.3 为通知类配置重试策略

在所有业务通知类中添加重试配置：

```php
class AccountCreated extends Notification implements ShouldQueue
{
    use Queueable;

    public int $tries = 3;
    public int $backoff = 60; // 每次重试间隔 60 秒

    public function failed(\Throwable $exception)
    {
        Log::error('Notification failed', [
            'notification' => static::class,
            'user_id' => $this->user->id,
            'error' => $exception->getMessage(),
        ]);
    }
}
```

#### 8.3.4 启用 failover 邮件驱动

在 `.env` 中配置：
```env
MAIL_MAILER=failover
```

这样当 SMTP 发送失败时，会自动降级将邮件内容写入 `laravel.log`，至少可以事后追查。

#### 8.3.5 ServerInstalled 改为异步发送

将 ServerInstalled 中的 `sendNow()` 改为异步队列：
- 避免阻塞 Wings 的 HTTP 请求
- 失败自动重试
- 失败可通过 `php artisan queue:retry` 重新发送

#### 8.3.6 监控队列健康状态

1. 配置队列监控，当失败任务过多时告警
2. 定期清理 `failed_jobs` 表（模型自带 `MassPrunable`）
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
       │   └─ 创建 SendQueuedNotifications Job →→→→→→→│ 入队 Redis
       │                                               │
       │  HTTP Response 返回（无阻塞）                 │
       │                                               │
       │                                               │  BRPOP 取出 Job
       │                                               │      ↓
       │                                               │  ChannelManager
       │                                               │   ├─ via() → ['mail']
       │                                               │   └─ MailChannel::send()
       │                                               │      ↓
       │                                               │  Mailer::send()
       │                                               │   ├─ 成功 → Job ACK 删除
       │                                               │   └─ 失败 → --tries 重试
       │                                               │            → 超次数写 failed_jobs
```

### 同步通知（以 ServerInstalled 为例）

```
[HTTP Request Thread - Wings API]
       │
       │  ServerInstallController::store()
       │   ├─ 更新服务器 installed_at
       │   └─ 检查全局开关 → 触发 ServerInstalled 事件
       │           ↓
       │  ServerInstalled::handle()（作为事件监听器同步执行）
       │   └─ Dispatcher::sendNow($user, $this)
       │           ↓
       │  ChannelManager
       │   ├─ via() → ['mail']
       │   └─ MailChannel::send()
       │           ↓
       │  Mailer::send()
       │   ├─ ✅ 成功 → 继续执行 → HTTP 204 返回给 Wings
       │   └─ ❌ 失败 → 抛出 TransportException
       │           ↓
       │  全局 Handler::report()
       │   ├─ generateCleanedExceptionStack() 清敏感信息
       │   └─ 记录到 laravel.log
       │           ↓
       │  HTTP 500 错误返回给 Wings
```

### 管理员测试邮件（MailTested）

```
[HTTP Request Thread - Admin Panel]
       │
       │  MailController::test()
       │   └─ try {
       │        Notification::route('mail', $email)
       │          → notify(new MailTested($user))
       │             │
       │             ├─ AnonymousNotifiable
       │             ├─ 不带 ShouldQueue → 同步执行
       │             ├─ via() → ['mail']
       │             └─ MailChannel::send()
       │                   │
       │                   ├─ ✅ 成功 → return HTTP 204
       │                   └─ ❌ 失败 → 抛出 Exception
       │      } catch (\Exception $e) {
       │          return response($e->getMessage(), 500);
       │      }
       │
       │  前端收到 204 或 500 → 显示成功/失败提示
```

### 活动日志完整链路（写入 + 读取 + 展示）

```
[业务代码]                              [数据库]                          [前端]
     │                                     │                                 │
     │  Activity::event('server:backup')   │                                 │
     │    → subject($server)               │                                 │
     │    → property('name', $name)        │                                 │
     │    → withRequestMetadata()          │                                 │
     │    → log()                          │                                 │
     │          │                            │                                 │
     │          ├─ try {                    │                                 │
     │          │    save()                 │                                 │
     │          │      │                    │                                 │
     │          │      ├─ DB::transaction { │                                 │
     │          │      │    activity_logs   │  INSERT → → → → → → → → → →    │
     │          │      │    activity_log_   │  INSERT subjects → → → → → →   │
     │          │      │  }                 │                                 │
     │          │  }                        │                                 │
     │          │  catch (Throwable $e) {   │                                 │
     │          │    非生产环境：rethrow     │                                 │
     │          │    生产环境：Log::error() │                                 │
     │          │  }                         │                                 │
     │          └─ 返回 ActivityLog 对象     │  （可能未写入数据库）            │
     │                                     │                                 │
     │                                     │  ActivityLog::created 事件       │
     │                                     │    → dispatch ActivityLogged     │
     │                                     │      （仅写入成功时触发）         │
     │                                     │                                 │
     │                                     │                                 │  useActivityLogs() Hook
     │                                     │                                 │    ↓
     │                                     │                                 │  GET /api/client/account/activity
     │                                     │                                 │    ↓
     │                                     │  ActivityLogController           │
     │                                     │    → forActor($user) 筛选       │
     │                                     │    → paginate(10)               │
     │                                     │    → fractal transform          │
     │                                     │          ↓                       │
     │                                     │  JSON 响应 ←←←←←←←←←←←←←←←←←←←   │
     │                                     │                                 │    ↓
     │                                     │                                 │  ActivityLogContainer
     │                                     │                                 │    → 渲染 ActivityLogEntry 列表
     │                                     │                                 │    → 用户在页面上看到（站内信）
```
