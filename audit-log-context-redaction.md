# Pterodactyl Panel 审计日志系统分析报告

## 概述

Pterodactyl Panel 包含两套审计日志系统：

1. **新系统：Activity Log**（活动日志）- 当前主要使用的系统，基于 `activity_logs` 和 `activity_log_subjects` 表
2. **旧系统：Audit Log**（审计日志）- 已标记为废弃（deprecated），将在未来版本中移除

本报告主要分析 **Activity Log** 系统，覆盖日志采集、角色可见范围、筛选参数流转、额外元数据按钮触发条件和属性归一化处理等完整链路。

---

## 一、系统架构与代码链路

### 1.1 核心组件

| 组件 | 文件路径 | 作用 |
|------|----------|------|
| 主模型 | [ActivityLog.php](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/app/Models/ActivityLog.php) | 活动日志主模型，定义字段、关系、禁用事件、清理逻辑 |
| 关联模型 | [ActivityLogSubject.php](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/app/Models/ActivityLogSubject.php) | 活动日志多态关联对象表 |
| 核心服务 | [ActivityLogService.php](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/app/Services/Activity/ActivityLogService.php) | 活动日志创建服务，提供流式API记录日志 |
| Facade | [Activity.php](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/app/Facades/Activity.php) | 活动日志门面，方便全局调用 |
| 批处理服务 | [ActivityLogBatchService.php](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/app/Services/Activity/ActivityLogBatchService.php) | 批量日志处理，同请求共享batch UUID |
| 目标服务 | [ActivityLogTargetableService.php](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/app/Services/Activity/ActivityLogTargetableService.php) | 目标对象管理（actor/subject/apiKey） |
| 服务提供者 | [ActivityLogServiceProvider.php](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/app/Providers/ActivityLogServiceProvider.php) | 服务注册与绑定 |
| 配置文件 | [activity.php](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/config/activity.php) | 活动日志配置（保留天数、隐藏管理员活动） |
| 账户API | [ActivityLogController.php](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/app/Http/Controllers/Api/Client/ActivityLogController.php) | 用户账户活动日志API |
| 服务器API | [ActivityLogController.php](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/app/Http/Controllers/Api/Client/Servers/ActivityLogController.php) | 服务器活动日志API |
| 远程上报API | [ActivityProcessingController.php](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/app/Http/Controllers/Api/Remote/ActivityProcessingController.php) | Wings节点上报活动日志 |
| 数据转换器 | [ActivityLogTransformer.php](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/app/Transformers/Api/Client/ActivityLogTransformer.php) | API响应数据转换、脱敏、属性归一化 |
| 前端容器 | [ActivityLogContainer.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/resources/scripts/components/dashboard/activity/ActivityLogContainer.tsx) | 账户活动日志前端容器 |
| 前端条目 | [ActivityLogEntry.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/resources/scripts/components/elements/activity/ActivityLogEntry.tsx) | 活动日志条目组件 |
| 元数据按钮 | [ActivityLogMetaButton.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/resources/scripts/components/elements/activity/ActivityLogMetaButton.tsx) | 额外元数据弹窗按钮 |
| 前端API | [activity.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/resources/scripts/api/account/activity.ts) | 前端SWR钩子与类型定义 |
| 参数转换 | [http.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/resources/scripts/api/http.ts) | withQueryBuilderParams 参数转换函数 |
| Hash解析 | [useLocationHash.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/resources/scripts/plugins/useLocationHash.ts) | URL hash 解析插件 |

### 1.2 数据库迁移文件

- 主表迁移：[2022_05_28_135717_create_activity_logs_table.php](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/database/migrations/2022_05_28_135717_create_activity_logs_table.php)
- 关联表迁移：[2022_05_29_140349_create_activity_log_actors_table.php](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/database/migrations/2022_05_29_140349_create_activity_log_actors_table.php)
- API密钥追踪迁移：[2022_06_18_112822_track_api_key_usage_for_activity_events.php](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/database/migrations/2022_06_18_112822_track_api_key_usage_for_activity_events.php)

---

## 二、日志采集链路

### 2.1 存储字段详解

#### activity_logs 表（主表）

| 字段 | 类型 | 说明 | 是否敏感 |
|------|------|------|----------|
| id | bigint | 自增主键 | 否 |
| batch | uuid(可空) | 批处理UUID，同一次请求产生的多条日志共享同一个batch | 否 |
| event | string | 事件类型标识，如 `server:console.command`、`auth:login` | 否 |
| ip | string | 操作者IP地址（顶层IP字段） | **是** |
| description | text(可空) | 事件描述文本（可空，通常通过翻译模板动态生成） | 视内容而定 |
| actor_type | string(可空) | 操作者类型（多态关联，通常是 `user`） | 否 |
| actor_id | bigint(可空) | 操作者ID | 否 |
| api_key_id | bigint(可空) | API密钥ID，通过API操作时记录 | 否 |
| properties | json | 事件属性/元数据，JSON格式 | **视内容而定** |
| timestamp | timestamp | 事件发生时间 | 否 |

