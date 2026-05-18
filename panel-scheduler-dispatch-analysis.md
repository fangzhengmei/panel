# Panel 排程器任务派发流程分析

## 1. 概述

本分析文档详细阐述了 Pterodactyl Panel 中排程器（Scheduler）将电源、命令、备份等任务派发给守护进程（Wings）执行的完整流程。整个系统横跨多层架构，包括：

- **任务存储层**：Schedule、Task、TaskLog 模型
- **调度触发层**：Laravel 调度器 + ProcessRunnableCommand
- **任务执行层**：ProcessScheduleService + RunTaskJob
- **通信通道层**：DaemonRepository 系列 HTTP 客户端
- **失败重试层**：Laravel Queue 机制 + 业务逻辑容错

---

## 2. 任务存储结构

### 2.1 Schedule 模型（排程）

**文件**：`app/Models/Schedule.php`

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | int | 主键 |
| `server_id` | int | 关联服务器 ID |
| `name` | string | 排程名称 |
| `cron_minute` | string | Cron 分钟字段（默认：`*`） |
| `cron_hour` | string | Cron 小时字段（默认：`*`） |
| `cron_day_of_month` | string | Cron 日期字段（默认：`*`） |
| `cron_month` | string | Cron 月份字段（默认：`*`） |
| `cron_day_of_week` | string | Cron 星期字段（默认：`*`） |
| `is_active` | bool | 是否启用（默认：`true`） |
| `is_processing` | bool | 是否正在执行中（默认：`false`） |
| `only_when_online` | bool | 仅在服务器在线时执行（默认：`false`） |
| `last_run_at` | timestamp | 上次执行时间 |
| `next_run_at` | timestamp | 下次执行时间 |

**核心方法**：
```php
// 使用 dragonmantank/cron-expression 库解析 Cron 表达式
public function getNextRunDate(): CarbonImmutable
{
    $formatted = sprintf('%s %s %s %s %s', 
        $this->cron_minute, 
        $this->cron_hour, 
        $this->cron_day_of_month, 
        $this->cron_month, 
        $this->cron_day_of_week
    );
    return CarbonImmutable::instance(
        (new CronExpression($formatted))->getNextRunDate()
    );
}
```

### 2.2 Task 模型（任务）

**文件**：`app/Models/Task.php`

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | int | 主键 |
| `schedule_id` | int | 关联排程 ID |
| `sequence_id` | int | 执行顺序（从 1 开始） |
| `action` | string | 任务类型：`power` / `command` / `backup` |
| `payload` | text | 任务参数（电源动作、命令内容、备份忽略文件） |
| `time_offset` | int | 延迟执行秒数（0-900） |
| `is_queued` | bool | 是否已加入队列 |
| `continue_on_failure` | bool | 失败后是否继续执行后续任务 |

**任务类型常量**：
```php
public const ACTION_POWER = 'power';    // 电源操作
public const ACTION_COMMAND = 'command'; // 发送命令
public const ACTION_BACKUP = 'backup';   // 创建备份
```

### 2.3 TaskLog 模型（任务日志）

**文件**：`app/Models/TaskLog.php`

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | int | 主键 |
| `task_id` | int | 关联任务 ID |
| `run_status` | int | 执行状态 |
| `run_time` | datetime | 执行时间 |

---

## 3. Cron 解析与调度触发

### 3.1 系统级调度入口

**文件**：`app/Console/Kernel.php`

Laravel 调度器每分钟执行一次 `ProcessRunnableCommand`：

```php
// 每分钟执行，且防止重叠
$schedule->command(ProcessRunnableCommand::class)->everyMinute()->withoutOverlapping();
```

### 3.2 可运行排程查询

**文件**：`app/Console/Commands/Schedule/ProcessRunnableCommand.php`

```php
$schedules = Schedule::query()
    ->with('tasks')
    ->whereRelation('server', fn (Builder $builder) => $builder->whereNull('status'))
    ->where('is_active', true)
    ->where('is_processing', false)
    ->whereRaw('next_run_at <= NOW()')
    ->get();
```

**查询条件解读**：
1. 预加载关联的 tasks
2. 服务器状态为正常（`status` 为 null）
3. 排程已启用（`is_active = true`）
4. 排程未在执行中（`is_processing = false`）
5. 已到达下次执行时间（`next_run_at <= NOW()`）

