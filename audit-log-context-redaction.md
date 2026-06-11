# Pterodactyl Panel 审计日志系统分析报告

## 概述

Pterodactyl Panel 包含两套审计日志系统：

1. **新系统：Activity Log**（活动日志）- 当前主要使用的系统，基于 `activity_logs` 和 `activity_log_subjects` 表
2. **旧系统：Audit Log**（审计日志）- 已标记为废弃（deprecated），将在未来版本中移除

本报告主要分析 **Activity Log** 系统，同时简要提及旧系统。

---

## 一、系统架构与代码链路

### 1.1 核心组件

| 组件 | 文件路径 | 作用 |
|------|----------|------|
| 模型 | [ActivityLog.php](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/app/Models/ActivityLog.php) | 活动日志主模型 |
| 模型 | [ActivityLogSubject.php](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/app/Models/ActivityLogSubject.php) | 活动日志关联对象模型 |
| 服务 | [ActivityLogService.php](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/app/Services/Activity/ActivityLogService.php) | 活动日志创建服务 |
| Facade | [Activity.php](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/app/Facades/Activity.php) | 活动日志门面 |
| 批处理服务 | [ActivityLogBatchService.php](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/app/Services/Activity/ActivityLogBatchService.php) | 批量日志处理 |
| 目标服务 | [ActivityLogTargetableService.php](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/app/Services/Activity/ActivityLogTargetableService.php) | 目标对象管理 |
| 服务提供者 | [ActivityLogServiceProvider.php](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/app/Providers/ActivityLogServiceProvider.php) | 服务注册 |
| 配置 | [activity.php](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/config/activity.php) | 活动日志配置 |
| 客户端API控制器 | [ActivityLogController.php](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/app/Http/Controllers/Api/Client/ActivityLogController.php) | 用户账户活动日志API |
| 服务器API控制器 | [ActivityLogController.php](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/app/Http/Controllers/Api/Client/Servers/ActivityLogController.php) | 服务器活动日志API |
| 远程API控制器 | [ActivityProcessingController.php](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/app/Http/Controllers/Api/Remote/ActivityProcessingController.php) | Wings节点上报活动日志 |
| 数据转换器 | [ActivityLogTransformer.php](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/app/Transformers/Api/Client/ActivityLogTransformer.php) | API响应数据转换与脱敏 |
| 前端组件 | [ActivityLogContainer.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/resources/scripts/components/dashboard/activity/ActivityLogContainer.tsx) | 账户活动日志前端 |
| 前端组件 | [ServerActivityLogContainer.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/resources/scripts/components/server/ServerActivityLogContainer.tsx) | 服务器活动日志前端 |
| 前端条目组件 | [ActivityLogEntry.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/resources/scripts/components/elements/activity/ActivityLogEntry.tsx) | 活动日志条目组件 |

### 1.2 数据库迁移

- 主表迁移：[2022_05_28_135717_create_activity_logs_table.php](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/database/migrations/2022_05_28_135717_create_activity_logs_table.php)
- 关联表迁移：[2022_05_29_140349_create_activity_log_actors_table.php](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/database/migrations/2022_05_29_140349_create_activity_log_actors_table.php)
- API密钥追踪迁移：[2022_06_18_112822_track_api_key_usage_for_activity_events.php](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/database/migrations/2022_06_18_112822_track_api_key_usage_for_activity_events.php)

---

## 二、日志采集与存储

### 2.1 存储字段

#### activity_logs 表

| 字段 | 类型 | 说明 | 是否敏感 |
|------|------|------|----------|
| id | bigint | 自增主键 | 否 |
| batch | uuid(可空) | 批处理UUID，用于关联同一次请求的多个日志 | 否 |
| event | string | 事件类型标识，如 `server:console.command` | 否 |
| ip | string | 操作者IP地址 | **是** |
| description | text(可空) | 事件描述 | 视内容而定 |
| actor_type | string(可空) | 操作者类型（多态关联） | 否 |
| actor_id | bigint(可空) | 操作者ID | 否 |
| api_key_id | bigint(可空) | API密钥ID（如果是通过API操作） | 否 |
| properties | json | 事件属性/元数据 | **视内容而定** |
| timestamp | timestamp | 事件发生时间 | 否 |