**代码位置**：模型定义见 [ActivityLog.php#L20-L34](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/app/Models/ActivityLog.php#L20-L34)

#### activity_log_subjects 表（多态关联表）

| 字段 | 类型 | 说明 |
|------|------|------|
| id | bigint | 自增主键 |
| activity_log_id | bigint | 关联的活动日志ID |
| subject_id | bigint | 关联对象ID |
| subject_type | string | 关联对象类型（如 `server`, `user`, `backup`, `subuser`） |

**代码位置**：模型定义见 [ActivityLogSubject.php#L12-L17](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/app/Models/ActivityLogSubject.php#L12-L17)

### 2.2 properties 中的常见字段

`properties` 是 JSON 字段，存储各事件的自定义元数据。根据代码分析，常见属性包括：

#### 敏感属性

| 属性键 | 说明 | 敏感程度 | 出现场景 |
|--------|------|----------|----------|
| `ip` | IP地址 | 高 | 调用 `withRequestMetadata()` 时记录 |
| `useragent` | 用户代理字符串 | 中 | 调用 `withRequestMetadata()` 时记录 |
| `email` | 邮箱地址 | 中 | 子用户管理、账户操作 |
| `command` | 控制台命令 | 中 | 服务器命令执行 |
| `old` / `new` | 旧值/新值 | 视内容 | 邮箱、配置等变更操作 |
| `identifier` | API密钥标识 | 中 | API密钥操作 |
| `fingerprint` | SSH密钥指纹 | 低 | SSH密钥管理 |
| 凭证字段 | 登录凭证 | **极高** | 登录失败时可能记录密码 |

#### 非敏感属性

| 属性键 | 说明 | 出现场景 |
|--------|------|----------|
| `name` | 名称 | 备份、调度任务、子用户等 |
| `file` | 文件路径 | 文件操作 |
| `directory` | 目录路径 | 文件目录操作 |
| `files` | 文件列表（数组） | 批量文件操作 |
| `count` / `*_count` | 数量计数 | 批量操作（归一化后） |
| `locked` | 是否锁定 | 备份锁定状态 |
| `truncate` | 是否截断 | 备份恢复操作 |
| `using_sftp` | 是否使用SFTP | SFTP 文件操作 |
| `variable` | 变量名 | 启动参数修改 |
| `allocation` | 端口分配 | 网络分配操作 |
| `notes` | 备注 | 分配备注 |
| `action` | 动作类型 | 调度任务动作 |
| `from` / `to` | 源/目标 | 重命名操作 |

### 2.3 日志采集流程

#### 采集入口：ActivityLogService

[ActivityLogService.php](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/app/Services/Activity/ActivityLogService.php) 是日志记录的核心服务，通过流式API构建日志。

**初始化时自动填入的字段**（见 `getActivity()` 方法）：

```php
// L202-L207
$this->activity = new ActivityLog([
    'ip' => Request::ip(),                          // 顶层IP，自动获取
    'batch' => $this->batch->uuid(),                // 批处理UUID
    'properties' => Collection::make([]),           // 空属性集合
    'api_key_id' => $this->targetable->apiKeyId(),  // API密钥ID（如有）
]);
```

**自动获取操作者**（L213-L217）：
- 优先使用 `targetable->actor()`（中间件设置）
- 否则使用当前登录用户 `manager->guard()->user()`
- 可通过 `anonymous()` 方法清空操作者

#### 四种记录方式

**方式1：Facade 直接调用**

```php
Activity::event('server:console.command')
    ->property('command', $command)
    ->subject($server)
    ->withRequestMetadata()      // 额外记录 IP 和 useragent 到 properties
    ->log();
```

**方式2：事务包装**

操作成功才记录日志，失败则回滚：

```php
$result = Activity::event('server:backup.start')
    ->transaction(function ($log) use ($action, $server, $request) {
        $backup = $action->handle($server, $request->input('name'));
        $log->subject($backup)->property(['name' => $backup->name]);
        return $backup;
    });
```

**方式3：事件监听器**

通过监听 Laravel 事件自动记录：
- [AuthenticationListener.php](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/app/Listeners/AuthenticationListener.php) - 监听登录事件
- [TwoFactorListener.php](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/app/Listeners/TwoFactorListener.php) - 监听2FA事件

> **风险点**：`AuthenticationListener` 在登录失败时会将**所有凭证字段**记录到 properties，可能包含密码。

**方式4：Wings 节点远程上报**

Wings 守护进程通过远程 API 上报活动日志：
- [ActivityProcessingController.php](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/app/Http/Controllers/Api/Remote/ActivityProcessingController.php)

#### 事件类型分类

事件类型定义在翻译文件中（[activity.php](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/resources/lang/en/activity.php)），主要分为三类：

| 分类 | 前缀 | 示例 |
|------|------|------|
| 认证 | `auth:` | `auth:login`, `auth:fail`, `auth:2fa` |
| 用户 | `user:` | `user:password`, `user:email`, `user:apikey` |
| 服务器 | `server:` | `server:power`, `server:console.command`, `server:file` |

### 2.4 禁用事件

部分事件被标记为禁用，不会在 API 响应中返回（查询层通过 `whereNotIn` 过滤）：

```php
// ActivityLog.php L65
public const DISABLED_EVENTS = ['server:file.upload'];
```

**当前仅 `server:file.upload` 被禁用**。

**代码位置**：
- 常量定义：[ActivityLog.php#L61-L65](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/app/Models/ActivityLog.php#L61-L65)
- 查询过滤：账户接口 [ActivityLogController.php#L22](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/app/Http/Controllers/Api/Client/ActivityLogController.php#L22)、服务器接口 [ActivityLogController.php#L30](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/app/Http/Controllers/Api/Client/Servers/ActivityLogController.php#L30)

### 2.5 日志保留与清理

配置项 `activity.prune_days` 控制日志保留天数，默认 **90天**。

超过保留期的日志通过 Laravel 的 `MassPrunable` 特性自动清理（`prunable()` 方法）。

**代码位置**：[ActivityLog.php#L136-L143](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/app/Models/ActivityLog.php#L136-L143)

---

## 三、角色可见范围

### 3.1 角色定义

| 角色 | 标识字段 | 说明 |
|------|----------|------|
| 超级管理员 | `root_admin = true` | 全局管理员，拥有所有权限 |
| 服务器所有者 | `server.owner_id` | 服务器的拥有者 |
| 子用户 | `subusers` 关联 | 被授予特定服务器权限的用户 |
| 普通用户 | `root_admin = false` | 普通注册用户 |
| 系统 | `actor_id = null` | 系统自动操作（无操作者） |

### 3.2 账户活动日志可见范围

**接口**：`GET /api/client/account/activity`

**控制器**：[ActivityLogController.php](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/app/Http/Controllers/Api/Client/ActivityLogController.php)

#### 查询范围

查询基于 `$request->user()->activity()` 关系，这是一个 **subject 多态关联**：

```php
// User.php L278-L281
public function activity(): MorphToMany
{
    return $this->morphToMany(ActivityLog::class, 'subject', 'activity_log_subjects');
}
```

**关键理解**：
- 查询的是**当前用户作为主体（subject）**的日志，**不是**作为操作者（actor）的日志
- 即：凡是将当前用户设为 subject 的活动日志，都会出现在账户活动列表中

#### 举例说明

| 场景 | actor（操作者） | subject（主体） | 用户A能否看到 | 用户B能否看到 |
|------|----------------|----------------|-------------|-------------|
| 用户A改自己密码 | A | A | ✅ 能 | ❌ 不能 |
| 管理员改用户A邮箱 | 管理员 | A | ✅ 能 | ❌ 不能 |
| 用户A改用户B权限 | A | B | ❌ 不能 | ✅ 能 |
| 用户A创建备份 | A | backup+server | ❌ 不在账户列表（在服务器列表） | ❌ 不能 |

**代码位置**：
- 控制器查询：[ActivityLogController.php#L18](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/app/Http/Controllers/Api/Client/ActivityLogController.php#L18)
- 用户模型关系：[User.php#L278-L281](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/app/Models/User.php#L278-L281)

### 3.3 服务器活动日志可见范围

**接口**：`GET /api/client/servers/{server}/activity`

**控制器**：[ActivityLogController.php](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/app/Http/Controllers/Api/Client/Servers/ActivityLogController.php)

#### 权限要求

必须拥有 `activity.read` 权限才能访问：

```php
// L24
$this->authorize(Permission::ACTION_ACTIVITY_READ, $server);
```

**权限常量定义**：
```php
// Permission.php L66
public const ACTION_ACTIVITY_READ = 'activity.read';
```

**代码位置**：
- 权限检查：[ActivityLogController.php#L24](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/app/Http/Controllers/Api/Client/Servers/ActivityLogController.php#L24)
- 权限定义：[Permission.php#L203-L208](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/app/Models/Permission.php#L203-L208)

#### 查询范围

查询基于 `$server->activity()` 关系，同样是 subject 多态关联：

```php
// Server.php L378-L381
public function activity(): MorphToMany
{
    return $this->morphToMany(ActivityLog::class, 'subject', 'activity_log_subjects');
}
```

即：凡是将该服务器设为 subject 的活动日志，都会出现在服务器活动列表中。

**代码位置**：[Server.php#L378-L381](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/app/Models/Server.php#L378-L381)

### 3.4 IP 可见性差异

IP 脱敏在 [ActivityLogTransformer.php](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/app/Transformers/Api/Client/ActivityLogTransformer.php) 中实现。

#### 顶层 ip 字段

```php
// L29
'ip' => $this->canViewIP($model->actor) ? $model->ip : null,
```

可见条件（满足任一即可）：
1. 操作者是当前用户本人：`optional($actor)->is($this->request->user())`
2. 当前用户是管理员：`$this->request->user()->root_admin`

**代码位置**：[ActivityLogTransformer.php#L114-L117](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/app/Transformers/Api/Client/ActivityLogTransformer.php#L114-L117)

#### properties 中的 ip 字段

```php
// L58-L60
if ($key === 'ip' && !optional($model->actor)->is($this->request->user())) {
    return [$key => '[hidden]'];
}
```

可见条件：**仅操作者本人**可见。

> **重要差异**：管理员能看到顶层 `ip` 字段，但同一日志的 `properties.ip` 仍是 `"[hidden]"`，两者不一致。

**代码位置**：[ActivityLogTransformer.php#L58-L60](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/app/Transformers/Api/Client/ActivityLogTransformer.php#L58-L60)

### 3.5 可见性差异总表

| 信息项 | 服务器所有者 | 子用户（有activity.read） | 非成员管理员 |
|--------|-------------|------------------------|-------------|
| 服务器活动日志接口 | ✅ 可访问 | ✅ 可访问 | ✅ 可访问（通过客户端API） |
| 自己操作的顶层 ip | ✅ 可见 | ✅ 可见 | ✅ 可见 |
| 他人操作的顶层 ip | ❌ null | ❌ null | ✅ 可见（root_admin） |
| 他人的 properties.ip | ❌ "[hidden]" | ❌ "[hidden]" | ❌ "[hidden]"（仅本人可见） |
| 禁用事件 | ❌ 不可见 | ❌ 不可见 | ❌ 不可见 |
| 非成员管理员的活动 | ✅ 可见 | ✅ 可见 | ✅ 可见 |
| 非成员管理员的活动<br>(hide_admin_activity=true) | ❌ **隐藏** | ❌ **隐藏** | 自己的操作也被隐藏 |

---

## 四、管理员活动隐藏机制

### 4.1 配置项

```php
// config/activity.php L7-L11
'hide_admin_activity' => env('APP_ACTIVITY_HIDE_ADMIN', false),
```

默认值：`false`（不隐藏）

**代码位置**：[activity.php#L7-L11](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/config/activity.php#L7-L11)

### 4.2 生效范围

**仅服务器活动日志接口生效**（`GET /api/client/servers/{server}/activity`），账户活动日志接口无此逻辑。

**代码位置**：[ActivityLogController.php#L31-L46](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/app/Http/Controllers/Api/Client/Servers/ActivityLogController.php#L31-L46)

### 4.3 核心实现逻辑

当 `hide_admin_activity = true` 时，查询会执行以下操作：

**步骤1：获取服务器成员列表**
```php
$subusers = $server->subusers()->pluck('user_id')->merge($server->owner_id);
```
收集所有服务器成员的 user_id（含所有者）。

**步骤2：LEFT JOIN users 表**
```php
$builder->select('activity_logs.*')
    ->leftJoin('users', function (JoinClause $join) {
        $join->on('users.id', 'activity_logs.actor_id')
            ->where('activity_logs.actor_type', (new User())->getMorphClass());
    })
```
通过 `actor_id` + `actor_type` 左连接用户表。系统操作（`actor_id = null`）LEFT JOIN 后 `users.id` 为 null。

**步骤3：WHERE 正向保留条件**
```php
->where(function (Builder $builder) use ($subusers) {
    $builder->whereNull('users.id')           // 条件1：系统操作
        ->orWhere('users.root_admin', 0)      // 条件2：非管理员
        ->orWhereIn('users.id', $subusers);   // 条件3：服务器成员
})
```

> **关键理解**：代码通过 **WHERE ... OR ... OR ... 正向保留**三条条件的记录。**不在这三条中的记录会被过滤掉（即隐藏）**。

### 4.4 保留/隐藏矩阵

| 操作者类型 | 是否服务器成员 | hide_admin_activity=false | hide_admin_activity=true | 匹配的条件 |
|-----------|--------------|--------------------------|-------------------------|-----------|
| 系统操作（actor_id=null） | - | ✅ 保留 | ✅ 保留 | 条件1 |
| 非管理员用户 | 否 | ✅ 保留 | ✅ 保留 | 条件2 |
| 非管理员用户 | 是 | ✅ 保留 | ✅ 保留 | 条件2+3 |
| 管理员用户 | 否 | ✅ 保留 | ❌ **隐藏** | 三个条件都不匹配 |
| 管理员用户 | 是 | ✅ 保留 | ✅ 保留 | 条件3 |

### 4.5 特殊注意点

**被隐藏的唯一情况**：只有"**非服务器成员的管理员**"执行的操作会被隐藏。

**关于查询者本人**：
- 该逻辑过滤的是日志的 **actor（操作者）**，不区分发起查询的当前用户是谁
- 因此，如果查询者本人是管理员但不是服务器成员，**他自己的操作也会被隐藏**（因为他的 actor_id 不在 `$subusers` 列表中）
- 这是一个潜在的设计问题：非成员管理员查询服务器日志时，看不到自己的操作记录

---

## 五、筛选参数流转链路

### 5.1 完整链路图

筛选参数从前端界面到后端数据库的完整传递路径：

```
┌─────────────────────────────────────────────────────────────┐
│  前端界面（浏览器）                                           │
│                                                              │
│  URL hash: #ip=1.2.3.4&event=server:power                   │
│      │                                                       │
│      ▼                                                       │
│  useLocationHash() 解析 hash                                 │
│      │                                                       │
│      ▼                                                       │
│  setFilters({ filters: { ip: "1.2.3.4", event: "server:power" }, sorts: { timestamp: -1 }, page: 1 })
│      │                                                       │
│      ▼                                                       │
│  useActivityLogs(filters)  →  SWR cache key                 │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│  HTTP 请求                                                   │
│                                                              │
│  GET /api/client/account/activity                           │
│  ?filter[ip]=1.2.3.4                                         │
│  &filter[event]=server:power                                 │
│  &sort=-timestamp                                            │
│  &page=1                                                     │
│  &include[]=actor                                            │
│  &per_page=25                                                │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│  后端 QueryBuilder（spatie/laravel-query-builder）          │
│                                                              │
│  ✅ filter[event] → AllowedFilter::partial('event') 生效      │
│  ❌ filter[ip]    → 未注册，被静默忽略                        │
│  ✅ sort         → allowedSorts(['timestamp']) 生效          │
│  ✅ page/per_page → Laravel 分页 生效                         │
└─────────────────────────────────────────────────────────────┘
```

### 5.2 前端部分详解

#### 5.2.1 URL hash 解析

通过 [useLocationHash.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/resources/scripts/plugins/useLocationHash.ts) 解析 URL 的 hash 部分：

```typescript
// 解析逻辑：#ip=1.2.3.4&event=server:power
// → { ip: "1.2.3.4", event: "server:power" }
```

**两种触发方式**：
1. **手动修改 URL**：用户手动在地址栏添加 `#ip=xxx&event=xxx`
2. **点击事件标签**：点击日志条目上的事件类型链接，调用 `pathTo({ event: activity.event })` 设置 hash

**代码位置**：[useLocationHash.ts#L7-L31](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/resources/scripts/plugins/useLocationHash.ts#L7-L31)

#### 5.2.2 类型定义

前端类型定义在 [activity.ts#L9](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/resources/scripts/api/account/activity.ts#L9)：

```typescript
export type ActivityLogFilters = QueryBuilderParams<'ip' | 'event', 'timestamp'>;
```

声明支持：
- 筛选键：`ip`、`event`
- 排序键：`timestamp`

#### 5.2.3 参数转换（withQueryBuilderParams）

[http.ts#L137-L160](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/resources/scripts/api/http.ts#L137-L160) 中的 `withQueryBuilderParams` 函数负责将前端格式转换为 Laravel Query Builder 格式：

**filters 转换**：
```
输入: { filters: { ip: "1.2.3.4", event: "server:power" } }
输出: { "filter[ip]": "1.2.3.4", "filter[event]": "server:power" }
```

**sorts 转换**：
```
输入: { sorts: { timestamp: -1 } }
输出: { sort: "-timestamp" }
规则: -1 / "desc" → 加前缀 "-"
      1 / "asc"  → 不加前缀
```

**page 转换**：直接透传。

### 5.3 后端部分详解

两个接口都使用 `spatie/laravel-query-builder` 包处理查询参数。

#### 5.3.1 账户活动日志接口

**注册的筛选器**：
```php
// ActivityLogController.php L20-L21
->allowedFilters([AllowedFilter::partial('event')])
->allowedSorts(['timestamp'])
```

**实际生效的参数**：

| 参数 | 类型 | 是否生效 | 说明 |
|------|------|----------|------|
| `filter[event]` | 部分匹配 | ✅ 生效 | SQL LIKE 匹配 event 字段 |
| `filter[ip]` | - | ❌ 不生效 | 前端声明但后端未注册，被静默忽略 |
| `sort` | 排序 | ✅ 生效 | 仅允许 `timestamp` 排序 |
| `page` | 页码 | ✅ 生效 | Laravel 分页 |
| `per_page` | 每页数量 | ✅ 生效 | 默认25，最大100 |
| `include` | 关联包含 | ✅ 生效 | `include=actor` 包含操作者 |

**代码位置**：[ActivityLogController.php#L18-L24](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/app/Http/Controllers/Api/Client/ActivityLogController.php#L18-L24)

#### 5.3.2 服务器活动日志接口

**注册的筛选器**：
```php
// Servers/ActivityLogController.php L28-L29
->allowedSorts(['timestamp'])
->allowedFilters([AllowedFilter::partial('event')])
```

与账户接口完全一致，仅支持 `event` 筛选和 `timestamp` 排序。

**代码位置**：[ActivityLogController.php#L26-L30](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/app/Http/Controllers/Api/Client/Servers/ActivityLogController.php#L26-L30)

### 5.4 前后端不一致问题

**问题**：前端类型声明支持 `filter[ip]`，但后端两个接口均未注册 IP 筛选器。

**影响**：
- 用户手动在 URL 添加 `#ip=xxx` 后，前端会发送 `filter[ip]=xxx` 参数
- 后端 QueryBuilder 收到未注册的筛选参数后，会**静默忽略**（不报错）
- 界面看起来像在筛选 IP，但实际返回的是全部数据

**这是一个潜在的误导性设计**。

---

## 六、属性归一化处理

活动日志的 `properties` 在返回给前端前，会经过 `ActivityLogTransformer` 的 `properties()` 方法进行归一化处理。

**代码位置**：[ActivityLogTransformer.php#L50-L80](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/app/Transformers/Api/Client/ActivityLogTransformer.php#L50-L80)

### 6.1 处理流程

```
原始 properties（Illuminate\Support\Collection）
    │
    ▼
mapWithKeys 遍历每个键值对 ──────────────────┐
    │                                          │
    ├─ 步骤1：IP 脱敏处理                       │
    │    （仅 ip 键，非本人替换为"[hidden]"）    │
    │                                          │
    ├─ 步骤2：目录路径标准化                     │
    │    （仅 directory 键，格式化为/path/）    │
    │                                          │
    └─ 步骤3：数组值计数                         │
         （数组新增 {$key}_count 键）           │
    │                                          │
    └──────────────────────────────────────────┘
    │
    ▼
步骤4：单个 *_count 重命名为 count
    │
    ▼
转换为 stdClass 对象返回
```

### 6.2 各步骤详细说明

#### 步骤1：IP 脱敏

```php
// L58-L60
if ($key === 'ip' && !optional($model->actor)->is($this->request->user())) {
    return [$key => '[hidden]'];
}
```

- **作用**：对 `properties.ip` 进行脱敏
- **条件**：键名等于 `ip` 且当前用户不是操作者本人
- **处理**：值替换为字符串 `"[hidden]"`
- **注意**：管理员也看不到他人的 `properties.ip`（与顶层 `ip` 字段不一致）

#### 步骤2：目录路径标准化

```php
// L64-L66
if ($key === 'directory') {
    $value = str_replace('//', '/', '/' . trim($value, '/') . '/');
}
```

- **作用**：统一目录路径格式
- **条件**：键名等于 `directory` 且值不是数组
- **处理逻辑**：
  1. 去掉首尾的 `/`
  2. 前后各加一个 `/`
  3. 将所有 `//` 替换为 `/`
- **示例**：
  - `"logs/"` → `"/logs/"`
  - `"/etc//nginx"` → `"/etc/nginx/"`
  - `"path/to/dir"` → `"/path/to/dir/"`

#### 步骤3：数组值计数

```php
// L62-L71
if (!is_array($value)) {
    return [$key => $value];  // 非数组直接返回
}
// 数组：同时返回原数组和长度计数
return [$key => $value, "{$key}_count" => count($value)];
```

- **作用**：为数组属性增加长度计数，方便翻译模板引用
- **条件**：属性值是数组类型
- **处理**：除了返回原数组外，额外新增一个 `{$key}_count` 键，值为数组长度
- **示例**：
  - 输入：`files: ["a.txt", "b.txt", "c.txt"]`
  - 输出：`files: ["a.txt", "b.txt", "c.txt"], files_count: 3`

#### 步骤4：单个计数重命名

```php
// L74-L77
$keys = $properties->keys()->filter(fn ($key) => Str::endsWith($key, '_count'))->values();
if ($keys->containsOneItem()) {
    $properties = $properties->merge(['count' => $properties->get($keys[0])])->except($keys[0]);
}
```

- **作用**：当只有一个 `*_count` 键时，简化为 `count`
- **条件**：properties 中只有一个以 `_count` 结尾的键
- **处理**：将该键重命名为 `count`
- **示例**：
  - 只有 `files_count: 3` → 重命名为 `count: 3`
  - 同时有 `files_count: 3` 和 `folders_count: 1` → 保持不变（不重命名）

### 6.3 完整示例

**输入（原始 properties）**：
```json
{
  "ip": "192.168.1.100",
  "useragent": "Mozilla/5.0",
  "files": ["a.txt", "b.txt"],
  "directory": "logs/"
}
```

**非本人操作时的输出（归一化后）**：
```json
{
  "ip": "[hidden]",
  "useragent": "Mozilla/5.0",
  "files": ["a.txt", "b.txt"],
  "count": 2,
  "directory": "/logs/"
}
```

---

## 七、额外元数据按钮

"额外元数据"按钮用于展示未在事件描述中体现的属性数据。按钮是否显示由 `hasAdditionalMetadata()` 方法控制。

**代码位置**：[ActivityLogTransformer.php#L91-L108](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/app/Transformers/Api/Client/ActivityLogTransformer.php#L91-L108)

### 7.1 前端表现

- **按钮图标**：剪贴板图标（ClipboardListIcon）
- **触发方式**：点击按钮弹出对话框
- **展示内容**：以 JSON 格式（格式化）展示所有 `properties`
- **对应 API 字段**：`has_additional_metadata`（布尔值）

**代码位置**：
- 按钮组件：[ActivityLogMetaButton.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/resources/scripts/components/elements/activity/ActivityLogMetaButton.tsx)
- 条目渲染：[ActivityLogEntry.tsx#L95](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/resources/scripts/components/elements/activity/ActivityLogEntry.tsx#L95)

### 7.2 判断逻辑

#### 核心流程

```
properties 为空？
    │
    ├─ 是 → 返回 false（不显示按钮）
    │
    └─ 否
         │
         ▼
    获取事件对应的翻译模板
    （event 中的 : 替换为 .，拼接 "activity." 前缀）
         │
         ▼
    用正则提取模板中所有 :占位符 key
    （如 ":name" → "name"，":old.email" → "old.email"）
         │
         ▼
    构建排除列表 = 占位符key + ["ip", "useragent", "using_sftp"]
         │
         ▼
    遍历**原始** properties 的所有 key
         │
         ▼
    存在 key 不在排除列表中？
         │
         ├─ 是 → 返回 true（显示按钮）
         │
         └─ 否 → 返回 false（不显示按钮）
```

#### 核心代码

```php
protected function hasAdditionalMetadata(ActivityLog $model): bool
{
    // 前置条件
    if (is_null($model->properties) || $model->properties->isEmpty()) {
        return false;
    }

    // 1. 获取翻译模板字符串
    $str = trans('activity.' . str_replace(':', '.', $model->event));

    // 2. 提取模板中的所有 :占位符 key
    // 正则匹配 :name, :old, :new.email 等
    preg_match_all('/:(?<key>[\w.-]+\w)(?:[^\w:]?|$)/', $str, $matches);

    // 3. 构建排除列表（模板占位符 + 默认排除的3个key）
    $exclude = array_merge($matches['key'], ['ip', 'useragent', 'using_sftp']);

    // 4. 检查是否有未被排除的 key
    foreach ($model->properties->keys() as $key) {
        if (!in_array($key, $exclude, true)) {
            return true;  // 发现未排除的 key，立即返回 true
        }
    }

    return false;
}
```

### 7.3 关键细节

#### 1. 基于原始 properties 判断

判断使用的是 `$model->properties`（**原始数据**），而不是 `$this->properties()` 方法返回的归一化数据。

**影响**：
- 归一化产生的 `files_count`、`count` 等键不会影响判断
- 即使 `properties.ip` 被脱敏为 `"[hidden]"`，`ip` 这个 key 仍然存在，因为它在默认排除列表中，不会触发按钮

#### 2. 默认排除三个 key

`ip`、`useragent`、`using_sftp` 这三个 key 被固定加入排除列表。

**原因**：
- `ip` 和 `useragent` 是请求元数据，通常不作为业务属性展示
- `using_sftp` 是操作方式标记，通过图标展示（SFTP图标），不需要额外展示

#### 3. 翻译模板占位符提取

使用正则 `:(?<key>[\w.-]+\w)(?:[^\w:]?|$)/` 从翻译模板中提取占位符。

**匹配规则**：
- 以 `:` 开头
- key 中可包含字母、数字、下划线、点、横杠
- key 以字母或数字结尾
- 后接非单词字符或字符串结尾

**示例**：
- `"Deleted backup :name."` → 提取 `["name"]`
- `"Changed email from :old to :new."` → 提取 `["old", "new"]`
- `"Updated :variable to :value."` → 提取 `["variable", "value"]`

### 7.4 触发条件总结

按钮会显示，当且仅当：
1. `properties` 不为空
2. 存在至少一个属性 key，同时满足：
   - 该 key **不是**翻译模板中的占位符
   - 该 key **不是** `ip`、`useragent`、`using_sftp` 三者之一

### 7.5 示例

**场景1：备份删除事件**
- 事件：`server:backup.delete`
- 翻译模板：`"Deleted backup :name."`
- 占位符：`["name"]`
- properties：`{ "name": "backup-2024", "ip": "1.2.3.4", "useragent": "..." }`
- 排除列表：`["name", "ip", "useragent"]`
- 所有 key 都在排除列表中 → **不显示按钮**

**场景2：文件压缩事件**
- 事件：`server:file.compress`
- 翻译模板：`"Compressed :files files."`
- 占位符：`["files"]`
- properties：`{ "files": [...], "files_count": 5, "directory": "/backups/", "ip": "1.2.3.4" }`
- 排除列表：`["files", "ip", "useragent", "using_sftp"]`
- `files_count` 和 `directory` 不在排除列表中 → **显示按钮**

> 注意：`files_count` 是归一化产生的键，但判断基于原始 properties。如果原始 properties 中没有 `files_count`（确实没有，因为它是步骤3生成的），则它不会触发按钮。但 `directory` 是原始 key，会触发按钮。

---

## 八、配置项说明

配置文件：[activity.php](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/config/activity.php)

| 配置项 | 环境变量 | 默认值 | 说明 |
|--------|----------|--------|------|
| `prune_days` | `APP_ACTIVITY_PRUNE_DAYS` | 90 | 活动日志保留天数，超过自动清理 |
| `hide_admin_activity` | `APP_ACTIVITY_HIDE_ADMIN` | false | 是否对服务器成员隐藏非成员管理员的活动 |

---

## 九、旧 AuditLog 系统

### 9.1 状态

已标记为 **废弃（deprecated）**，将在未来版本中移除。

**代码位置**：[AuditLog.php#L11-L12](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/app/Models/AuditLog.php#L11-L12)

### 9.2 存储字段

| 字段 | 类型 | 说明 |
|------|------|------|
| id | bigint | 自增主键 |
| uuid | char(36) | UUID |
| is_system | boolean | 是否系统操作 |
| user_id | unsignedInteger(可空) | 用户ID |
| server_id | unsignedInteger(可空) | 服务器ID |
| action | string | 操作类型 |
| subaction | string(可空) | 子操作类型 |
| device | json | 设备信息（ip_address, user_agent） |
| metadata | json | 元数据 |
| created_at | timestamp | 创建时间 |

### 9.3 迁移文件

[2021_01_17_102401_create_audit_logs_table.php](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/database/migrations/2021_01_17_102401_create_audit_logs_table.php)

---

## 十、风险与问题总结

### 10.1 安全风险

1. **登录失败可能记录密码**：`AuthenticationListener` 将所有凭证字段写入 properties，可能包含明文密码
2. **IP 脱敏不一致**：管理员能看到顶层 `ip`，但 `properties.ip` 仍是 `"[hidden]"`
3. **个人信息未脱敏**：邮箱地址、用户代理、文件路径等均未脱敏

### 10.2 设计问题

1. **前后端筛选不一致**：前端声明支持 IP 筛选，后端未实现，参数被静默忽略
2. **hide_admin_activity 副作用**：非成员管理员查询自己也看不到自己的操作
3. **缺少管理员专用接口**：管理员只能通过客户端API查看，没有全平台审计接口
4. **额外元数据判断基于原始数据**：不考虑属性是否被脱敏，可能出现"有按钮但内容是[hidden]"的情况

### 10.3 建议

1. **检查登录失败日志**：清除 properties 中的密码字段
2. **扩展脱敏规则**：对邮箱、用户代理等个人信息增加脱敏选项
3. **修复筛选不一致**：要么实现 IP 筛选，要么从前端类型中移除
4. **统一 IP 脱敏**：管理员查看时 properties.ip 也应显示实际值
5. **hide_admin_activity 优化**：查询者本人的操作始终可见
6. **管理员专用审计接口**：提供全平台的活动日志查询接口

---

## 十一、关键代码索引

### 模型层
- [ActivityLog.php](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/app/Models/ActivityLog.php) - 活动日志主模型（字段、关系、DISABLED_EVENTS、MassPrunable）
- [ActivityLogSubject.php](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/app/Models/ActivityLogSubject.php) - 多态关联表模型
- [Permission.php](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/app/Models/Permission.php) - 权限定义（ACTION_ACTIVITY_READ）

### 服务层
- [ActivityLogService.php](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/app/Services/Activity/ActivityLogService.php) - 核心日志服务（event/property/subject/log/transaction/withRequestMetadata）
- [ActivityLogBatchService.php](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/app/Services/Activity/ActivityLogBatchService.php) - 批处理服务
- [ActivityLogTargetableService.php](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/app/Services/Activity/ActivityLogTargetableService.php) - 目标对象服务

### API 层
- [账户活动日志控制器](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/app/Http/Controllers/Api/Client/ActivityLogController.php) - 账户活动日志API
- [服务器活动日志控制器](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/app/Http/Controllers/Api/Client/Servers/ActivityLogController.php) - 服务器活动日志API（含 hide_admin_activity 逻辑）
- [远程上报控制器](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/app/Http/Controllers/Api/Remote/ActivityProcessingController.php) - Wings 节点上报

### 数据转换与脱敏
- [ActivityLogTransformer.php](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/app/Transformers/Api/Client/ActivityLogTransformer.php) - 转换器（IP脱敏、属性归一化四步、额外元数据判断）

### 监听器
- [AuthenticationListener.php](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/app/Listeners/AuthenticationListener.php) - 认证事件监听
- [TwoFactorListener.php](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/app/Listeners/TwoFactorListener.php) - 2FA 事件监听

### 前端
- [ActivityLogContainer.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/resources/scripts/components/dashboard/activity/ActivityLogContainer.tsx) - 账户活动日志容器
- [ActivityLogEntry.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/resources/scripts/components/elements/activity/ActivityLogEntry.tsx) - 日志条目组件
- [ActivityLogMetaButton.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/resources/scripts/components/elements/activity/ActivityLogMetaButton.tsx) - 元数据弹窗按钮
- [activity.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/resources/scripts/api/account/activity.ts) - 前端API与类型定义
- [http.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/resources/scripts/api/http.ts) - withQueryBuilderParams 参数转换
- [useLocationHash.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/resources/scripts/plugins/useLocationHash.ts) - URL hash 解析插件

### 配置
- [activity.php](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/config/activity.php) - 活动日志配置