### 3.3 Cron 表达式解析

使用第三方库 `dragonmantank/cron-expression` 进行 Cron 解析：

```php
// Schedule.php:119-124
public function getNextRunDate(): CarbonImmutable
{
    $formatted = sprintf('%s %s %s %s %s', 
        $this->cron_minute, $this->cron_hour, 
        $this->cron_day_of_month, $this->cron_month, 
        $this->cron_day_of_week
    );
    return CarbonImmutable::instance(
        (new CronExpression($formatted))->getNextRunDate()
    );
}
```

---

## 4. 任务派发流程

### 4.1 ProcessScheduleService（排程处理器）

**文件**：`app/Services/Schedules/ProcessScheduleService.php`

**核心流程**：

```php
public function handle(Schedule $schedule, bool $now = false): void
{
    // 1. 获取排程的第一个任务（按 sequence_id 排序）
    $task = $schedule->tasks()->orderBy('sequence_id')->first();
    
    // 2. 数据库事务：标记排程为处理中，计算下次执行时间
    $this->connection->transaction(function () use ($schedule, $task) {
        $schedule->forceFill([
            'is_processing' => true,
            'next_run_at' => $schedule->getNextRunDate(),
        ])->saveOrFail();
        $task->update(['is_queued' => true]);
    });
    
    // 3. 创建任务 Job
    $job = new RunTaskJob($task, $now);
    
    // 4. 仅在线检查（如果配置了 only_when_online）
    if ($schedule->only_when_online) {
        $details = $this->serverRepository->setServer($schedule->server)->getDetails();
        $state = $details['state'] ?? 'offline';
        if (in_array($state, ['offline', 'stopping'])) {
            $job->failed();
            return;
        }
    }
    
    // 5. 派发任务到队列
    if (!$now) {
        $this->dispatcher->dispatch($job->delay($task->time_offset));
    } else {
        $this->dispatcher->dispatchNow($job);
    }
}
```

**关键点**：
- 使用数据库事务确保状态一致性
- 预计算下次执行时间并保存，避免重复执行
- 支持 `only_when_online` 选项，服务器离线时跳过
- 支持延迟执行（`time_offset`）

### 4.2 RunTaskJob（任务执行者）

**文件**：`app/Jobs/Schedule/RunTaskJob.php`

**任务分发逻辑**：

```php
public function handle(
    DaemonCommandRepository $commandRepository,
    InitiateBackupService $backupService,
    DaemonPowerRepository $powerRepository,
) {
    // 前置检查
    if (!$this->task->schedule->is_active && !$this->manualRun) {
        $this->markTaskNotQueued();
        $this->markScheduleComplete();
        return;
    }
    
    // 按任务类型分发
    try {
        switch ($this->task->action) {
            case Task::ACTION_POWER:
                $powerRepository->setServer($server)->send($this->task->payload);
                break;
            case Task::ACTION_COMMAND:
                $commandRepository->setServer($server)->send($this->task->payload);
                break;
            case Task::ACTION_BACKUP:
                $backupService->setIgnoredFiles(explode(PHP_EOL, $this->task->payload))
                    ->handle($server, null, true);
                break;
        }
    } catch (\Exception $exception) {
        // 如果配置了 continue_on_failure 且是连接异常，忽略错误继续
        if (!($this->task->continue_on_failure && $exception instanceof DaemonConnectionException)) {
            throw $exception;
        }
    }
    
    // 标记当前任务完成，派发下一个任务
    $this->markTaskNotQueued();
    $this->queueNextTask();
}
```

**任务链式执行**：

```php
private function queueNextTask()
{
    $nextTask = Task::query()
        ->where('schedule_id', $this->task->schedule_id)
        ->orderBy('sequence_id', 'asc')
        ->where('sequence_id', '>', $this->task->sequence_id)
        ->first();
    
    if (is_null($nextTask)) {
        $this->markScheduleComplete();
        return;
    }
    
    $nextTask->update(['is_queued' => true]);
    $this->dispatch((new self($nextTask, $this->manualRun))
        ->delay($nextTask->time_offset));
}
```

---

## 5. 下发通道（Daemon 通信）

### 5.1 DaemonRepository 基类