#### activity_log_subjects 表

| 字段 | 类型 | 说明 |
|------|------|------|
| id | bigint | 自增主键 |
| activity_log_id | bigint | 关联的活动日志ID |
| subject_id | bigint | 关联对象ID |
| subject_type | string | 关联对象类型（如 `server`, `user`, `backup`） |

### 2.2 记录的属性字段（properties）

根据代码分析，`properties` JSON 字段中可能包含以下敏感或非敏感信息：

#### 可能包含敏感信息的字段

| 属性键 | 说明 | 敏感程度 | 出现场景 |
|--------|------|----------|----------|
| `ip` | IP地址 | 高 | 调用 `withRequestMetadata()` 时记录 |
| `useragent` | 用户代理字符串 | 中 | 调用 `withRequestMetadata()` 时记录 |
| `email` | 邮箱地址 | 中 | 子用户管理相关操作 |
| `command` | 执行的控制台命令 | 中 | 服务器命令执行 |
| `password` / 密码相关 | 密码 | **极高** | 注意：登录失败时可能记录凭证 |
| `old` / `new` | 旧值/新值 | 视内容 | 邮箱、配置等变更操作 |
| `url` | URL地址 | 低 | 文件下载操作 |
| `identifier` | 标识符 | 中 | API密钥标识 |
| `fingerprint` | SSH密钥指纹 | 中 | SSH密钥管理 |

#### 非敏感属性字段

| 属性键 | 说明 | 出现场景 |
|--------|------|----------|
| `name` | 名称 | 备份、调度任务等 |
| `file` | 文件路径 | 文件操作 |
| `directory` | 目录路径 | 文件操作 |
| `files` | 文件列表（数组） | 文件批量操作 |
| `count` / `*_count` | 数量计数 | 批量操作 |
| `locked` | 是否锁定 | 备份锁定状态 |
| `truncate` | 是否截断 | 备份恢复 |
| `using_sftp` | 是否使用SFTP | SFTP操作 |
| `variable` | 变量名 | 启动参数修改 |
| `allocation` | 分配端口 | 网络分配 |
| `notes` | 备注 | 分配备注 |
| `action` | 动作类型 | 调度任务 |
| `from` / `to` | 源/目标 | 重命名操作 |

### 2.3 事件类型

活动日志事件类型定义在 [activity.php](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/resources/lang/en/activity.php) 中，主要分类：

- **auth**：认证相关（登录失败、登录成功、密码重置、2FA等）
- **user**：用户账户相关（邮箱修改、密码修改、API密钥、SSH密钥、2FA）
- **server**：服务器操作（重启、控制台命令、电源操作、备份、数据库、文件、SFTP、分配、调度、设置、启动参数、子用户）

### 2.4 日志记录方式

#### 方式一：Facade 直接调用

通过 `Activity` Facade 进行日志记录：

```php
Activity::event('server:console.command')
    ->property('command', $command)
    ->subject($server)
    ->withRequestMetadata()
    ->log();
```

#### 方式二：事务包装

使用 `transaction` 方法确保操作成功后才记录日志：

```php
$backup = Activity::event('server:backup.start')
    ->transaction(function ($log) use ($action, $server, $request) {
        $backup = $action->handle($server, $request->input('name'));
        $log->subject($backup)->property(['name' => $backup->name]);
        return $backup;
    });
```

#### 方式三：事件监听器

通过事件监听器自动记录，如：
- [AuthenticationListener.php](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/app/Listeners/AuthenticationListener.php) - 认证事件
- [TwoFactorListener.php](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/app/Listeners/TwoFactorListener.php) - 2FA事件

