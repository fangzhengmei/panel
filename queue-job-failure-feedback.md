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

### 1.2 失败任务表结构（完整字段演进）

`failed_jobs` 表经历了 **3 次迁移**，完整字段如下：

| 迁移文件 | 新增字段 | 说明 |
|----------|----------|------|
| [2016_01_23_200421_create_failed_jobs_table.php](file:///d:/fz/0508-3/solo-dogfeeding/code/206-panel/database/migrations/2016_01_23_200421_create_failed_jobs_table.php) | `id`, `connection`, `queue`, `payload`, `failed_at` | 初始建表 |
| [2016_09_04_172028_update_failed_jobs_table.php](file:///d:/fz/0508-3/solo-dogfeeding/code/206-panel/database/migrations/2016_09_04_172028_update_failed_jobs_table.php) | `exception` | 存储异常堆栈字符串（text 类型） |
| [2023_01_24_210051_add_uuid_column_to_failed_jobs_table.php](file:///d:/fz/0508-3/solo-dogfeeding/code/206-panel/database/migrations/2023_01_24_210051_add_uuid_column_to_failed_jobs_table.php) | `uuid` | 唯一标识符（nullable + unique），用于 `queue:retry` 按 UUID 重试，迁移会为历史记录自动生成 UUID |

**最终完整字段表**：

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | int auto_increment | 自增主键 |
| `uuid` | varchar nullable unique | UUID 唯一标识（Laravel 8+ `database-uuids` 驱动要求） |
| `connection` | text | 队列连接名（如 `redis`） |
| `queue` | text | 队列名称（如 `standard`） |
| `payload` | longText | 任务序列化 JSON（包含 `job` 类名、`data` 属性、关联模型 IDs 等） |
| `exception` | text | 完整异常堆栈（Laravel Queue Worker 在任务最终失败时写入） |
| `failed_at` | timestamp | 任务标记为最终失败的时间戳 |

`payload` 字段 JSON 结构示意：
```json
{
  "uuid": "550e8400-e29b-41d4-a716-446655440000",
  "job": "Illuminate\\Queue\\CallQueuedHandler@call",
  "displayName": "Pterodactyl\\Jobs\\Schedule\\RunTaskJob",
  "maxTries": 1,
  "maxExceptions": null,
  "failOnTimeout": false,
  "backoff": null,
  "timeout": null,
  "retryUntil": null,
  "data": {
    "commandName": "Pterodactyl\\Jobs\\Schedule\\RunTaskJob",
    "command": "O:42:\"Pterodactyl\\Jobs\\Schedule\\RunTaskJob\":..."
  }
}
```

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
      │       ├─ 写入 failed_jobs 表（payload + exception 堆栈）
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

所有最终失败的队列任务由 Laravel Queue Worker 自动写入 `failed_jobs` 表：

```
任务最终失败（超过 $tries 或 $maxExceptions）
  ↓
Laravel Queue Worker 执行 FailedJobProvider::log()
  ├─ 写入 connection、queue、payload（完整任务序列化数据）
  ├─ 写入 exception（$e->__toString() 包含完整堆栈追踪）
  ├─ 生成并写入 uuid（database-uuids 驱动）
  └─ 写入 failed_at = 当前时间
```

运维可通过 `php artisan queue:failed` 查看，通过 `php artisan queue:retry {uuid|id}` 重试。

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

## 七、前端状态板展示链路（深度解析）

### 7.1 WebSocket 连接全链路

前端**直接连接 Wings Daemon** 的 WebSocket，不经 Panel 中转：

```
用户进入服务器页面（ServerRouter.tsx）
  ↓
ServerRouter [ServerRouter.tsx:103-105] 挂载三个核心监听器：
  ├─ <InstallListener />    — 安装/恢复事件
  ├─ <TransferListener />   — 服务器迁移事件
  └─ <WebsocketHandler />   — WebSocket 连接管理
      ↓
WebsocketHandler.tsx:32-88 调用 connect(uuid)
  ├─ GET /api/client/servers/{uuid}/websocket
  │   → WebsocketController.php:33-72 返回 { token, socket }
  │     ├─ token: JWT（10分钟过期，携带用户权限）
  │     └─ socket: ws(s)://{node_address}/api/servers/{server_uuid}/ws
  ├─ new Websocket()（基于 Sockette 库）
  │   ├─ 连接 Wings 节点 WebSocket 端点
  │   ├─ send('auth', token) 鉴权
  │   └─ 注册事件监听器
  ├─ ServerContext.socket.instance = socket
  └─ ServerContext.socket.connected = true
```

关键点：[WebsocketController.php](file:///d:/fz/0508-3/solo-dogfeeding/code/206-panel/app/Http/Controllers/Api/Client/Servers/WebsocketController.php) 的第 64 行明确将节点连接地址从 http(s) 替换为 ws(s)，前端直接与 Wings 建立 WebSocket 长连接。

### 7.2 SWR 数据获取与刷新机制

**项目版本**：[package.json](file:///d:/fz/0508-3/solo-dogfeeding/code/206-panel/package.json) 使用 `swr@^0.2.3`（非常早期的版本）。

**前端未配置全局 `SWRConfig`**，所有 SWR Hook 使用库的默认行为：

| 配置项 | SWR 0.2.x 默认值 | getServerBackups 是否覆盖 | 实际行为 |
|--------|-------------------|--------------------------|----------|
| `refreshInterval` | `0`（不主动轮询） | 否 | **不自动定时刷新** |
| `revalidateOnFocus` | `true` | 否 | **窗口/标签页获得焦点时重新验证** |
| `revalidateOnReconnect` | `true` | 否 | **浏览器网络重连时重新验证** |
| `revalidateOnMount` | `true` | 否 | 组件挂载时重新验证 |
| `dedupingInterval` | `2000`（2秒） | 否 | 2秒内相同 key 的请求去重 |
| `errorRetryInterval` | `5000`（5秒） | 否 | 出错后 5 秒重试 |

**前端备份列表 Hook**：[getServerBackups.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/206-panel/resources/scripts/api/swr/getServerBackups.ts)

```typescript
// 只传 key 和 fetcher，不传 options → 完全依赖 SWR 默认配置
return useSWR<BackupResponse>(['server:backups', uuid, page], async () => {
    const { data } = await http.get(`/api/client/servers/${uuid}/backups`, { params: { page } });
    return {
        items: (data.data || []).map(rawDataToServerBackup),
        pagination: getPaginationSet(data.meta.pagination),
        backupCount: data.meta.backup_count,
    };
});
```

**结论：SWR 实际触发刷新的时机只有 3 种**：
1. **组件首次挂载**（进入备份页面）
2. **浏览器窗口重新获得焦点**（用户从其他标签页切回来）
3. **浏览器网络断开后重连**（例如 WiFi 重连）
4. 代码中手动调用 `mutate()`（由 WebSocket 事件触发，见下文）

**没有后台自动轮询**！如果用户停留在备份页面且不切换标签页、网络稳定——SWR 永远不会主动刷新。

### 7.3 前端备份完成事件写入状态列表的完整链路

从 Wings 发出事件到前端 UI 渲染 "Failed" 标签的完整流程：

```
【Wings 端】备份执行完成（成功/失败）
  │
  ├─ 步骤1：Wings 回调 Panel REST API
  │   POST https://panel.example.com/api/remote/backups/{backup_uuid}
  │   Body: { successful: false, ... }
  │   → BackupStatusController.php 更新数据库
  │     backups.is_successful = false
  │     backups.completed_at = NOW()
  │
  └─ 步骤2：Wings 向已连接的 WebSocket 客户端广播
      emit('backup completed', JSON.stringify({
          uuid: backup_uuid,
          is_successful: false,
          checksum_type: null,
          checksum: null,
          file_size: 0
      }))
          ↓
【前端】WebSocket 消息接收
  Websocket.ts:25-31 Sockette onmessage
  ├─ JSON.parse(e.data) → { event: 'backup completed:abc-uuid', args: [...] }
  └─ this.emit('backup completed:abc-uuid', ...args)
      ↓
【前端】事件监听器触发
  useWebsocketEvent.ts:13-21（BackupRow 组件注册）
  ├─ instance.addListener('backup completed:abc-uuid', callback)
  └─ callback(data) 执行 → BackupRow.tsx:24-48
      ↓
【前端】SWR 缓存本地更新（Optimistic Update）
  mutate(updater, false)
  ├─ false = 不重新请求后端 API（避免网络请求）
  ├─ updater 函数在内存中遍历 items 数组
  │   └─ 匹配 uuid → 更新字段
  │       ├─ isSuccessful = parsed.is_successful || true  ⚠️ BUG：失败时也会变成 true！（见下）
  │       ├─ checksum = checksum_type + ':' + checksum
  │       ├─ bytes = file_size
  │       └─ completedAt = new Date()  ← 关键：前端本地设置当前时间
  └─ SWR 缓存变更 → 触发所有使用该 key 的组件重渲染
      ↓
【前端】React 组件重渲染
  BackupRow.tsx:50-101
  ├─ 重新计算判断条件
  │   backup.completedAt !== null && !backup.isSuccessful
  └─ 条件满足 → 渲染红色 "Failed" 标签
```

### 7.4 WebSocket 事件监听注册机制

每个备份行独立注册监听器（按 uuid 精确匹配），代码在 [BackupRow.tsx:24-48](file:///d:/fz/0508-3/solo-dogfeeding/code/206-panel/resources/scripts/components/server/backups/BackupRow.tsx#L24-L48)：

```typescript
// 每个 BackupRow 组件挂载时注册，卸载时自动移除
useWebsocketEvent(
    `backup completed:${backup.uuid}` as SocketEvent,  // 精确到具体备份 UUID
    (data) => { /* mutate 更新缓存 */ }
);
```

`useWebsocketEvent` Hook 实现见 [useWebsocketEvent.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/206-panel/resources/scripts/plugins/useWebsocketEvent.ts)：
- 依赖 `ServerContext.socket.connected` 和 `instance`
- 连接断开时自动移除监听，重连后重新注册

### 7.5 前端代码中的潜在 Bug

[BackupRow.tsx:36](file:///d:/fz/0508-3/solo-dogfeeding/code/206-panel/resources/scripts/components/server/backups/BackupRow.tsx#L36)：

```typescript
isSuccessful: parsed.is_successful || true,
```

这段代码存在逻辑错误：当 `parsed.is_successful = false`（备份失败）时，`false || true` 的结果是 `true`。

**但实际 UI 仍能显示 "Failed"**，原因是：
- `completedAt` 在失败时也被设置为 `new Date()`（非 null）
- 然而 `isSuccessful` 被错误设置为 `true`，按理应该显示成功状态
- 这说明**实际失败展示依赖的是 SWR 兜底重新拉取 API 数据**，而非 WebSocket 的乐观更新

当 WebSocket 的乐观更新数据错误后，用户切换标签页触发 `revalidateOnFocus`，SWR 重新请求 `/api/client/servers/{uuid}/backups`，后端返回真实的 `is_successful=false`，此时 UI 才正确显示 "Failed"。

### 7.6 失败状态的视觉呈现

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

### 7.7 "为什么备份失败半天才显示"的完整原因

结合代码分析，延迟显示是多层因素叠加的结果：

| 阶段 | 耗时来源 | 说明 |
|------|----------|------|
| **Wings 执行备份** | 几秒到几小时 | 取决于备份文件大小、磁盘 IO、网络上传速度。此阶段 Panel 完全不知道进度，前端一直显示 Spinner |
| **Wings → Panel 回调** | 毫秒级 | Wings 完成后立即调用 Panel REST API 更新数据库 |
| **Wings → 前端 WebSocket 广播** | 毫秒级 | Wings 同时向已连接的 WS 客户端推送事件 |
| **前端乐观更新 Bug** | — | WebSocket 到达后 `isSuccessful` 被错误设为 `true`，UI 不会显示 Failed |
| **等待 SWR 触发刷新** | 几秒到几分钟 | 必须等待以下事件之一：<br>1. 用户切换标签页再切回（`revalidateOnFocus`）<br>2. 用户浏览器网络重连（`revalidateOnReconnect`）<br>3. 用户手动刷新页面<br>4. 用户离开备份页面再进入（组件重新挂载） |
| **SWR 拉取真实数据** | 几百毫秒 | 重新请求 API，后端返回 `is_successful=false` |
| **React 重渲染** | 毫秒级 | 状态变更后渲染红色 "Failed" 标签 |

**核心结论**：备份失败不会"立即"显示的根本原因是：
1. 备份任务本身在 Wings 端异步执行，Panel 无进度感知能力
2. **WebSocket 乐观更新代码存在 Bug**，导致即使事件到达也无法正确显示失败
3. SWR 默认无自动轮询，必须依赖用户交互触发重新验证才能获取真实失败状态

### 7.8 SWR 分页缓存隔离机制

**SWR Key 包含页码，每页缓存完全独立**：

[getServerBackups.ts:17-29](file:///d:/fz/0508-3/solo-dogfeeding/code/206-panel/resources/scripts/api/swr/getServerBackups.ts#L17-L29) 的第 21 行：

```typescript
return useSWR<BackupResponse>(['server:backups', uuid, page], async () => { /* ... */ });
```

SWR Key 结构为 `['server:backups', serverUuid, pageNumber]`，意味着：

| 页码 | SWR Key | 缓存是否独立 |
|------|---------|--------------|
| 第 1 页 | `['server:backups', 'abc-uuid', 1]` | ✅ 独立缓存 |
| 第 2 页 | `['server:backups', 'abc-uuid', 2]` | ✅ 独立缓存 |
| 第 3 页 | `['server:backups', 'abc-uuid', 3]` | ✅ 独立缓存 |

**这直接导致以下行为**：

1. **某一页调用 `mutate()` 不会影响其他页的缓存**：第 1 页的 BackupRow 触发 mutate 只更新第 1 页缓存，第 2、3 页数据完全不变
2. **翻页触发全新 API 请求**：从第 1 页切到第 2 页时，SWR 发现 Key 变化（`page` 从 1 变 2），立即发起新的 HTTP 请求拉取第 2 页数据
3. **每页数据互不感知**：第 1 页新创建的备份，在第 2 页翻回来时 SWR 会重新验证（因为组件卸载后重新挂载触发 `revalidateOnMount`）

分页切换流程：
```
用户点击第 2 页按钮
  → BackupContainer.tsx:15 setPage(2)
    → ServerBackupContext.page 更新为 2
      → getServerBackups() 的 useSWR Key 变化
        → SWR 触发新请求 GET /api/client/servers/{uuid}/backups?page=2
          → 返回第 2 页数据，存入独立缓存
            → BackupRow 组件批量卸载（第 1 页）+ 批量挂载（第 2 页）
```

### 7.9 哪些备份行能接收 "backup completed" 事件

**核心结论：只有当前渲染在 DOM 中的 BackupRow 组件才注册了 WebSocket 事件监听器，才能接收到事件。**

#### 7.9.1 事件监听器注册的前提条件

每个 BackupRow 通过 [useWebsocketEvent.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/206-panel/resources/scripts/plugins/useWebsocketEvent.ts) 注册监听器，第 13-22 行：

```typescript
return useEffect(() => {
    const eventListener = (event: SocketEvent) => savedCallback.current(event);
    if (connected && instance) {          // ← 条件1：WebSocket 已连接
        instance.addListener(event, eventListener);  // ← 条件2：instance 不为 null
    }
    return () => {
        instance && instance.removeListener(event, eventListener);
    };
}, [event, connected, instance]);  // ← 依赖项变化会重新注册
```

**监听器注册必须同时满足**：
- ✅ `ServerContext.socket.connected === true`（WebSocket 鉴权成功）
- ✅ `ServerContext.socket.instance !== null`（Websocket 对象已创建）
- ✅ BackupRow 组件已挂载且未被卸载

#### 7.9.2 各场景下能否接收事件

| 场景 | 备份行是否在 DOM 中 | WebSocket 是否连接 | 能否接收事件 | 说明 |
|------|---------------------|-------------------|-------------|------|
| **当前页显示的备份** | ✅ 是 | ✅ 是 | ✅ 能 | 每个可见行都注册了独立监听器 |
| **其他页的备份** | ❌ 否（组件已卸载） | ✅ 是 | ❌ 不能 | 翻页时旧 BackupRow 卸载，监听器被 removeListener 移除 |
| **WebSocket 正在连接时** | ✅ 是 | ❌ 否（connecting） | ❌ 暂不能 | 等连接成功后 useEffect 依赖项变化会自动重新注册 |
| **用户未打开备份页面** | ❌ 否（整个路由未渲染） | ✅ 是 | ❌ 不能 | BackupContainer 未挂载，无任何 BackupRow 存在 |
| **用户在服务器控制台页面** | ❌ 否 | ✅ 是 | ❌ 不能 | 备份路由未激活，组件不存在 |
| **新创建的备份（刚插入列表）** | ✅ 是（mutate 插入后立即挂载） | ✅ 是 | ✅ 能 | 新 BackupRow 挂载时立即注册监听器 |
| **浏览器标签页非活动** | ✅ 是（DOM 仍存在） | ✅ 是（连接未断） | ✅ 能 | 组件未卸载，监听器仍在，但重渲染可能被浏览器节流 |

#### 7.9.3 监听器注册的时序问题

**WebSocket 连接与 BackupRow 挂载存在竞态条件**：

```
时序 A（先连 WS 后进入备份页）：常见场景
  1. WebsocketHandler 连接成功 → connected=true, instance=socket
  2. 用户切换到 Backups 标签 → BackupContainer 挂载
  3. 每个 BackupRow 挂载 → useWebsocketEvent 检测 connected=true → 注册监听器 ✅

时序 B（先进入备份页后连 WS）：首次进入服务器页面时可能发生
  1. 用户直接访问 Backups 页面 → BackupContainer 挂载
  2. BackupRow 挂载 → useWebsocketEvent 检测 connected=false → 不注册 ❌
  3. WebsocketHandler 异步连接成功 → connected 变为 true
  4. useWebsocketEvent 的 useEffect 依赖 [connected] 变化 → 重新执行 → 注册监听器 ✅

时序 C（备份完成时 WS 恰好断开重连）：
  1. 事件到达时 instance=null → 监听器不存在 → 事件被丢弃
  2. 重连成功后监听器重新注册 → 但事件已发送过，Wings 不会重发
  3. 最终只能靠 SWR 焦点刷新获取状态 ⚠️
```

### 7.10 本地客户端 vs 其他客户端：刷新路径差异

#### 7.10.1 创建备份时的不同刷新路径

**场景**：用户 A 在浏览器标签 A 创建备份，用户 B 在另一设备/标签页同时查看同一服务器的备份列表。

**客户端 A（发起创建的标签页）刷新路径**：

```
用户点击 "Create backup" → 表单提交
  ↓
CreateBackupButton.tsx:82-87
  ├─ POST /api/client/servers/{uuid}/backups → 后端返回新建备份对象
  └─ mutate(updater, false)  ← 关键：本地乐观更新，不请求 API
      ├─ updater: data => ({
      │     ...data,
      │     items: data.items.concat(backup),  ← 直接拼接到 items 数组末尾
      │     backupCount: data.backupCount + 1
      │  })
      └─ false = 不重新请求后端
          ↓
SWR 缓存（第 1 页）立即更新 → 触发 BackupRow 重渲染
  ↓
新的 BackupRow 挂载 → useWebsocketEvent 注册 `backup completed:{new_uuid}` 监听器 ✅
  ↓
备份完成时 Wings 广播事件 → 监听器触发 → 再次 mutate 更新状态
```

**客户端 B（其他设备/标签页）刷新路径**：

```
客户端 A 发起创建备份
  ↓
客户端 B 的 SWR 无任何感知（SWR 缓存是浏览器内内存级，不跨标签/设备）
  ↓
备份完成时 Wings 广播 `backup completed:{uuid}` 事件
  ↓
客户端 B 的情况：
  ├─ 情况 1：B 当前在备份页第 1 页，新备份在第 1 页可见
  │   └─ BackupRow 已挂载 → 监听器存在 → 收到事件 → mutate 更新 ✅
  │
  ├─ 情况 2：B 在备份页第 1 页，但新备份因分页不在第 1 页（如已有 20+ 条）
  │   └─ 无对应 BackupRow → 无监听器 → 事件丢弃 ❌
  │      （需要 B 翻到最后一页或刷新页面才能看到）
  │
  ├─ 情况 3：B 不在备份页（在控制台/设置等其他路由）
  │   └─ 整个 BackupContainer 未挂载 → 无任何 BackupRow → 所有事件丢弃 ❌
  │
  └─ 情况 4：B 的浏览器标签页非活动状态
      └─ 即使监听器存在，React 重渲染可能被浏览器挂起
         等用户切回标签页时才会重渲染，同时触发 revalidateOnFocus
```

#### 7.10.2 同一条备份在不同场景显示失败告警的时间点对比

假设同一条备份在 Wings 端执行失败，T0 时刻 Wings 同时触发：
1. REST API 回调 Panel（写入数据库 `is_successful=false`）
2. WebSocket 广播 `backup completed:{uuid}` 事件

不同用户看到 "Failed" 标签的时间点：

| 用户场景 | T0+100ms | T0+5s | T0+30s（用户切回标签页） | T0+2min（用户翻页） | 最终是否看到 |
|----------|----------|-------|------------------------|---------------------|-------------|
| **创建者在第 1 页，备份可见** | ⚠️ Bug 导致不显示（isSuccessful=true） | ⚠️ 仍不显示 | ✅ 显示（revalidateOnFocus） | ✅ 显示 | T0+用户交互时间 |
| **创建者在第 2 页，备份在第 1 页** | ❌ 不显示（监听器不存在） | ❌ 不显示 | ❌ 仍不显示（只刷新第 2 页） | ✅ 显示（翻回第 1 页触发 revalidateOnMount） | T0+用户翻页时间 |
| **其他用户在第 1 页，备份可见 + 无 Bug** | ✅ 立即显示（事件到达 + mutate） | ✅ 已显示 | ✅ 已显示 | ✅ 已显示 | T0+100ms（理想） |
| **其他用户在第 1 页，备份可见 + 有 Bug** | ⚠️ 不显示 | ⚠️ 不显示 | ✅ 显示 | ✅ 显示 | T0+用户交互时间 |
| **其他用户不在备份页** | ❌ 不显示 | ❌ 不显示 | ❌ 不显示 | ❌ 不显示（直到进入备份页） | T0+用户打开备份页面 |
| **其他用户在备份页但备份在其他页** | ❌ 不显示 | ❌ 不显示 | ❌ 不显示（只刷新当前页） | ✅ 显示（翻到对应页） | T0+用户翻页时间 |

#### 7.10.3 事件被"丢弃"后的兜底机制

当 WebSocket 事件因上述任一原因未被有效处理时，备份失败状态只能通过以下途径最终同步到前端：

| 兜底触发方式 | 触发条件 | 刷新范围 |
|-------------|----------|----------|
| `revalidateOnFocus` | 用户从其他标签切回当前标签 | 仅刷新**当前页**的 SWR 缓存 |
| `revalidateOnReconnect` | 浏览器网络断开后重连（WiFi 切换等） | 所有失效的 SWR 缓存 |
| `revalidateOnMount` | 组件首次挂载（用户进入备份页/翻页） | 仅当前页码的缓存 |
| 用户手动 F5 刷新页面 | 显式操作 | 重新加载所有数据 |
| 用户离开备份页再进入 | 路由切换触发组件卸载重挂载 | 重新请求第 1 页数据 |

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
1. 检查 backups 表（备份不走队列，因此 failed_jobs 通常为空）：
   SELECT uuid, name, is_successful, is_locked, completed_at, created_at, bytes
   FROM backups WHERE server_id = ? ORDER BY created_at DESC LIMIT 10;
   → completed_at=null 表示 Wings 尚未回调（可能在执行或回调失败）
   → is_successful=false + completed_at!=null 表示明确失败

2. 检查 activity_logs 表：
   SELECT event, created_at, properties FROM activity_logs 
   WHERE event LIKE 'server:backup%' ORDER BY created_at DESC LIMIT 20;
   → 对比 start 和 complete/fail 事件的时间差

3. 检查 Laravel 日志 storage/logs/laravel-YYYY-MM-DD.log
   搜索 backup、BackupStatusController、DaemonConnectionException

4. 检查 Wings 节点日志（根据备份回调的 X-Request-Id 关联）
   Wings 日志位置通常在 /var/log/pterodactyl/wings.log

5. 验证 Wings → Panel 网络连通性：
   在 Wings 节点上 curl -X POST https://panel.example.com/api/remote/backups/test
   检查是否能到达 Panel 的 /api/remote/* 路由

6. 检查前端是否收到 WebSocket 事件：
   浏览器 F12 → Network → WS → 选中 Wings 的 WebSocket 连接 → Messages 标签
   查看是否收到 backup completed 事件
```

### 10.2 队列任务失败排查步骤

```bash
# 查看失败任务列表（显示 uuid、连接、队列、失败时间、类名）
php artisan queue:failed

# 查看某个失败任务的完整 exception 堆栈
# （直接查数据库，artisan 命令默认不显示完整堆栈）
mysql -e "SELECT id, uuid, exception FROM failed_jobs ORDER BY failed_at DESC LIMIT 1\G"

# 重试指定任务（支持 uuid 或 id）
php artisan queue:retry {uuid}
php artisan queue:retry 5

# 重试所有失败任务
php artisan queue:retry all

# 忽略/删除单个失败任务
php artisan queue:forget {uuid}

# 清空所有失败任务
php artisan queue:flush

# 检查 Worker 进程是否存活
ps aux | grep "queue:work"
supervisorctl status
```

### 10.3 关键配置项

| 配置文件 | 配置项 | 说明 | 默认值 |
|----------|--------|------|--------|
| `queue.php` | `connections.redis.retry_after` | 任务超时秒数（超过则判定为僵死任务，重新入队） | 90 |
| `queue.php` | `failed.driver` | 失败任务存储驱动 | `database-uuids` |
| `backups.php` | `throttles.limit` | 时间窗口内最大备份数 | - |
| `backups.php` | `throttles.period` | 节流时间窗口（秒） | - |
| `pterodactyl.php` | `guzzle.timeout` | Wings API 请求超时 | - |
| `pterodactyl.php` | `guzzle.connect_timeout` | Wings API 连接超时 | - |
| `pterodactyl.php` | `email.send_install_notification` | 安装邮件通知开关 | true |
| `pterodactyl.php` | `email.send_reinstall_notification` | 重装邮件通知开关 | true |

---

## 十一、关键文件索引

| 文件路径 | 职责 |
|----------|------|
| [config/queue.php](file:///d:/fz/0508-3/solo-dogfeeding/code/206-panel/config/queue.php) | 队列连接与失败存储配置 |
| [database/migrations/2016_01_23_200421_create_failed_jobs_table.php](file:///d:/fz/0508-3/solo-dogfeeding/code/206-panel/database/migrations/2016_01_23_200421_create_failed_jobs_table.php) | failed_jobs 初始建表 |
| [database/migrations/2016_09_04_172028_update_failed_jobs_table.php](file:///d:/fz/0508-3/solo-dogfeeding/code/206-panel/database/migrations/2016_09_04_172028_update_failed_jobs_table.php) | failed_jobs 新增 exception 字段 |
| [database/migrations/2023_01_24_210051_add_uuid_column_to_failed_jobs_table.php](file:///d:/fz/0508-3/solo-dogfeeding/code/206-panel/database/migrations/2023_01_24_210051_add_uuid_column_to_failed_jobs_table.php) | failed_jobs 新增 uuid 字段 |
| [app/Jobs/Schedule/RunTaskJob.php](file:///d:/fz/0508-3/solo-dogfeeding/code/206-panel/app/Jobs/Schedule/RunTaskJob.php) | 定时任务执行与失败处理 |
| [app/Jobs/RevokeSftpAccessJob.php](file:///d:/fz/0508-3/solo-dogfeeding/code/206-panel/app/Jobs/RevokeSftpAccessJob.php) | SFTP 撤销任务与重试退避 |
| [app/Services/Backups/InitiateBackupService.php](file:///d:/fz/0508-3/solo-dogfeeding/code/206-panel/app/Services/Backups/InitiateBackupService.php) | 备份发起服务（同步非队列） |
| [app/Http/Controllers/Api/Remote/Backups/BackupStatusController.php](file:///d:/fz/0508-3/solo-dogfeeding/code/206-panel/app/Http/Controllers/Api/Remote/Backups/BackupStatusController.php) | Wings 备份状态回调处理 |
| [app/Http/Controllers/Api/Remote/Servers/ServerInstallController.php](file:///d:/fz/0508-3/solo-dogfeeding/code/206-panel/app/Http/Controllers/Api/Remote/Servers/ServerInstallController.php) | Wings 安装状态回调处理 |
| [app/Http/Controllers/Api/Client/Servers/WebsocketController.php](file:///d:/fz/0508-3/solo-dogfeeding/code/206-panel/app/Http/Controllers/Api/Client/Servers/WebsocketController.php) | WebSocket JWT 签发与节点地址返回 |
| [app/Exceptions/DisplayException.php](file:///d:/fz/0508-3/solo-dogfeeding/code/206-panel/app/Exceptions/DisplayException.php) | 可展示异常基类 |
| [app/Exceptions/Http/Connection/DaemonConnectionException.php](file:///d:/fz/0508-3/solo-dogfeeding/code/206-panel/app/Exceptions/Http/Connection/DaemonConnectionException.php) | Wings 连接异常（可恢复） |
| [app/Exceptions/Handler.php](file:///d:/fz/0508-3/solo-dogfeeding/code/206-panel/app/Exceptions/Handler.php) | 全局异常处理器 |
| [app/Models/Backup.php](file:///d:/fz/0508-3/solo-dogfeeding/code/206-panel/app/Models/Backup.php) | 备份模型与状态字段 |
| [app/Models/Server.php](file:///d:/fz/0508-3/solo-dogfeeding/code/206-panel/app/Models/Server.php) | 服务器模型与状态常量 |
| [app/Notifications/ServerInstalled.php](file:///d:/fz/0508-3/solo-dogfeeding/code/206-panel/app/Notifications/ServerInstalled.php) | 安装完成邮件通知 |
| [resources/scripts/routers/ServerRouter.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/206-panel/resources/scripts/routers/ServerRouter.tsx) | 服务器路由与监听器挂载点 |
| [resources/scripts/components/server/backups/BackupContainer.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/206-panel/resources/scripts/components/server/backups/BackupContainer.tsx) | 备份列表容器（分页 + BackupRow 渲染） |
| [resources/scripts/components/elements/Pagination.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/206-panel/resources/scripts/components/elements/Pagination.tsx) | 通用分页组件 |
| [resources/scripts/api/swr/getServerBackups.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/206-panel/resources/scripts/api/swr/getServerBackups.ts) | 前端备份列表 SWR Hook |
| [resources/scripts/api/server/backups/createServerBackup.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/206-panel/resources/scripts/api/server/backups/createServerBackup.ts) | 创建备份 API 封装 |
| [resources/scripts/components/server/backups/BackupRow.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/206-panel/resources/scripts/components/server/backups/BackupRow.tsx) | 备份行组件（失败状态展示 + WebSocket 事件处理） |
| [resources/scripts/components/server/backups/CreateBackupButton.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/206-panel/resources/scripts/components/server/backups/CreateBackupButton.tsx) | 创建备份按钮（新备份本地插入） |
| [app/Http/Controllers/Api/Client/Servers/BackupController.php](file:///d:/fz/0508-3/solo-dogfeeding/code/206-panel/app/Http/Controllers/Api/Client/Servers/BackupController.php) | 客户端备份 API 控制器（分页列表、创建、删除等） |
| [resources/scripts/components/server/WebsocketHandler.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/206-panel/resources/scripts/components/server/WebsocketHandler.tsx) | WebSocket 连接管理 |
| [resources/scripts/components/server/InstallListener.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/206-panel/resources/scripts/components/server/InstallListener.tsx) | 安装/恢复事件监听 |
| [resources/scripts/plugins/Websocket.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/206-panel/resources/scripts/plugins/Websocket.ts) | WebSocket 客户端实现（Sockette 封装） |
| [resources/scripts/plugins/useWebsocketEvent.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/206-panel/resources/scripts/plugins/useWebsocketEvent.ts) | WebSocket 事件监听 Hook |
| [resources/scripts/components/server/events.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/206-panel/resources/scripts/components/server/events.ts) | WebSocket 事件枚举 |
| [resources/scripts/api/server/getWebsocketToken.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/206-panel/resources/scripts/api/server/getWebsocketToken.ts) | 前端获取 WebSocket Token |
| [resources/scripts/state/server/index.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/206-panel/resources/scripts/state/server/index.ts) | 服务器全局状态（easy-peasy Store） |
| [resources/scripts/state/server/socket.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/206-panel/resources/scripts/state/server/socket.ts) | WebSocket 实例状态存储 |
| [package.json](file:///d:/fz/0508-3/solo-dogfeeding/code/206-panel/package.json) | 前端依赖（SWR 版本：0.2.3） |