**文件**：`app/Repositories/Wings/DaemonRepository.php`

```php
abstract class DaemonRepository
{
    public function getHttpClient(array $headers = []): Client
    {
        return new Client([
            'verify' => $this->app->environment('production'),
            'base_uri' => $this->node->getConnectionAddress(),
            'timeout' => config('pterodactyl.guzzle.timeout'),
            'connect_timeout' => config('pterodactyl.guzzle.connect_timeout'),
            'headers' => array_merge($headers, [
                'Authorization' => 'Bearer ' . $this->node->getDecryptedKey(),
                'Accept' => 'application/json',
                'Content-Type' => 'application/json',
            ]),
        ]);
    }
}
```

**通信机制**：
- 使用 Guzzle HTTP 客户端
- 基于节点的连接地址和认证密钥
- Bearer Token 认证方式
- 生产环境启用 SSL 证书验证

### 5.2 电源操作通道

**文件**：`app/Repositories/Wings/DaemonPowerRepository.php`

```php
public function send(string $action): ResponseInterface
{
    return $this->getHttpClient()->post(
        sprintf('/api/servers/%s/power', $this->server->uuid),
        ['json' => ['action' => $action]]
    );
}
```

**API 端点**：`POST /api/servers/{uuid}/power`

**Payload**：`{"action": "start|stop|restart|kill"}`

### 5.3 命令发送通道

**文件**：`app/Repositories/Wings/DaemonCommandRepository.php`

```php
public function send(array|string $command): ResponseInterface
{
    return $this->getHttpClient()->post(
        sprintf('/api/servers/%s/commands', $this->server->uuid),
        ['json' => ['commands' => is_array($command) ? $command : [$command]]]
    );
}
```

**API 端点**：`POST /api/servers/{uuid}/commands`

**Payload**：`{"commands": ["command1", "command2"]}`

### 5.4 备份操作通道

**文件**：`app/Repositories/Wings/DaemonBackupRepository.php`

```php
public function backup(Backup $backup): ResponseInterface
{
    return $this->getHttpClient()->post(
        sprintf('/api/servers/%s/backup', $this->server->uuid),
        ['json' => [
            'adapter' => $this->adapter ?? config('backups.default'),
            'uuid' => $backup->uuid,
            'ignore' => implode("\n", $backup->ignored_files),
        ]]
    );
}
```

**API 端点**：`POST /api/servers/{uuid}/backup`

---

## 6. 失败重试机制

### 6.1 Laravel 队列配置

**文件**：`config/queue.php`

```php
'redis' => [
    'driver' => 'redis',
    'connection' => env('REDIS_QUEUE_CONNECTION', 'default'),
    'queue' => env('REDIS_QUEUE', 'standard'),
    'retry_after' => (int) env('REDIS_QUEUE_RETRY_AFTER', 90), // 90秒后重试
    'block_for' => null,
    'after_commit' => false,
],
```

**队列特性**：
- 默认使用 Redis 作为队列后端
- 任务放入 `standard` 队列
- `retry_after` 配置：任务执行超过 90 秒未完成，自动重新排队

### 6.2 任务级失败处理

**RunTaskJob::failed() 方法**：

```php
public function failed(?\Exception $exception = null)
{
    $this->markTaskNotQueued();
    $this->markScheduleComplete();
}
```

**失败后的清理工作**：
1. 标记当前任务为非排队状态
2. 标记整个排程为完成状态
3. **注意**：不会自动重试失败的任务，后续任务也不会执行

### 6.3 业务级容错

**continue_on_failure 机制**：

```php
// RunTaskJob.php:74-80
catch (\Exception $exception) {
    // 如果配置了 continue_on_failure 且是守护进程连接异常
    if (!($this->task->continue_on_failure && $exception instanceof DaemonConnectionException)) {
        throw $exception;
    }
    // 否则，忽略错误继续执行后续任务
}
```

**仅当满足以下所有条件时才继续**：
- 任务配置了 `continue_on_failure = true`
- 异常类型是 `DaemonConnectionException`（网络/连接问题）

### 6.4 失败日志存储

**配置**：
```php
'failed' => [
    'driver' => env('QUEUE_FAILED_DRIVER', 'database-uuids'),
    'database' => env('DB_CONNECTION', 'mysql'),
    'table' => 'failed_jobs',
],
```