#### 方式四：Wings 节点上报

Wings 节点通过远程 API 上报活动日志：
- [ActivityProcessingController.php](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/app/Http/Controllers/Api/Remote/ActivityProcessingController.php)

### 2.5 日志保留策略

配置项 `activity.prune_days` 控制日志保留天数，默认 **90天**。超过保留期的日志会通过 Laravel 的 `MassPrunable` 特性自动清理。

---

## 三、API 查询接口

### 3.1 可用接口

| 接口 | 路径 | 说明 |
|------|------|------|
| 账户活动日志 | `GET /api/client/account/activity` | 查看当前用户的活动日志 |
| 服务器活动日志 | `GET /api/client/servers/{server}/activity` | 查看指定服务器的活动日志 |
| 远程上报 | `POST /api/remote/activity` | Wings节点上报活动日志（内部使用） |

> **注意**：当前代码中**没有管理员专用**的活动日志API端点。管理员需通过客户端API查看，但能看到更多信息（如IP地址）。

### 3.2 分页功能

- 默认每页：**25条**
- 最大每页：**100条**
- 参数：`per_page`
- 使用 Laravel 标准分页响应

### 3.3 过滤功能

使用 `spatie/laravel-query-builder` 实现：

| 过滤器 | 类型 | 说明 |
|--------|------|------|
| `filter[event]` | 部分匹配 | 按事件类型过滤 |
| `sort` | - | 支持按 `timestamp` 排序 |

### 3.4 包含关系

支持 `include=actor` 参数包含操作者信息。

---

## 四、脱敏与隐藏机制

所有脱敏逻辑位于 [ActivityLogTransformer.php](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/app/Transformers/Api/Client/ActivityLogTransformer.php) 中。

### 4.1 IP地址脱敏

#### 顶层 ip 字段

```php
'ip' => $this->canViewIP($model->actor) ? $model->ip : null,
```

**可见条件**（满足任一即可）：
1. 当前用户是日志的操作者本人
2. 当前用户是管理员（`root_admin = true`）

