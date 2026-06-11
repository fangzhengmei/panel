# Pterodactyl Panel 队列任务失败反馈流程分析

## 概述

本文档详细分析 Pterodactyl Panel 中后台队列任务从**入队 → Worker 执行 → 异常捕获 → 失败记录 → 面板状态展示 → 用户通知**的完整链路，重点阐述重试与放弃策略、可恢复与不可恢复错误的分类，以及任务最终态对资源状态的影响。

---

## 一、队列基础设施

### 1.1 队列配置

队列配置位于 [queue.php](file:///d:/fz/0508-3/solo-dogfeeding/code/206-panel/config/queue.php)：

- **默认驱动**：Redis（`QUEUE_CONNECTION=redis`）
- **队列名称**：`standard`（通过 `QUEUE_STANDARD` 环境变量配置）
- **重试超时**：`retry_after = 90` 秒（任务执行超过 90 秒视为超时，重新入队）
- **失败存储**：数据库表 `failed_jobs`，使用 UUID 驱动（`database-uuids`）

### 1.2 失败任务表结构

`failed_jobs` 表由 [2016_01_23_200421_create_failed_jobs_table.php](file:///d:/fz/0508-3/solo-dogfeeding/code/206-panel/database/migrations/2016_01_23_200421_create_failed_jobs_table.php) 定义，包含字段：

| 字段 | 说明 |
|------|------|
| `id` | 自增主键 |
| `connection` | 队列连接名（如 redis） |
| `queue` | 队列名称（如 standard） |
| `payload` | 任务序列化数据（JSON longText） |
| `failed_at` | 失败时间戳 |

---

## 二、任务入队流程

### 2.1 定时任务（Schedule）入队

**入口服务**：[ProcessScheduleService.php](file:///d:/fz/0508-3/solo-dogfeeding/code/206-panel/app/Services/Schedules/ProcessScheduleService.php)

```
调度触发 → ProcessScheduleService::handle()
  ├─ 数据库事务
  │   ├─ schedule.is_processing = true
  │   ├─ schedule.next_run_at = 下次执行时间
  │   └─ task.is_queued = true
  ├─ 检查 only_when_online（可选）
  │   └─ 查询 Wings 确认服务器运行状态
  └─ 派发 RunTaskJob
      ├─ 非立即执行：dispatch($job->delay($task->time_offset))
      └─ 立即执行：dispatchNow($job) + 手动 failed() 捕获
```

### 2.2 SFTP 撤销任务入队

任务类 [RevokeSftpAccessJob.php](file:///d:/fz/0508-3/solo-dogfeeding/code/206-panel/app/Jobs/RevokeSftpAccessJob.php) 通过 Laravel 的事件监听器触发（如用户密码变更、服务器删除等场景），实现 `ShouldBeUnique` 接口避免重复入队。

### 2.3 备份任务的特殊性

**注意**：备份任务**不是传统的队列 Job**。备份由 [InitiateBackupService.php](file:///d:/fz/0508-3/solo-dogfeeding/code/206-panel/app/Services/Backups/InitiateBackupService.php) 同步执行：

1. 节流检查（`backups.throttles.limit` / `period`）
2. 备份配额检查，超限则删除最旧的未锁定备份
3. 数据库事务创建 `Backup` 记录（`is_successful=false`, `completed_at=null`）
4. 同步调用 Wings Daemon API：`POST /api/servers/{uuid}/backup`

备份的实际执行在 Wings 端完成，完成后通过**回调接口**通知 Panel。

---

## 三、Worker 执行与异常捕获

### 3.1 核心任务类对比

| 任务类 | 连接 | 队列 | $tries | $maxExceptions | 特征 |
|--------|------|------|--------|----------------|------|
| [RunTaskJob](file:///d:/fz/0508-3/solo-dogfeeding/code/206-panel/app/Jobs/Schedule/RunTaskJob.php) | 默认 | `standard` | 未设置（Laravel 默认 1） | 未设置 | 自定义 `failed()` 方法，链式任务调度 |
| [RevokeSftpAccessJob](file:///d:/fz/0508-3/solo-dogfeeding/code/206-panel/app/Jobs/RevokeSftpAccessJob.php) | 默认 | 默认 | **3** | **1** | `ShouldBeUnique`，手动 `release()` 退避重试 |

### 3.2 RunTaskJob 执行流程

代码位于 [RunTaskJob.php:35-84](file:///d:/fz/0508-3/solo-dogfeeding/code/206-panel/app/Jobs/Schedule/RunTaskJob.php#L35-L84)

```
handle()
  ├─ 前置检查
  │   ├─ schedule 未激活且非手动触发 → markTaskNotQueued + markScheduleComplete → return
  │   └─ server.status 非空（服务器被暂停/重装中）→ 调用 failed() → return
  ├─ 任务执行（try-catch）
  │   ├─ ACTION_POWER: DaemonPowerRepository::send()
  │   ├─ ACTION_COMMAND: DaemonCommandRepository::send()
  │   ├─ ACTION_BACKUP: InitiateBackupService::handle()
  │   └─ 异常处理逻辑
  │       └─ 仅当 task.continue_on_failure=true 且异常为 DaemonConnectionException
  │          → 静默忽略，继续后续任务
  │          → 否则重新抛出异常，由 Laravel Queue 处理
  ├─ 成功路径
  │   ├─ markTaskNotQueued()
  │   └─ queueNextTask() → 查找下一个 sequence_id，dispatch 延迟任务
  └─ 失败路径（异常未被 catch）
      └─ Laravel Queue 调用 failed() 方法
```

### 3.3 RevokeSftpAccessJob 执行流程

代码位于 [RevokeSftpAccessJob.php:41-56](file:///d:/fz/0508-3/solo-dogfeeding/code/206-panel/app/Jobs/RevokeSftpAccessJob.php#L41-L56)

```
handle()
  └─ try
      ├─ DaemonRevocationRepository::deauthorize()
      └─ catch DaemonConnectionException
          └─ release(attempts() * 10)  // 线性退避：10s, 20s, 30s
```

---

## 四、异常分类体系

### 4.1 异常继承链

```
\Exception
  └─ PterodactylException [app/Exceptions/PterodactylException.php]
      └─ DisplayException [app/Exceptions/DisplayException.php]
          ├─ 实现 HttpExceptionInterface
          ├─ 支持 4 种日志级别：debug / info / warning / error
          ├─ render(): API 返回 JSON，Web 请求则 flash 消息并重定向
          └─ DaemonConnectionException [app/Exceptions/Http/Connection/DaemonConnectionException.php]
              ├─ 封装 GuzzleHttp TransferException
              ├─ 提取 Wings 返回的 X-Request-Id
              └─ 根据 HTTP 状态码决定日志级别
```

### 4.2 可恢复错误 vs 不可恢复错误

| 分类 | 异常类型 | 处理策略 | 典型场景 |
|------|----------|----------|----------|
| **可恢复** | `DaemonConnectionException` | 重试/退避 | Wings 节点暂时离线、网络超时、502/504 网关错误 |
| **不可恢复** | `\InvalidArgumentException` | 立即失败 | 无效的 Task action 类型 |
| **不可恢复** | `TooManyBackupsException` | 立即失败 | 服务器备份配额已满 |
| **不可恢复** | `TooManyRequestsHttpException` | 立即失败 | 备份节流触发 |
| **不可恢复** | `ModelNotFoundException` | 立即失败（`DeleteWhenMissingModels`） | 任务关联的 Server/Node 已被删除 |

### 4.3 DaemonConnectionException 细节

代码位于 [DaemonConnectionException.php](file:///d:/fz/0508-3/solo-dogfeeding/code/206-panel/app/Exceptions/Http/Connection/DaemonConnectionException.php)

- **状态码映射**：
  - 无响应 → 504 Gateway Timeout
  - 2XX 但异常抛出 → 502 Bad Gateway
  - 其他 → 使用 Wings 返回的实际状态码
- **日志级别**：
  - 5XX 且非 504 → `LEVEL_ERROR`
  - 其余 → `LEVEL_WARNING`
- **report()**：自动记录日志并附带 `X-Request-Id` 便于 Wings 侧排查

---

## 五、重试与放弃策略

### 5.1 Laravel Queue 标准机制

Laravel Queue Worker 对任务的处理顺序：

```
任务出队 → 执行 handle()
  ├─ 成功 → 删除任务
  └─ 抛出异常
      ├─ 检查 attempts() < $tries
      │   ├─ 是 → release() 延迟后重新入队
      │   └─ 否 → 标记为最终失败
      │       ├─ 调用 Job::failed() 方法
      │       ├─ 写入 failed_jobs 表
      │       └─ 触发 Queue::failing() 事件
```

### 5.2 各任务重试策略

**RevokeSftpAccessJob**：
- `$tries = 3`：最多尝试 3 次
- `$maxExceptions = 1`：遇到 1 次异常后不再重试（但实际代码中手动 `release()` 绕过了此限制）
- **手动退避**：`release($this->attempts() * 10)`，延迟 10s → 20s → 30s
- 适用场景：节点暂时不可达，稍后可能恢复

**RunTaskJob**：
- 未设置 `$tries`，Laravel 默认 `tries=1`（不重试）
- 但有 `continue_on_failure` 任务级配置：
  - `task.continue_on_failure = true` 且异常为 `DaemonConnectionException` → 静默跳过当前任务，继续执行下一个序列任务
  - 其他情况 → 立即终止整个 Schedule，标记完成

### 5.3 任务删除保护

[RevokeSftpAccessJob](file:///d:/fz/0508-3/solo-dogfeeding/code/206-panel/app/Jobs/RevokeSftpAccessJob.php#L18) 使用 `#[DeleteWhenMissingModels]` 属性：
- 当任务反序列化时关联的 `Server` 或 `Node` 模型已被删除 → 直接删除任务，不报错
- 适用于"用户已删除服务器，SFTP 撤销已无意义"的场景

### 5.4 任务唯一性

[RevokeSftpAccessJob](file:///d:/fz/0508-3/solo-dogfeeding/code/206-panel/app/Jobs/RevokeSftpAccessJob.php#L34-L39) 实现 `ShouldBeUnique` 接口：
- `uniqueId()` 格式：`revoke-sftp:{user}:{node/server}:{uuid}`
- Redis 中维持唯一锁，避免同一用户重复撤销任务堆积

---

## 六、失败记录持久化

### 6.1 标准失败任务表

所有最终失败的队列任务由 Laravel 自动写入 `failed_jobs` 表，包含完整 `payload`（任务类、属性、关联模型 IDs 等），运维可通过 `php artisan queue:failed` 查看并重试。

### 6.2 备份失败的特殊记录路径

备份失败不走队列失败表，而是通过 **Wings → Panel 回调** 更新数据库：

**回调入口**：[BackupStatusController.php](file:///d:/fz/0508-3/solo-dogfeeding/code/206-panel/app/Http/Controllers/Api/Remote/Backups/BackupStatusController.php)，路由定义在 [api-remote.php:21-24](file:///d:/fz/0508-3/solo-dogfeeding/code/206-panel/routes/api-remote.php#L21-L24)

```
Wings 备份完成（成功或失败）
  → POST /api/remote/backups/{backup_uuid}
  → BackupStatusController::index()
      ├─ 权限校验：node_id 匹配
      ├─ 幂等校验：已成功的备份不可更新
      ├─ Activity 事务记录
      │   ├─ 事件：server:backup.complete / server:backup.fail
      │   └─ 字段更新
      │       ├─ is_successful = request.successful
      │       ├─ is_locked = 成功时保持原值，失败强制解锁（便于删除）
      │       ├─ checksum = 成功时 checksum_type:checksum，失败 null
      │       ├─ bytes = 成功时 file_size，失败 0
      │       └─ completed_at = 当前时间
      └─ S3 多分片上传处理（如适用）
          ├─ 成功 → CompleteMultipartUpload
          └─ 失败 → AbortMultipartUpload
```

### 6.3 服务器安装失败记录

[ServerInstallController.php:53-88](file:///d:/fz/0508-3/solo-dogfeeding/code/206-panel/app/Http/Controllers/Api/Remote/Servers/ServerInstallController.php#L53-L88)

```
Wings 安装完成回调
  → POST /api/remote/servers/{uuid}/install
  → ServerInstallController::store()
      ├─ successful=false
      │   ├─ 首次安装 → status = install_failed
      │   └─ 重装 → status = reinstall_failed
      ├─ 服务器已暂停 → 保持 suspended 状态
      ├─ 更新 installed_at
      └─ 成功时触发 ServerInstalled 事件 → 邮件通知
```

---

## 七、前端状态板展示链路

### 7.1 数据获取：SWR 轮询 + WebSocket 推送

**前端备份列表 Hook**：[getServerBackups.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/206-panel/resources/scripts/api/swr/getServerBackups.ts)

使用 Vercel SWR（stale-while-revalidate）策略：
- 首次加载 → 请求 `/api/client/servers/{uuid}/backups`
- 后台持续轮询（SWR 默认配置）
- WebSocket 事件到达时 → `mutate()` 局部更新缓存，无需全量刷新

### 7.2 WebSocket 事件机制

**事件定义**：[events.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/206-panel/resources/scripts/components/server/events.ts)

| 事件 | 触发时机 |
|------|----------|
| `BACKUP_COMPLETED` | 备份完成（成功/失败） |
| `BACKUP_RESTORE_COMPLETED` | 备份恢复完成 |
| `STATUS` | 服务器状态变更 |
| `INSTALL_COMPLETED` | 服务器安装完成 |

**WebSocket 客户端**：[Websocket.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/206-panel/resources/scripts/plugins/Websocket.ts)
- 基于 Sockette 库
- 最大重连 20 次
- Token 15 分钟过期，通过 `token expiring` / `token expired` 事件自动续期
- Wings 返回 4400/4409 状态码时停止重连（服务器被暂停）

### 7.3 备份行实时更新

**组件**：[BackupRow.tsx:24-48](file:///d:/fz/0508-3/solo-dogfeeding/code/206-panel/resources/scripts/components/server/backups/BackupRow.tsx#L24-L48)

```tsx
useWebsocketEvent(`backup completed:${backup.uuid}`, (data) => {
    const parsed = JSON.parse(data);
    mutate(
        (data) => ({
            ...data,
            items: data.items.map((b) =>
                b.uuid !== backup.uuid ? b : {
                    ...b,
                    isSuccessful: parsed.is_successful || true,
                    checksum: (parsed.checksum_type || '') + ':' + (parsed.checksum || ''),
                    bytes: parsed.file_size || 0,
                    completedAt: new Date(),
                }
            ),
        }),
        false  // false = 不重新请求 API，直接使用本地更新
    );
});
```

### 7.4 失败状态的视觉呈现

[BackupRow.tsx:66-72](file:///d:/fz/0508-3/solo-dogfeeding/code/206-panel/resources/scripts/components/server/backups/BackupRow.tsx#L66-L72)

```tsx
{backup.completedAt !== null && !backup.isSuccessful && (
    <span css={tw`bg-red-500 py-px px-2 rounded-full text-white text-xs uppercase border border-red-600 mr-2`}>
        Failed
    </span>
)}
```

**判断条件**：`completedAt !== null && isSuccessful === false`
- `completedAt = null` → Spinner 加载中图标
- `completedAt !== null && isSuccessful = true` → 正常归档图标 + 文件大小
- `completedAt !== null && isSuccessful = false` → 红色 "Failed" 标签

### 7.5 "为什么备份失败半天才显示"的原因

1. **备份本身非队列任务**：Panel 发起 Wings API 调用后即返回，备份在 Wings 后台执行，耗时取决于文件大小
2. **没有进度回调**：Wings 执行期间 Panel 不知道进度，UI 一直显示 Spinner
3. **依赖 Wings 回调**：只有当 Wings 完成（成功/失败）后调用 `/api/remote/backups/{uuid}` 回调，Panel 才更新 `completed_at` 和 `is_successful`
4. **WebSocket 延迟**：Wings 触发回调后，还需要通过 WebSocket 将 `backup completed` 事件推送到前端浏览器
5. **SWR 兜底**：如果 WebSocket 未送达，前端只能等下一次 SWR 轮询（默认间隔较长）才会刷新到失败状态

---

## 八、用户通知回写机制

### 8.1 通知类型总览

| 场景 | 通知方式 | 触发点 |
|------|----------|--------|
| 服务器安装/重装完成 | 邮件 | `ServerInstalled` 事件 |
| 账户创建 | 邮件 | `AccountCreated` 通知 |
| 添加为服务器子用户 | 邮件 | `AddedToServer` 通知 |
| 从服务器移除 | 邮件 | `RemovedFromServer` 通知 |
| 密码重置 | 邮件 | `SendPasswordReset` 通知 |
| **备份失败** | **无直接通知** | 仅前端 UI + Activity Log |
| **定时任务失败** | **无直接通知** | 仅 failed_jobs 表 |

### 8.2 服务器安装通知流程

```
Wings 回调安装成功
  → ServerInstallController::store()
      └─ event(new ServerInstalled($server))
          → EventServiceProvider 映射
              [ServerInstalledEvent::class => [ServerInstalledNotification::class]]
                  → ServerInstalledNotification::handle()
                      └─ Dispatcher::sendNow($user, $notification)
                          → toMail() 生成邮件
```

关键代码：
- 事件映射：[EventServiceProvider.php:25-27](file:///d:/fz/0508-3/solo-dogfeeding/code/206-panel/app/Providers/EventServiceProvider.php#L25-L27)
- 通知实现：[ServerInstalled.php](file:///d:/fz/0508-3/solo-dogfeeding/code/206-panel/app/Notifications/ServerInstalled.php)
- 开关配置：`pterodactyl.email.send_install_notification` / `send_reinstall_notification`

### 8.3 Activity 日志（审计追踪）

Panel 对所有关键操作使用 Activity Log 记录，备份相关事件：

| 事件名 | 触发时机 |
|--------|----------|
| `server:backup.start` | 用户发起备份 |
| `server:backup.complete` | Wings 回调备份成功 |
| `server:backup.fail` | Wings 回调备份失败 |
| `server:backup.restore` | 用户发起恢复 |
| `server:backup.restore-complete` | 恢复成功 |
| `server:backup.restore-failed` | 恢复失败 |
| `server:backup.lock` / `unlock` | 锁定/解锁备份 |
| `server:backup.delete` | 删除备份 |
| `server:backup.download` | 下载备份 |

用户可在服务器的 Activity 页面和账户 Activity 页面查看历史记录。

---

## 九、任务最终态对资源状态的影响

### 9.1 备份资源状态机

[Backup.php](file:///d:/fz/0508-3/solo-dogfeeding/code/206-panel/app/Models/Backup.php) 模型关键字段：

| 字段 | 创建时 | 成功 | 失败 |
|------|--------|------|------|
| `is_successful` | `false` | `true` | `false` |
| `completed_at` | `null` | 当前时间 | 当前时间 |
| `is_locked` | 用户指定 | 保持原值 | **强制 false** |
| `bytes` | `0` | 实际文件大小 | `0` |
| `checksum` | `null` | `sha1:xxxx` | `null` |

**失败后的特殊处理**：
- `is_locked` 强制解锁 → 失败的备份可以被用户/系统自动清理
- 不占用服务器备份配额（`getNonFailedBackups()` 统计时排除 `is_successful=false` 且 `completed_at!=null` 的记录）

### 9.2 服务器状态机

[Server.php:122-126](file:///d:/fz/0508-3/solo-dogfeeding/code/206-panel/app/Models/Server.php#L122-L126) 定义的状态常量：

| 状态值 | 含义 | 可否执行操作 |
|--------|------|-------------|
| `null` | 正常运行 | ✅ 所有操作 |
| `installing` | 首次安装中 | ❌ |
| `install_failed` | 首次安装失败 | ❌ |
| `reinstall_failed` | 重装失败 | ❌ |
| `suspended` | 已暂停 | ❌ |
| `restoring_backup` | 恢复备份中 | ❌ |

RunTaskJob 执行前检查：`server.status !== null` → 直接调用 `failed()` 不执行任务。

备份恢复流程对状态的变更：
1. 发起恢复 → `server.status = restoring_backup`
2. Wings 回调 → `server.status = null`（无论恢复成功/失败）

失败时用户可再次尝试恢复或使用重装功能。

### 9.3 定时任务状态

[Schedule.php](file:///d:/fz/0508-3/solo-dogfeeding/code/206-panel/app/Models/Schedule.php) 和 [Task.php](file:///d:/fz/0508-3/solo-dogfeeding/code/206-panel/app/Models/Task.php)：

- `schedule.is_processing`：执行中设为 `true`，成功/失败均设为 `false`
- `task.is_queued`：入队时 `true`，执行完成（成功/跳过/失败）设为 `false`
- `schedule.last_run_at`：本次执行开始时间
- `schedule.next_run_at`：下次 Cron 触发时间

失败不会导致 Schedule 被停用，下次 Cron 时间到达仍会继续触发。

---

## 十、运维排障指南

### 10.1 备份失败排查步骤

```
1. 检查 failed_jobs 表（但备份通常不走队列）
2. 检查 backups 表：
   SELECT uuid, name, is_successful, completed_at, created_at 
   FROM backups WHERE server_id = ? ORDER BY created_at DESC LIMIT 10;
3. 检查 activity_logs 表：
   SELECT * FROM activity_logs WHERE event LIKE 'server:backup%' ORDER BY created_at DESC LIMIT 20;
4. 检查 Laravel 日志 storage/logs/laravel-YYYY-MM-DD.log
5. 检查 Wings 节点日志（根据 X-Request-Id 关联）
6. 验证 Wings → Panel 网络连通性（回调地址）
```

### 10.2 队列任务失败排查步骤

```bash
# 查看失败任务列表
php artisan queue:failed

# 重试指定任务
php artisan queue:retry {job-id}

# 重试所有失败任务
php artisan queue:retry all

# 忽略失败任务
php artisan queue:forget {job-id}

# 清空所有失败任务
php artisan queue:flush
```

### 10.3 关键配置项

| 配置文件 | 配置项 | 说明 | 默认值 |
|----------|--------|------|--------|
| `queue.php` | `connections.redis.retry_after` | 任务超时秒数 | 90 |
| `queue.php` | `failed.driver` | 失败任务存储 | `database-uuids` |
| `backups.php` | `throttles.limit` | 时间窗口内最大备份数 | - |
| `backups.php` | `throttles.period` | 节流时间窗口（秒） | - |
| `pterodactyl.php` | `guzzle.timeout` | Wings API 请求超时 | - |
| `pterodactyl.php` | `email.send_install_notification` | 安装邮件通知开关 | true |

---

## 十一、关键文件索引

| 文件路径 | 职责 |
|----------|------|
| [config/queue.php](file:///d:/fz/0508-3/solo-dogfeeding/code/206-panel/config/queue.php) | 队列连接与失败存储配置 |
| [app/Jobs/Schedule/RunTaskJob.php](file:///d:/fz/0508-3/solo-dogfeeding/code/206-panel/app/Jobs/Schedule/RunTaskJob.php) | 定时任务执行与失败处理 |
| [app/Jobs/RevokeSftpAccessJob.php](file:///d:/fz/0508-3/solo-dogfeeding/code/206-panel/app/Jobs/RevokeSftpAccessJob.php) | SFTP 撤销任务与重试退避 |
| [app/Services/Backups/InitiateBackupService.php](file:///d:/fz/0508-3/solo-dogfeeding/code/206-panel/app/Services/Backups/InitiateBackupService.php) | 备份发起服务（同步非队列） |
| [app/Http/Controllers/Api/Remote/Backups/BackupStatusController.php](file:///d:/fz/0508-3/solo-dogfeeding/code/206-panel/app/Http/Controllers/Api/Remote/Backups/BackupStatusController.php) | Wings 备份状态回调处理 |
| [app/Http/Controllers/Api/Remote/Servers/ServerInstallController.php](file:///d:/fz/0508-3/solo-dogfeeding/code/206-panel/app/Http/Controllers/Api/Remote/Servers/ServerInstallController.php) | Wings 安装状态回调处理 |
| [app/Exceptions/DisplayException.php](file:///d:/fz/0508-3/solo-dogfeeding/code/206-panel/app/Exceptions/DisplayException.php) | 可展示异常基类 |
| [app/Exceptions/Http/Connection/DaemonConnectionException.php](file:///d:/fz/0508-3/solo-dogfeeding/code/206-panel/app/Exceptions/Http/Connection/DaemonConnectionException.php) | Wings 连接异常（可恢复） |
| [app/Exceptions/Handler.php](file:///d:/fz/0508-3/solo-dogfeeding/code/206-panel/app/Exceptions/Handler.php) | 全局异常处理器 |
| [app/Models/Backup.php](file:///d:/fz/0508-3/solo-dogfeeding/code/206-panel/app/Models/Backup.php) | 备份模型与状态字段 |
| [app/Models/Server.php](file:///d:/fz/0508-3/solo-dogfeeding/code/206-panel/app/Models/Server.php) | 服务器模型与状态常量 |
| [app/Notifications/ServerInstalled.php](file:///d:/fz/0508-3/solo-dogfeeding/code/206-panel/app/Notifications/ServerInstalled.php) | 安装完成邮件通知 |
| [resources/scripts/api/swr/getServerBackups.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/206-panel/resources/scripts/api/swr/getServerBackups.ts) | 前端备份列表 SWR Hook |
| [resources/scripts/components/server/backups/BackupRow.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/206-panel/resources/scripts/components/server/backups/BackupRow.tsx) | 备份行组件（失败状态展示） |
| [resources/scripts/components/server/WebsocketHandler.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/206-panel/resources/scripts/components/server/WebsocketHandler.tsx) | WebSocket 连接管理 |
| [resources/scripts/plugins/Websocket.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/206-panel/resources/scripts/plugins/Websocket.ts) | WebSocket 客户端实现 |
| [resources/scripts/components/server/events.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/206-panel/resources/scripts/components/server/events.ts) | WebSocket 事件枚举 |