所有失败的队列任务会被记录到 `failed_jobs` 表中，包含：
- 任务类名
- 异常信息
- 失败时间
- 任务 payload

---

## 7. 完整流程图

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Laravel Scheduler                           │
│              每分钟执行 p:schedule:process 命令                      │
└─────────────────────────────────┬───────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    ProcessRunnableCommand                           │
│  查询条件: is_active=true AND is_processing=false AND next_run_at<=NOW() │
│  遍历所有符合条件的 Schedule                                         │
└─────────────────────────────────┬───────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    ProcessScheduleService                           │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  事务:                                                         │  │
│  │    1. schedule.is_processing = true                            │  │
│  │    2. 计算并保存 next_run_at                                    │  │
│  │    3. task.is_queued = true                                    │  │
│  └───────────────────────────────────────────────────────────────┘  │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  可选检查: only_when_online                                    │  │
│  │    → 调用 Wings API 查询服务器状态                              │  │
│  │    → offline/stopping 时直接标记失败                           │  │
│  └───────────────────────────────────────────────────────────────┘  │
│  派发 RunTaskJob 到队列（延迟 time_offset 秒）                       │
└─────────────────────────────────┬───────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│                         RunTaskJob                                 │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  前置检查:                                                     │  │
│  │    1. schedule.is_active 或 manualRun                          │  │
│  │    2. server.status 为 null（正常状态）                         │  │
│  └───────────────────────────────────────────────────────────────┘  │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  按 action 分发到对应通道:                                      │  │
│  │    ├─ power   → DaemonPowerRepository::send()                  │  │
│  │    ├─ command → DaemonCommandRepository::send()                │  │
│  │    └─ backup  → InitiateBackupService::handle()                │  │
│  └───────────────────────────────────────────────────────────────┘  │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  异常处理:                                                      │  │
│  │    ├─ continue_on_failure + DaemonConnectionException → 继续   │  │
│  │    └─ 其他异常 → 抛出 → 调用 failed() 清理                       │  │
│  └───────────────────────────────────────────────────────────────┘  │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  完成后续:                                                      │  │
│  │    1. 当前 task.is_queued = false                               │  │
│  │    2. 查询下一个 sequence_id 的 task                             │  │
│  │    3. 存在 → 派发新的 RunTaskJob（链式执行）                     │  │
│  │    4. 不存在 → schedule.is_processing = false，更新 last_run_at │  │
│  └───────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        Wings Daemon API                            │
│  ┌─────────────┐  ┌────────────────┐  ┌─────────────────────────┐  │
│  │ /power      │  │ /commands      │  │ /backup                 │  │
│  │ 执行电源操作 │  │ 发送控制台命令  │  │ 触发备份任务             │  │
│  └─────────────┘  └────────────────┘  └─────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 8. 关键设计要点

### 8.1 幂等性设计
- `is_processing` 标记防止排程重复执行
- `next_run_at` 预计算确保调度准确性
- `withoutOverlapping` 防止命令重叠执行

### 8.2 事务一致性
- 排程状态更新和任务入队在同一数据库事务中
- 确保状态变更的原子性

### 8.3 容错机制
- 网络连接异常可配置继续执行
- 失败任务记录到 `failed_jobs` 表
- 服务器状态检查避免无效执行

### 8.4 链式执行
- 任务按 `sequence_id` 顺序执行
- 每个任务完成后自动派发下一个
- 支持任务间延迟（`time_offset`）

---

## 9. 潜在问题与优化建议

### 9.1 现有限制
1. **无自动重试**：任务失败后不会自动重试，需要手动干预
2. **无进度追踪**：`TaskLog` 表存在但代码中未见实际写入逻辑
3. **单点执行**：`ProcessRunnableCommand` 单线程遍历所有排程，高并发下可能成为瓶颈
4. **重试机制简单**：仅依赖 Laravel Queue 的 `retry_after`，无指数退避策略

### 9.2 可能的优化方向
1. 为失败任务实现指数退避重试策略
2. 批量处理可运行排程，提升调度效率
3. 完善 `TaskLog` 记录，支持执行历史查询
4. 增加任务执行超时控制
5. 支持任务执行结果回调通知