**代码位置**：[ActivityLogTransformer.php#L114-L117](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/app/Transformers/Api/Client/ActivityLogTransformer.php#L114-L117)

#### properties 中的 ip 字段

```php
if ($key === 'ip' && !optional($model->actor)->is($this->request->user())) {
    return [$key => '[hidden]'];
}
```

**可见条件**：仅日志的操作者本人可见，否则显示为 `[hidden]`

> **注意**：管理员也无法通过 properties 看到他人的 IP，只能通过顶层 `ip` 字段看到。

**代码位置**：[ActivityLogTransformer.php#L58-L60](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/app/Transformers/Api/Client/ActivityLogTransformer.php#L58-L60)

### 4.2 ID 哈希

```php
'id' => sha1($model->id),
```

返回的 `id` 字段使用 SHA1 哈希而非原始自增ID。

**注意**：此非安全措施，仅为前端渲染性能优化提供唯一标识。

**代码位置**：[ActivityLogTransformer.php#L25](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/app/Transformers/Api/Client/ActivityLogTransformer.php#L25)

### 4.3 禁用事件

某些事件被禁用，不会在API响应中返回：

```php
public const DISABLED_EVENTS = ['server:file.upload'];
```

当前仅 `server:file.upload` 事件被禁用。

**代码位置**：[ActivityLog.php#L65](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/app/Models/ActivityLog.php#L65)

### 4.4 管理员活动隐藏（可选配置）

配置项 `activity.hide_admin_activity`（默认 `false`）控制是否对普通用户隐藏管理员的活动日志。

当启用时，服务器活动日志查询会过滤掉：
- 非服务器成员的管理员操作

保留的日志包括：
- 系统操作（无操作者）
- 非管理员用户的操作
- 是服务器成员的管理员操作

**代码位置**：[ActivityLogController.php#L31-L46](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/app/Http/Controllers/Api/Client/Servers/ActivityLogController.php#L31-L46)

---

## 五、角色与可见范围差异

### 5.1 角色定义

| 角色 | 标识字段 | 说明 |
|------|----------|------|
| 超级管理员 | `root_admin = true` | 全局管理员，拥有所有权限 |
| 服务器所有者 | `server.owner_id` | 服务器的拥有者 |
| 子用户 | `subusers` 关联 | 被授予特定服务器权限的用户 |
| 普通用户 | `root_admin = false` | 普通注册用户 |
| 系统 | `actor_id = null` | 系统自动操作 |

### 5.2 账户活动日志可见范围

**接口**：`GET /api/client/account/activity`

- **所有用户**：只能看到**自己作为主体**的活动日志
- 查询基于 `$request->user()->activity()` 关系

**代码位置**：[ActivityLogController.php#L18](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/app/Http/Controllers/Api/Client/ActivityLogController.php#L18)

### 5.3 服务器活动日志可见范围

**接口**：`GET /api/client/servers/{server}/activity`

#### 权限要求

需要 `activity.read` 权限才能访问服务器活动日志。

**代码位置**：[ActivityLogController.php#L24](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/app/Http/Controllers/Api/Client/Servers/ActivityLogController.php#L24)

#### 权限配置

权限定义在 [Permission.php](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/app/Models/Permission.php) 中：

```php
public const ACTION_ACTIVITY_READ = 'activity.read';
```

#### 可见性差异总结

| 信息项 | 服务器所有者 | 子用户(有activity.read) | 普通用户 | 管理员(非服务器成员) |
|--------|-------------|----------------------|----------|---------------------|
| 自己的IP | ✅ 可见 | ✅ 可见 | ✅ 可见 | ✅ 可见 |
| 他人的IP | ❌ null | ❌ null | ❌ null | ✅ 可见（顶层ip） |
| 他人properties.ip | ❌ [hidden] | ❌ [hidden] | ❌ [hidden] | ❌ [hidden] |
| 服务器所有活动 | ✅ 全部可见 | ✅ 全部可见 | ❌ 不能访问 | ✅ 全部可见（通过客户端API） |
| 管理员活动 | ✅ 可见 | ✅ 可见 | - | ✅ 可见 |
| 管理员活动(配置hide_admin_activity=true) | ✅ 可见 | ✅ 可见 | - | ❌ 对非成员隐藏 |

---

## 六、管理界面展示

### 6.1 前端展示组件

- **账户活动日志**：[ActivityLogContainer.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/resources/scripts/components/dashboard/activity/ActivityLogContainer.tsx)
- **服务器活动日志**：[ServerActivityLogContainer.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/resources/scripts/components/server/ServerActivityLogContainer.tsx)
- **日志条目**：[ActivityLogEntry.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/resources/scripts/components/elements/activity/ActivityLogEntry.tsx)
- **元数据按钮**：[ActivityLogMetaButton.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/resources/scripts/components/elements/activity/ActivityLogMetaButton.tsx)

### 6.2 展示内容

每个活动日志条目显示：
1. 操作者头像和用户名（或"System"）
2. 事件类型标签（可点击过滤）
3. 操作方式图标（API密钥 / SFTP）
4. 事件描述（通过翻译模板渲染）
5. IP地址（如果有权限查看）
6. 时间（相对时间 + 悬停显示精确时间）
7. "额外元数据"按钮（如果有未在描述中展示的属性）

### 6.3 元数据查看

当存在未在事件描述模板中使用的属性时，会显示"额外元数据"按钮，点击可查看完整的 `properties` JSON 内容。

判断逻辑位于 [ActivityLogTransformer.php#L91-L108](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/app/Transformers/Api/Client/ActivityLogTransformer.php#L91-L108) 的 `hasAdditionalMetadata` 方法中。

---

## 七、旧 AuditLog 系统

### 7.1 状态

已标记为 **废弃（deprecated）**，将在未来版本中移除。

**代码位置**：[AuditLog.php#L11-L12](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/app/Models/AuditLog.php#L11-L12)

### 7.2 存储字段

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

### 7.3 迁移文件

[2021_01_17_102401_create_audit_logs_table.php](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/database/migrations/2021_01_17_102401_create_audit_logs_table.php)

---

## 八、敏感信息总结与风险评估

### 8.1 明确记录的敏感信息

| 信息类型 | 存储位置 | 脱敏状态 | 风险等级 |
|----------|----------|----------|----------|
| IP地址 | `ip` 字段 + `properties.ip` | 部分脱敏（非本人隐藏） | 中 |
| 用户代理 | `properties.useragent` | 未脱敏 | 低 |
| 邮箱地址 | `properties.email` | 未脱敏 | 中 |
| 控制台命令 | `properties.command` | 未脱敏 | 中（可能包含敏感参数） |
| API密钥标识 | `properties.identifier` | 未脱敏 | 中 |
| SSH密钥指纹 | `properties.fingerprint` | 未脱敏 | 低 |
| 登录凭证 | `properties.*` | **可能记录** | **高** |

> **重要风险**：在 [AuthenticationListener.php](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/app/Listeners/AuthenticationListener.php#L26-L28) 中，登录失败时会将**所有凭证字段**记录到 properties 中，可能包含密码等敏感信息。

### 8.2 脱敏覆盖范围

**已脱敏**：
- 顶层 IP 字段（非本人/非管理员隐藏）
- properties 中的 IP 字段（非本人显示 `[hidden]`）
- 日志 ID（SHA1 哈希，非安全目的）

**未脱敏**：
- 邮箱地址
- 用户代理字符串
- 文件路径
- 控制台命令内容
- 配置参数的旧值/新值
- API 密钥标识符
- SSH 密钥指纹
- 其他业务属性

### 8.3 建议

1. **检查登录失败日志**：确认登录失败时记录的凭证中是否包含密码，如有应清除密码字段
2. **扩展脱敏规则**：对邮箱、用户代理等个人信息增加脱敏选项
3. **管理员专用接口**：建议添加管理员专用的活动日志查询接口，方便审计
4. **导出脱敏**：如需导出审计报告，应增加导出时的脱敏处理
5. **旧系统清理**：确认旧 AuditLog 系统是否还有数据，如有需考虑迁移或清理

---

## 九、关键代码索引

### 模型层
- [ActivityLog.php](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/app/Models/ActivityLog.php) - 活动日志主模型
- [ActivityLogSubject.php](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/app/Models/ActivityLogSubject.php) - 关联对象模型
- [Permission.php](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/app/Models/Permission.php) - 权限定义

### 服务层
- [ActivityLogService.php](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/app/Services/Activity/ActivityLogService.php) - 核心日志服务
- [ActivityLogBatchService.php](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/app/Services/Activity/ActivityLogBatchService.php) - 批处理服务
- [ActivityLogTargetableService.php](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/app/Services/Activity/ActivityLogTargetableService.php) - 目标服务

### API 层
- [客户端账户控制器](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/app/Http/Controllers/Api/Client/ActivityLogController.php) - 账户活动日志API
- [客户端服务器控制器](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/app/Http/Controllers/Api/Client/Servers/ActivityLogController.php) - 服务器活动日志API
- [远程上报控制器](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/app/Http/Controllers/Api/Remote/ActivityProcessingController.php) - Wings节点上报

### 数据转换与脱敏
- [ActivityLogTransformer.php](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/app/Transformers/Api/Client/ActivityLogTransformer.php) - 核心脱敏逻辑

### 监听器
- [AuthenticationListener.php](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/app/Listeners/AuthenticationListener.php) - 认证事件监听
- [TwoFactorListener.php](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/app/Listeners/TwoFactorListener.php) - 2FA事件监听

### 配置
- [activity.php](file:///d:/fz/0508-3/solo-dogfeeding/code/204-panel/config/activity.php) - 活动日志配置
