# Pterodactyl Panel 定时任务调度机制分析

## 一、整体架构概览

Pterodactyl Panel 的定时任务调度系统采用了 **Laravel 调度器 + 队列任务** 的混合架构，核心流程如下：

```
Laravel Cron (每分钟)
    ↓
ProcessRunnableCommand (筛选可执行任务)
    ↓
ProcessScheduleService (预处理任务)
    ↓
RunTaskJob (队列执行，下发 Wings 指令)
    ↓
Wings Daemon API (实际执行服务器操作)
```

---

## 二、核心代码链路详解

### 1. 调度入口：Kernel.php

**文件位置**：`app/Console/Kernel.php:29-50`

Laravel 的调度器配置入口，每分钟执行一次 `ProcessRunnableCommand`：

```php
protected function schedule(Schedule $schedule): void
{
    // 每分钟执行一次定时任务处理，且不重叠
    $schedule->command(ProcessRunnableCommand::class)->everyMinute()->withoutOverlapping();
}
```

**关键设计点**：
- 使用 `everyMinute()` 保证每分钟检查一次待执行任务
- `withoutOverlapping()` 防止任务重叠执行（避免重复调度）

---

### 2. 任务筛选：ProcessRunnableCommand

**文件位置**：`app/Console/Commands/Schedule/ProcessRunnableCommand.php:20-74`

负责从数据库中筛选出需要执行的定时任务：

#### 筛选条件（第 22-28 行）：
```php
$schedules = Schedule::query()
    ->with('tasks')
    ->whereRelation('server', fn (Builder $builder) => $builder->whereNull('status'))
    ->where('is_active', true)           // 任务处于激活状态
    ->where('is_processing', false)       // 当前不在处理中
    ->whereRaw('next_run_at <= NOW()')    // 下一次执行时间已到
    ->get();
```

**筛选逻辑说明**：
1. **服务器状态正常**：`server.status IS NULL`（非安装/重装/暂停等状态）
2. **任务激活**：`is_active = true`
3. **非处理中**：`is_processing = false`（避免重复执行）
4. **到达执行时间**：`next_run_at <= NOW()`

#### 处理流程（第 37-42 行）：
```php
foreach ($schedules as $schedule) {
    $this->processSchedule($schedule);
}
```

调用 `ProcessScheduleService::handle()` 处理每个符合条件的任务。

---

### 3. Cron 表达式配置与解析

#### Schedule 模型字段定义

**文件位置**：`app/Models/Schedule.php:14-28, 57-70`

Cron 表达式被拆分为 5 个独立字段存储：

| 字段名 | 说明 | 对应 Cron 位 |
|--------|------|-------------|
| `cron_minute` | 分钟 | 第 1 位 |
| `cron_hour` | 小时 | 第 2 位 |
| `cron_day_of_month` | 日期 | 第 3 位 |
| `cron_month` | 月份 | 第 4 位 |
| `cron_day_of_week` | 星期 | 第 5 位 |

#### Cron 表达式解析（第 119-124 行）：
```php
public function getNextRunDate(): CarbonImmutable
{
    // 组合成标准 Cron 格式：分 时 日 月 周
    $formatted = sprintf('%s %s %s %s %s', 
        $this->cron_minute, 
        $this->cron_hour, 
        $this->cron_day_of_month, 
        $this->cron_month, 
        $this->cron_day_of_week
    );

    return CarbonImmutable::instance((new CronExpression($formatted))->getNextRunDate());
}
```

**使用的库**：`dragonmantank/cron-expression`（CronExpression 类）

#### API 层配置入口

**文件位置**：`app/Http/Controllers/Api/Client/Servers/ScheduleController.php:53-77, 170-183`

创建/更新 Schedule 时，通过 `getNextRunAt()` 方法计算下一次执行时间：

```php
protected function getNextRunAt(Request $request): Carbon
{
    return Utilities::getScheduleNextRunDate(
        $request->input('minute'),
        $request->input('hour'),
        $request->input('day_of_month'),
        $request->input('month'),
        $request->input('day_of_week')
    );
}
```

---

### 4. 任务预处理：ProcessScheduleService

**文件位置**：`app/Services/Schedules/ProcessScheduleService.php:28-85`

负责调度任务的预处理和队列分发。

#### 核心处理流程：

##### 步骤 1：数据库事务（第 35-42 行）
```php
$this->connection->transaction(function () use ($schedule, $task) {
    // 标记调度任务为处理中，并更新下一次执行时间
    $schedule->forceFill([
        'is_processing' => true,
        'next_run_at' => $schedule->getNextRunDate(),
    ])->saveOrFail();

    // 标记第一个任务已入队
    $task->update(['is_queued' => true]);
});
```

##### 步骤 2：仅在线服务器检查（第 45-68 行）
如果设置了 `only_when_online`，会检查服务器状态：
```php
if ($schedule->only_when_online) {
    $details = $this->serverRepository->setServer($schedule->server)->getDetails();
    $state = $details['state'] ?? 'offline';
    
    // 服务器离线或停止中，不执行任务
    if (in_array($state, ['offline', 'stopping'])) {
        $job->failed();
        return;
    }
}
```

##### 步骤 3：队列分发（第 70-84 行）
```php
$job = new RunTaskJob($task, $now);

if (!$now) {
    // 延迟分发，支持任务的 time_offset
    $this->dispatcher->dispatch($job->delay($task->time_offset));
} else {
    // 手动触发时立即执行
    $this->dispatcher->dispatchNow($job);
}
```

---

### 5. 任务执行：RunTaskJob

**文件位置**：`app/Jobs/Schedule/RunTaskJob.php:35-134`

队列任务类，负责实际下发指令到 Wings Daemon。

#### 支持的任务类型（第 61-73 行）：
```php
switch ($this->task->action) {
    case Task::ACTION_POWER:     // 电源操作
        $powerRepository->setServer($server)->send($this->task->payload);
        break;
    case Task::ACTION_COMMAND:   // 发送命令
        $commandRepository->setServer($server)->send($this->task->payload);
        break;
    case Task::ACTION_BACKUP:    // 创建备份
        $backupService->handle($server, null, true);
        break;
}
```

#### Task 模型字段说明

**文件位置**：`app/Models/Task.php:58-66`

| 字段名 | 说明 |
|--------|------|
| `sequence_id` | 任务序列 ID，决定执行顺序 |
| `action` | 任务类型：`power`/`command`/`backup` |
| `payload` | 任务参数（如电源动作、命令内容） |
| `time_offset` | 延迟执行秒数（相对于前一个任务） |
| `continue_on_failure` | 失败后是否继续执行后续任务 |

#### 任务链执行（第 98-115 行）：
```php
private function queueNextTask()
{
    // 查找下一个序列的任务
    $nextTask = Task::query()->where('schedule_id', $this->task->schedule_id)
        ->orderBy('sequence_id', 'asc')
        ->where('sequence_id', '>', $this->task->sequence_id)
        ->first();

    if (is_null($nextTask)) {
        $this->markScheduleComplete();  // 所有任务完成
        return;
    }

    $nextTask->update(['is_queued' => true]);
    // 分发下一个任务（支持延迟）
    $this->dispatch((new self($nextTask, $this->manualRun))->delay($nextTask->time_offset));
}
```

#### 任务完成标记（第 120-126 行）：
```php
private function markScheduleComplete()
{
    $this->task->schedule()->update([
        'is_processing' => false,        // 标记为非处理中
        'last_run_at' => CarbonImmutable::now()->toDateTimeString(),
    ]);
}
```

---

### 6. Wings 指令下发

#### 基础 HTTP 客户端：DaemonRepository

**文件位置**：`app/Repositories/Wings/DaemonRepository.php:49-64`

```php
public function getHttpClient(array $headers = []): Client
{
    return new Client([
        'verify' => $this->app->environment('production'),
        'base_uri' => $this->node->getConnectionAddress(),  // Node 的连接地址
        'timeout' => config('pterodactyl.guzzle.timeout'),
        'connect_timeout' => config('pterodactyl.guzzle.connect_timeout'),
        'headers' => array_merge($headers, [
            'Authorization' => 'Bearer ' . $this->node->getDecryptedKey(),  // Node 的认证 Token
            'Accept' => 'application/json',
            'Content-Type' => 'application/json',
        ]),
    ]);
}
```

#### 电源操作：DaemonPowerRepository

**文件位置**：`app/Repositories/Wings/DaemonPowerRepository.php:22-34`

```php
public function send(string $action): ResponseInterface
{
    return $this->getHttpClient()->post(
        sprintf('/api/servers/%s/power', $this->server->uuid),
        ['json' => ['action' => $action]]  // start/stop/restart/kill
    );
}
```

#### 命令发送：DaemonCommandRepository

**文件位置**：`app/Repositories/Wings/DaemonCommandRepository.php:22-36`

```php
public function send(array|string $command): ResponseInterface
{
    return $this->getHttpClient()->post(
        sprintf('/api/servers/%s/commands', $this->server->uuid),
        [
            'json' => ['commands' => is_array($command) ? $command : [$command]],
        ]
    );
}
```

#### 备份操作：InitiateBackupService

**文件位置**：通过 `InitiateBackupService::handle()` 调用，最终通过 `DaemonBackupRepository` 下发到 Wings。

---

### 7. 结果回写与失败处理

#### 成功流程回写
1. **RunTaskJob::handle()** 执行成功后（第 82-83 行）：
   ```php
   $this->markTaskNotQueued();    // is_queued = false
   $this->queueNextTask();        // 触发下一个任务
   ```

2. 所有任务执行完成后，调用 `markScheduleComplete()`：
   - `is_processing = false`
   - 更新 `last_run_at` 为当前时间

#### 失败处理机制

**文件位置**：`app/Jobs/Schedule/RunTaskJob.php:74-80, 89-93`

##### 失败场景 1：Wings 连接异常（第 74-80 行）
```php
catch (\Exception $exception) {
    // 如果设置了 continue_on_failure 且是 Wings 连接异常，继续执行
    if (!($this->task->continue_on_failure && $exception instanceof DaemonConnectionException)) {
        throw $exception;  // 否则抛出异常触发 failed()
    }
}
```

##### 失败场景 2：任务失败回调（第 89-93 行）
Laravel 队列任务的 `failed()` 方法：
```php
public function failed(?\Exception $exception = null)
{
    $this->markTaskNotQueued();    // 标记任务未入队
    $this->markScheduleComplete(); // 标记调度完成（终止后续任务）
}
```

##### 异常记录：ProcessRunnableCommand

**文件位置**：`app/Console/Commands/Schedule/ProcessRunnableCommand.php:69-73`
```php
catch (\Throwable $exception) {
    Log::error($exception, ['schedule_id' => $schedule->id]);  // 记录错误日志
    $this->error("An error was encountered while processing Schedule #$schedule->id: " . $exception->getMessage());
}
```

---

### 8. 任务列表配置规则

**文件位置**：`app/Http/Controllers/Api/Client/Servers/ScheduleTaskController.php:40-174`

#### 8.1 每个计划的任务上限

```php
$limit = config('pterodactyl.client_features.schedules.per_schedule_task_limit', 10);
if ($schedule->tasks()->count() >= $limit) {
    throw new ServiceLimitExceededException("Schedules may not have more than $limit tasks associated with them. Creating this task would put this schedule over the limit.");
}
```

**配置说明**：
- **配置项**：`pterodactyl.client_features.schedules.per_schedule_task_limit`
- **环境变量**：`PTERODACTYL_PER_SCHEDULE_TASK_LIMIT`
- **默认值**：10
- **文件位置**：`config/pterodactyl.php:112-115`

#### 8.2 任务序号（sequence_id）调整逻辑

任务序号决定了任务链的执行顺序，支持在创建和更新时动态调整。

##### 创建任务时的序号调整（第 55-83 行）：
```php
$this->connection->transaction(function () use ($request, $schedule, $lastTask) {
    $sequenceId = ($lastTask->sequence_id ?? 0) + 1;
    $requestSequenceId = $request->integer('sequence_id', $sequenceId);

    // 确保序号至少为 1
    if ($requestSequenceId < 1) {
        $requestSequenceId = 1;
    }

    // 如果请求的序号小于下一个可用序号，需要将后续任务的序号后移
    if ($requestSequenceId < $sequenceId) {
        $schedule->tasks()
            ->where('sequence_id', '>=', $requestSequenceId)
            ->increment('sequence_id');
        $sequenceId = $requestSequenceId;
    }

    return $this->repository->create([...]);
});
```

##### 更新任务时的序号调整（第 111-138 行）：
```php
$this->connection->transaction(function () use ($request, $schedule, $task) {
    $sequenceId = $request->integer('sequence_id', $task->sequence_id);
    
    if ($sequenceId < 1) {
        $sequenceId = 1;
    }

    // 前移：目标序号 < 当前序号 → 中间任务序号 +1
    if ($sequenceId < $task->sequence_id) {
        $schedule->tasks()
            ->where('sequence_id', '>=', $sequenceId)
            ->where('sequence_id', '<', $task->sequence_id)
            ->increment('sequence_id');
    }
    // 后移：目标序号 > 当前序号 → 中间任务序号 -1
    elseif ($sequenceId > $task->sequence_id) {
        $schedule->tasks()
            ->where('sequence_id', '>', $task->sequence_id)
            ->where('sequence_id', '<=', $sequenceId)
            ->decrement('sequence_id');
    }

    $this->repository->update($task->id, [...]);
});
```

##### 删除任务时的序号调整（第 166-168 行）：
```php
$schedule->tasks()
    ->where('sequence_id', '>', $task->sequence_id)
    ->decrement('sequence_id');
```

**序号调整规则总结**：
| 操作 | 场景 | 调整逻辑 |
|------|------|----------|
| 创建 | 插入到中间位置 | 目标序号及以后的任务序号 +1 |
| 创建 | 追加到末尾 | 直接使用最大序号 +1 |
| 更新 | 前移（目标 < 当前） | 目标到当前之间的任务序号 +1 |
| 更新 | 后移（目标 > 当前） | 当前到目标之间的任务序号 -1 |
| 删除 | 任意位置 | 被删任务之后的任务序号 -1 |

#### 8.3 备份动作限制

**文件位置**：`app/Http/Controllers/Api/Client/Servers/ScheduleTaskController.php:47-49, 107-109`

```php
if ($server->backup_limit === 0 && $request->action === 'backup') {
    throw new HttpForbiddenException("A backup task cannot be created when the server's backup limit is set to 0.");
}
```

**限制条件**：
1. 当服务器的 `backup_limit = 0` 时，不允许创建备份类型的任务
2. 该限制在任务创建和更新时都会检查

---

### 9. 备份动作深度分析

#### 9.1 备份任务执行流程

**文件位置**：`app/Services/Backups/InitiateBackupService.php:76-126`

当 RunTaskJob 触发备份动作时，会调用 `InitiateBackupService::handle()`，执行以下步骤：

##### 步骤 1：频率限制检查（第 78-87 行）
```php
$limit = config('backups.throttles.limit');
$period = config('backups.throttles.period');
if ($period > 0) {
    $previous = $this->repository->getBackupsGeneratedDuringTimespan($server->id, $period);
    if ($previous->count() >= $limit) {
        throw new TooManyRequestsHttpException(...);
    }
}
```

##### 步骤 2：备份数量限制检查（第 91-107 行）
```php
$successful = $this->repository->getNonFailedBackups($server);
if (!$server->backup_limit || $successful->count() >= $server->backup_limit) {
    // 定时任务调用时 override=true，会尝试删除最旧的非锁定备份
    if (!$override || $server->backup_limit <= 0) {
        throw new TooManyBackupsException($server->backup_limit);
    }
    
    // 删除最旧的非锁定备份
    $oldest = $successful->where('is_locked', false)->orderBy('created_at')->first();
    if (!$oldest) {
        throw new TooManyBackupsException($server->backup_limit);
    }
    $this->deleteBackupService->handle($oldest);
}
```

**定时任务备份的特殊性**：
- 调用时 `override = true`（来自 `RunTaskJob:69`）
- 达到备份上限时会自动删除最旧的非锁定备份
- 如果所有备份都被锁定，则抛出异常

##### 步骤 3：创建备份记录并下发到 Wings（第 109-125 行）
```php
return $this->connection->transaction(function () use ($server, $name) {
    $backup = $this->repository->create([
        'server_id' => $server->id,
        'uuid' => Uuid::uuid4()->toString(),
        'name' => trim($name) ?: sprintf('Backup at %s', CarbonImmutable::now()->toDateTimeString()),
        'ignored_files' => array_values($this->ignoredFiles),
        'disk' => $this->backupManager->getDefaultAdapter(),
        'is_locked' => $this->isLocked,
    ], true, true);

    // 下发备份指令到 Wings
    $this->daemonBackupRepository->setServer($server)
        ->setBackupAdapter($this->backupManager->getDefaultAdapter())
        ->backup($backup);

    return $backup;
});
```

#### 9.2 备份结果回写流程

**文件位置**：`app/Http/Controllers/Api/Remote/Backups/BackupStatusController.php:32-81`

Wings 完成备份后，会通过回调接口将结果回写到 Panel：

##### 回调接口路由
```
POST /api/remote/backups/{backup}/status
```

##### 回写处理逻辑
```php
public function index(ReportBackupCompleteRequest $request, string $backup): JsonResponse
{
    $node = $request->attributes->get('node');
    
    $model = Backup::query()->where('uuid', $backup)->firstOrFail();
    
    // 验证节点权限
    if ($model->server->node_id !== $node->id) {
        throw new HttpForbiddenException(...);
    }
    
    // 防止重复更新
    if ($model->is_successful) {
        throw new BadRequestHttpException(...);
    }

    $action = $request->boolean('successful') ? 'server:backup.complete' : 'server:backup.fail';
    $log = Activity::event($action)->subject($model, $model->server)->property('name', $model->name);

    $log->transaction(function () use ($model, $request) {
        $successful = $request->boolean('successful');

        $model->fill([
            'is_successful' => $successful,
            'is_locked' => $successful ? $model->is_locked : false,
            'checksum' => $successful ? ($request->input('checksum_type') . ':' . $request->input('checksum')) : null,
            'bytes' => $successful ? $request->input('size') : 0,
            'completed_at' => CarbonImmutable::now(),
        ])->save();

        // S3 多部分上传完成处理
        $adapter = $this->backupManager->adapter();
        if ($adapter instanceof S3Filesystem) {
            $this->completeMultipartUpload($model, $adapter, $successful, $request->input('parts'));
        }
    });

    return new JsonResponse([], JsonResponse::HTTP_NO_CONTENT);
}
```

##### 回写字段说明（Backup 模型）
**文件位置**：`app/Models/Backup.php:47-62`

| 字段 | 类型 | 回写值 |
|------|------|--------|
| `is_successful` | bool | 备份是否成功 |
| `is_locked` | bool | 失败时强制设为 false |
| `checksum` | string | 成功时：`checksum_type:checksum` |
| `bytes` | int | 成功时：备份文件大小 |
| `completed_at` | datetime | 回写时间 |

#### 9.3 备份操作的异常路径深度分析

备份操作是三种任务类型中代码路径最复杂的，因为 `InitiateBackupService::handle()` 内部串联了多个子步骤，每个子步骤可能抛出不同类型的异常。理解这些异常在 `RunTaskJob` 的 catch 块中如何被处理，是区分"下发失败"和"执行失败"的关键。

##### 9.3.1 备份调用的完整代码路径

```
RunTaskJob::handle()                          ← 外层 try-catch
  │
  ├─ backupService->setIgnoredFiles(...)->handle($server, null, true)
  │    │
  │    │  ──── InitiateBackupService::handle() 内部 ────
  │    │
  │    ├─ [步骤1] 频率限制检查
  │    │   └─ 抛出 TooManyRequestsHttpException (extends HttpException)
  │    │
  │    ├─ [步骤2] 备份数量限制检查
  │    │   ├─ 抛出 TooManyBackupsException (extends DisplayException extends Exception)
  │    │   └─ 若 override=true → 删除最旧备份
  │    │       └─ DeleteBackupService::handle()
  │    │           ├─ 抛出 BackupLockedException (extends DisplayException)
  │    │           └─ DaemonBackupRepository::delete() → 抛出 DaemonConnectionException
  │    │
  │    └─ [步骤3] 数据库事务内：
  │        ├─ 创建 Backup 记录 (数据库操作)
  │        └─ DaemonBackupRepository::backup()  ← 向 Wings 发起 HTTP POST
  │            └─ 抛出 DaemonConnectionException (extends DisplayException extends Exception)
  │
  ├─ catch (\Exception $exception)             ← RunTaskJob 的 catch 块
  │    └─ 判断是否 continue_on_failure + DaemonConnectionException
  │         ├─ 是 → 吞掉异常，继续执行 queueNextTask()
  │         └─ 否 → throw，触发 RunTaskJob::failed()
  │
  ├─ markTaskNotQueued()                       ← 正常路径
  └─ queueNextTask()                           ← 正常路径
```

##### 9.3.2 "下发失败"的精确定义

**下发失败** = 在 `RunTaskJob::handle()` 的 try 块中，调用备份服务时抛出了异常。

这包含以下子场景：

| 场景 | 异常类型 | 异常继承链 | 触发条件 |
|------|----------|-----------|----------|
| 备份频率超限 | `TooManyRequestsHttpException` | `HttpException` → `RuntimeException` → `Exception` | 在 `backups.throttles.period` 秒内创建备份数超过 `backups.throttles.limit` |
| 备份数量超限 | `TooManyBackupsException` | `DisplayException` → `PterodactylException` → `Exception` | 成功备份数 ≥ `server.backup_limit` 且无旧备份可删 |
| 备份数量超限(全锁定) | `TooManyBackupsException` | 同上 | 成功备份数 ≥ `server.backup_limit` 且所有备份都被锁定 |
| 旧备份锁定 | `BackupLockedException` | `DisplayException` → `PterodactylException` → `Exception` | 删除旧备份时发现备份被锁定 |
| Wings 连接失败(备份指令) | `DaemonConnectionException` | `DisplayException` → `PterodactylException` → `Exception` | Wings 不可达、超时、或返回非 2xx |
| Wings 连接失败(删除旧备份) | `DaemonConnectionException` | 同上 | 删除旧备份时 Wings 不可达（404 除外） |
| 数据库写入失败 | `QueryException` | `RuntimeException` → `Exception` | 创建 Backup 记录时数据库异常 |

**关键代码**（`RunTaskJob.php:74-80`）：
```php
catch (\Exception $exception) {
    if (!($this->task->continue_on_failure && $exception instanceof DaemonConnectionException)) {
        throw $exception;
    }
}
```

##### 9.3.3 "执行失败"的精确定义

**执行失败** = `RunTaskJob::handle()` 正常返回后，Wings 在后台执行备份过程中失败。

```
Wings 后台执行备份
  │
  ├─ 备份成功 → Wings 回调 POST /api/remote/backups/{uuid}/status
  │              → BackupStatusController::index()
  │              → backup.is_successful = true, completed_at = now()
  │
  └─ 备份失败 → Wings 回调 POST /api/remote/backups/{uuid}/status
                 → BackupStatusController::index()
                 → backup.is_successful = false, completed_at = now()
                 → Activity Log: server:backup.fail
```

**下发失败与执行失败的本质区别**：

| 维度 | 下发失败 | 执行失败 |
|------|----------|----------|
| 发生时机 | RunTaskJob 执行期间 | RunTaskJob 已完成后 |
| 异常来源 | Panel 侧（PHP 代码） | Wings 侧（Go 代码） |
| 是否影响任务链 | 受 `continue_on_failure` 控制 | **不影响**，任务链已继续 |
| 记录方式 | Laravel 日志 / failed_jobs 表 | Activity Log (`server:backup.fail`) |
| Backup 记录状态 | 可能未创建，或已创建但 `is_successful=false, completed_at=null` | 已创建且回写 `is_successful=false, completed_at=now()` |

##### 9.3.4 中间状态：Backup 记录已创建但 Wings 未回调

当 `DaemonBackupRepository::backup()` 的 HTTP 请求成功发出（Wings 返回 202），但 Wings 在后台执行备份过程中崩溃或失去连接时，会出现：
- Backup 记录已存在（`is_successful=false, completed_at=null`）
- Wings 永远不会回调 `BackupStatusController`

这就是 `PruneOrphanedBackupsCommand` 存在的原因（`app/Console/Commands/Maintenance/PruneOrphanedBackupsCommand.php`），它会清理这种"悬挂"的备份记录。

---

### 10. 失败通知机制深度分析

#### 10.1 现有通知系统概览

**文件位置**：`app/Providers/EventServiceProvider.php:25-27`

```php
protected $listen = [
    ServerInstalledEvent::class => [ServerInstalledNotification::class],
];
```

当前系统仅在服务器安装完成时发送通知，**没有任何与定时任务相关的内置通知**。

#### 10.2 定时任务失败的记录方式

##### 方式 1：日志记录
**文件位置**：`app/Console/Commands/Schedule/ProcessRunnableCommand.php:69-73`
```php
catch (\Throwable $exception) {
    Log::error($exception, ['schedule_id' => $schedule->id]);
    $this->error("An error was encountered while processing Schedule #$schedule->id: " . $exception->getMessage());
}
```

##### 方式 2：Activity Log 记录
**文件位置**：`app/Http/Controllers/Api/Remote/Backups/BackupStatusController.php:55-56`
```php
$action = $request->boolean('successful') ? 'server:backup.complete' : 'server:backup.fail';
$log = Activity::event($action)->subject($model, $model->server)->property('name', $model->name);
```

**可用的 Activity 事件**：
| 事件 | 触发时机 |
|------|----------|
| `server:schedule.create` | 创建计划 |
| `server:schedule.update` | 更新计划 |
| `server:schedule.execute` | 手动执行计划 |
| `server:schedule.delete` | 删除计划 |
| `server:task.create` | 创建任务 |
| `server:task.update` | 更新任务 |
| `server:task.delete` | 删除任务 |
| `server:backup.complete` | 备份成功 |
| `server:backup.fail` | 备份失败 |

##### 方式 3：ActivityLogged 事件
**文件位置**：`app/Events/ActivityLogged.php:9-34`

每次 Activity Log 记录时都会触发 `ActivityLogged` 事件：
```php
class ActivityLogged extends Event
{
    public function __construct(public ActivityLog $model) {}

    public function is(string $event): bool { ... }
    public function isServerEvent(): bool { ... }
    public function isSystem(): bool { ... }
}
```

#### 10.3 失败通知扩展方案

虽然没有内置通知，但可以通过以下方式扩展：

##### 方案 1：监听 ActivityLogged 事件
```php
// 在 EventServiceProvider 中注册
ActivityLogged::class => [
    ScheduleFailedNotificationListener::class,
],

// 监听器示例
class ScheduleFailedNotificationListener
{
    public function handle(ActivityLogged $event)
    {
        if ($event->is('server:backup.fail')) {
            // 发送备份失败通知
            $backup = $event->model->subjects->first();
            $server = $backup->server;
            $user = $server->user;
            
            $user->notify(new BackupFailedNotification($backup));
        }
    }
}
```

##### 方案 2：扩展 RunTaskJob::failed() 方法
**文件位置**：`app/Jobs/Schedule/RunTaskJob.php:89-93`
```php
public function failed(?\Exception $exception = null)
{
    $this->markTaskNotQueued();
    $this->markScheduleComplete();
    
    // 扩展：发送失败通知
    $this->task->server->user->notify(
        new ScheduleTaskFailedNotification($this->task, $exception)
    );
}
```

##### 方案 3：监听 Laravel Queue 失败事件
```php
// 在 AppServiceProvider 中注册
Queue::failing(function (JobFailed $event) {
    if ($event->job instanceof RunTaskJob) {
        $task = $event->job->task;
        // 发送通知
    }
});
```

#### 10.4 `continue_on_failure` 的精确作用范围

`continue_on_failure` 是 Task 模型上的布尔字段，控制任务失败后任务链是否继续执行。但其作用范围非常**狭窄**，仅对一种特定的异常类型生效。

##### 核心判断代码（RunTaskJob.php:74-80）

```php
catch (\Exception $exception) {
    if (!($this->task->continue_on_failure && $exception instanceof DaemonConnectionException)) {
        throw $exception;
    }
}
```

这个条件等价于：

```
continue_on_failure = true  AND  异常是 DaemonConnectionException
    → 吞掉异常，执行 markTaskNotQueued() + queueNextTask()，任务链继续

其他所有情况
    → 抛出异常，触发 RunTaskJob::failed()，任务链终止
```

##### `continue_on_failure` 对各种异常的实际效果

| 异常类型 | continue_on_failure=true | continue_on_failure=false |
|----------|-------------------------|--------------------------|
| `DaemonConnectionException`（Wings 连接失败） | **继续**执行任务链 | **终止**任务链 |
| `TooManyBackupsException`（备份超限） | **终止**任务链 | **终止**任务链 |
| `TooManyRequestsHttpException`（频率超限） | **终止**任务链 | **终止**任务链 |
| `BackupLockedException`（旧备份被锁定） | **终止**任务链 | **终止**任务链 |
| `QueryException`（数据库错误） | **终止**任务链 | **终止**任务链 |
| `InvalidArgumentException`（无效 action） | **终止**任务链 | **终止**任务链 |
| 任意 `\Exception` 子类（非 DaemonConnectionException） | **终止**任务链 | **终止**任务链 |

**结论**：`continue_on_failure` **只对 DaemonConnectionException 生效**，即只有"与 Wings 守护进程的网络通信失败"这一种场景可以被容忍。所有业务逻辑异常（备份超限、频率超限、锁定等）无论 `continue_on_failure` 如何设置，都会终止任务链。

##### 三种任务类型的 `continue_on_failure` 效果对比

| 任务类型 | 可能抛出 DaemonConnectionException 的位置 | continue_on_failure 生效场景 |
|----------|------------------------------------------|---------------------------|
| `power` | `DaemonPowerRepository::send()` — Wings 连接失败 | Wings 不可达时继续执行后续任务 |
| `command` | `DaemonCommandRepository::send()` — Wings 连接失败 | Wings 不可达时继续执行后续任务 |
| `backup` | `DaemonBackupRepository::backup()` — Wings 连接失败 | Wings 不可达时继续执行后续任务 |
| `backup` | `DaemonBackupRepository::delete()` — 删除旧备份时 Wings 不可达 | Wings 不可达时继续执行后续任务 |

**注意**：备份操作有两处可能抛出 `DaemonConnectionException`：
1. 删除旧备份时（`InitiateBackupService:106` → `DeleteBackupService:49` → `DaemonBackupRepository::delete()`）
2. 发起备份指令时（`InitiateBackupService:120-122` → `DaemonBackupRepository::backup()`）

这两处都被 `RunTaskJob` 的同一个 catch 块捕获，`continue_on_failure` 对两处都生效。

##### 任务链是否继续执行的完整判断流程

```
RunTaskJob::handle() 执行任务动作
  │
  ├─ 无异常
  │   └─ markTaskNotQueued() → queueNextTask() → 任务链继续 ✅
  │
  └─ 抛出异常
      │
      ├─ 异常是 DaemonConnectionException？
      │   ├─ 否 → throw → RunTaskJob::failed() → 任务链终止 ❌
      │   └─ 是
      │       ├─ continue_on_failure = true？
      │       │   ├─ 是 → 吞掉异常 → markTaskNotQueued() → queueNextTask() → 任务链继续 ✅
      │       │   └─ 否 → throw → RunTaskJob::failed() → 任务链终止 ❌
      │
      └─ 特殊：服务器 status != null（被暂停/重装）
          └─ 直接调用 failed()，不经过 catch 块 → 任务链终止 ❌
              （continue_on_failure 无效）
```

**另一个绕过 `continue_on_failure` 的路径**：`RunTaskJob:53-57`，当服务器 status 不为 null 时，直接调用 `$this->failed()`，这完全在 try-catch 之外，`continue_on_failure` 无法介入。

##### failed() 对任务链的影响

**文件位置**：`app/Jobs/Schedule/RunTaskJob.php:89-93`

```php
public function failed(?\Exception $exception = null)
{
    $this->markTaskNotQueued();    // is_queued = false
    $this->markScheduleComplete(); // is_processing = false, last_run_at = now()
}
```

调用 `failed()` 后：
1. 当前任务的 `is_queued` 被重置为 false
2. **整个 Schedule** 被标记为完成（`is_processing = false`）
3. 后续任务**不会被分发**，任务链彻底终止
4. Schedule 的 `next_run_at` 在 `ProcessScheduleService` 中已经更新为下一次时间，所以下次 Cron 周期仍会触发

#### 10.5 手动执行与队列执行的失败记录差异

定时任务有两条执行路径，它们的失败记录方式存在根本性差异。这个差异的根源在于 `ProcessScheduleService::handle()` 的第二个参数 `$now`，以及 Laravel 队列系统对 `dispatch()` 与 `dispatchNow()` 的不同处理方式。

##### 10.5.1 两条路径的入口

| 触发方式 | 入口 | `$now` 参数 | 分发方式 |
|----------|------|-------------|----------|
| 定时调度 | `ProcessRunnableCommand` → `ProcessScheduleService::handle($schedule)` | `false`（默认） | `dispatch($job->delay(...))` |
| 手动执行 | `ScheduleController::execute()` → `ProcessScheduleService::handle($schedule, true)` | `true` | `dispatchNow($job)` |

**文件位置**：
- 定时调度：`app/Console/Commands/Schedule/ProcessRunnableCommand.php:63`
- 手动执行：`app/Http/Controllers/Api/Client/Servers/ScheduleController.php:146`

##### 10.5.2 队列执行路径（定时调度，`now = false`）

```
ProcessRunnableCommand::processSchedule()
  │
  └─ ProcessScheduleService::handle($schedule)    // now = false
       │
       ├─ only_when_online 检查（若开启）
       │   ├─ Wings 返回 offline/stopping → $job->failed() → return
       │   ├─ 非 DaemonConnectionException → $job->failed($exception) → return
       │   └─ DaemonConnectionException → $job->failed() → return
       │
       └─ $this->dispatcher->dispatch($job->delay($task->time_offset))
            │
            │   ──── Job 进入队列，由 queue:work 消费 ────
            │
            └─ RunTaskJob::handle()
                 │
                 ├─ 无异常 → markTaskNotQueued() → queueNextTask()
                 │
                 ├─ continue_on_failure + DaemonConnectionException
                 │   → 吞掉异常 → markTaskNotQueued() → queueNextTask()
                 │
                 ├─ 抛出异常（非上述情况）
                 │   → Laravel 框架自动：
                 │     1. 调用 RunTaskJob::failed($exception)
                 │     2. 写入 failed_jobs 表
                 │     3. 触发 JobFailed 事件
                 │
                 └─ server.status != null
                     → $this->failed() → Laravel 框架不额外调用 failed()（已手动调用）
```

**关键特征**：
- Job 在队列 worker 进程中执行，与触发者（ProcessRunnableCommand）完全解耦
- `RunTaskJob` 未声明 `$tries` 属性，Laravel 默认 `tries=1`，即**不重试**
- 异常抛出后，Laravel 队列框架**自动**调用 `failed()` 并写入 `failed_jobs` 表
- `ProcessRunnableCommand` **无法感知** Job 执行结果（已 fire-and-forget）

##### 10.5.3 手动执行路径（手动触发，`now = true`）

```
ScheduleController::execute()
  │
  └─ ProcessScheduleService::handle($schedule, true)   // now = true
       │
       ├─ only_when_online 检查（若开启）
       │   ├─ Wings 返回 offline/stopping → $job->failed() → return
       │   ├─ 非 DaemonConnectionException → $job->failed($exception) → return
       │   └─ DaemonConnectionException → $job->failed() → return
       │
       └─ $this->dispatcher->dispatchNow($job)
            │
            │   ──── dispatchNow：同步执行，当前进程内直接运行 ────
            │
            └─ RunTaskJob::handle()
                 │
                 ├─ 无异常 → markTaskNotQueued() → queueNextTask()
                 │
                 ├─ continue_on_failure + DaemonConnectionException
                 │   → 吞掉异常 → markTaskNotQueued() → queueNextTask()
                 │
                 ├─ 抛出异常（非上述情况）
                 │   → Laravel 的 dispatchNow **不会**自动调用 failed()
                 │   → 异常冒泡到 ProcessScheduleService 的 catch 块：
                 │       catch (\Exception $exception) {
                 │           $job->failed($exception);   // 手动调用
                 │           throw $exception;           // 继续抛出
                 │       }
                 │   → 异常继续冒泡到 ScheduleController::execute()
                 │   → 最终由 Laravel 全局异常处理器返回 HTTP 500 错误给前端
                 │
                 └─ server.status != null
                     → $this->failed() → 已调用，无需再次调用
```

**关键特征**：
- `dispatchNow` 在当前请求进程内同步执行，不经过队列
- Laravel 的 `dispatchNow` **不会自动调用 `failed()`**，这是 Pterodactyl 在 `ProcessScheduleService:73-83` 手动补调的原因
  - **相关 Issue**：https://github.com/pterodactyl/panel/issues/2550
- 手动调用 `failed()` 后，异常仍被 `throw` 向上冒泡
- **不会**写入 `failed_jobs` 表（因为从未进入队列）
- 异常最终到达 HTTP 层，前端可以收到错误响应

##### 10.5.4 `failed()` 的所有调用时机

`RunTaskJob::failed()` 在以下场景被调用，但调用者和上下文各不相同：

| 调用场景 | 调用者 | 执行路径 | `failed_jobs` 写入 | HTTP 错误响应 |
|----------|--------|----------|-------------------|--------------|
| only_when_online + offline/stopping | `ProcessScheduleService:53` | 两种路径都有 | ❌ 不写入 | ❌ 无（静默返回） |
| only_when_online + 非 DaemonConnectionException | `ProcessScheduleService:62` | 两种路径都有 | ❌ 不写入 | ❌ 无（静默返回） |
| only_when_online + DaemonConnectionException | `ProcessScheduleService:64` | 两种路径都有 | ❌ 不写入 | ❌ 无（静默返回） |
| server.status != null | `RunTaskJob:54` | 两种路径都有 | 队列路径：✅ 写入 | 手动路径：❌ |
| 任务执行抛异常（队列） | Laravel 框架自动 | 仅队列路径 | ✅ 写入 | ❌ 无（已解耦） |
| 任务执行抛异常（手动） | `ProcessScheduleService:80` | 仅手动路径 | ❌ 不写入 | ✅ HTTP 500 |
| schedule 非激活 | `RunTaskJob:42-43` | 两种路径都有 | ❌ 不写入（不算失败） | ❌ 无（正常结束） |

**关键发现**：
1. `ProcessScheduleService` 中的 3 处 `$job->failed()` 调用发生在 **Job 分发之前**，此时 Job 尚未进入队列，因此 `failed_jobs` 表不会被写入
2. `RunTaskJob:54` 的 `$this->failed()` 调用发生在 Job **已分发之后**，但 `failed()` 是手动调用而非框架触发——队列路径下框架还会再次尝试调用 `failed()`（幂等，无副作用）
3. **只有队列路径中由 Laravel 框架自动触发的 `failed()` 才会写入 `failed_jobs` 表**
4. 手动路径中的 `failed()` 永远不会导致 `failed_jobs` 写入

##### 10.5.5 `failed_jobs` 表的使用范围

**配置**（`config/queue.php:87-91`）：
```php
'failed' => [
    'driver' => env('QUEUE_FAILED_DRIVER', 'database-uuids'),
    'database' => env('DB_CONNECTION', 'mysql'),
    'table' => 'failed_jobs',
],
```

**写入条件**（Laravel 框架行为）：
1. Job 必须**通过队列分发**（`dispatch()`），而非同步执行（`dispatchNow()`）
2. Job 执行过程中抛出异常
3. Job 的重试次数耗尽（`RunTaskJob` 未声明 `$tries`，默认 `tries=1`，首次失败即写入）

**`RunTaskJob` 的 `failed_jobs` 记录内容**：
- `queue`: `standard`
- `payload`: 序列化的 `RunTaskJob` 实例（含 Task 模型快照）
- `exception`: 异常的完整堆栈信息
- `uuid`: 唯一标识符

**不写入 `failed_jobs` 的场景**：
1. 手动执行路径（`dispatchNow`）
2. `ProcessScheduleService` 中分发前的 `failed()` 调用
3. `continue_on_failure` 生效时（异常被吞掉，不抛出）
4. schedule 非激活导致的提前返回

##### 10.5.6 两种路径的监控方式对比

| 监控维度 | 队列执行（定时调度） | 手动执行 |
|----------|---------------------|----------|
| **失败记录** | `failed_jobs` 表 + Laravel 日志 | HTTP 500 响应 + Laravel 日志 |
| **查询失败** | `php artisan queue:failed` 或查询 `failed_jobs` 表 | 查看 Web 服务器访问日志 / 浏览器控制台 |
| **重试失败任务** | `php artisan queue:retry {id}` | 不适用（无队列记录，需手动重新触发） |
| **前端反馈** | 无（fire-and-forget） | HTTP 错误响应（用户可立即感知） |
| **ProcessRunnableCommand 日志** | ✅ 仅记录 `ProcessScheduleService::handle()` 阶段的异常 | 不涉及 |
| **Activity Log** | ❌ 无定时任务失败记录 | ❌ 无定时任务失败记录 |
| **Laravel 日志** | ✅ Job 执行异常由框架写入 | ✅ 全局异常处理器写入 |

**监控盲区**：

1. **队列路径的静默失败**：`ProcessScheduleService` 中 `only_when_online` 检查失败时，直接调用 `$job->failed()` 然后 `return`，没有任何日志记录。这种"本应执行但被跳过"的情况在系统中不留痕迹。

2. **队列路径的 Job 执行失败**：虽然写入 `failed_jobs` 表，但**没有 Activity Log 记录**，也没有通知。管理员必须主动查询 `failed_jobs` 表才能发现。

3. **手动路径的 `failed()` 调用**：`ProcessScheduleService:80` 手动调用 `$job->failed($exception)` 后又 `throw $exception`，`failed()` 方法本身不记录任何日志，异常信息仅通过后续的 throw 传递到 HTTP 层。

---

## 三、关键数据结构

### schedules 表字段
| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | int | 主键 |
| `server_id` | int | 关联服务器 ID |
| `name` | varchar | 任务名称 |
| `cron_minute` | varchar | Cron 分钟 |
| `cron_hour` | varchar | Cron 小时 |
| `cron_day_of_month` | varchar | Cron 日期 |
| `cron_month` | varchar | Cron 月份 |
| `cron_day_of_week` | varchar | Cron 星期 |
| `is_active` | tinyint | 是否激活 |
| `is_processing` | tinyint | 是否处理中 |
| `only_when_online` | tinyint | 仅服务器在线时执行 |
| `last_run_at` | timestamp | 上次执行时间 |
| `next_run_at` | timestamp | 下次执行时间 |

### tasks 表字段
| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | int | 主键 |
| `schedule_id` | int | 关联调度 ID |
| `sequence_id` | int | 执行顺序 |
| `action` | varchar | 任务类型 |
| `payload` | text | 任务参数 |
| `time_offset` | int | 延迟秒数 |
| `is_queued` | tinyint | 是否已入队 |
| `continue_on_failure` | tinyint | 失败后继续 |

---

## 四、执行时序图

```
1. Laravel Cron (每分钟)
   ↓
2. ProcessRunnableCommand::handle()
   ├─ 查询符合条件的 schedules
   │  └─ is_active=true, is_processing=false, next_run_at<=NOW()
   └─ 对每个 schedule 调用 processSchedule()
      ↓
3. ProcessScheduleService::handle()
   ├─ 事务更新 schedule: is_processing=true, next_run_at=下一次时间
   ├─ 更新第一个 task: is_queued=true
   └─ 分发 RunTaskJob 到队列（延迟 time_offset）
      ↓
4. RunTaskJob::handle() (队列中执行)
   ├─ 检查 schedule 状态和服务器状态
   ├─ 调用 Wings API 执行具体操作
   │  ├─ power: POST /api/servers/{uuid}/power
   │  ├─ command: POST /api/servers/{uuid}/commands
   │  └─ backup: 调用备份服务
   ├─ 成功：markTaskNotQueued() + queueNextTask()
   └─ 失败：failed() → markScheduleComplete()
      ↓
5. queueNextTask() → 分发下一个 RunTaskJob
   ...（循环直到所有任务完成）
      ↓
6. markScheduleComplete()
   └─ schedule: is_processing=false, last_run_at=NOW()
```

---

## 五、设计特点与注意事项

### 优点
1. **分布式执行**：通过队列任务实现，支持横向扩展
2. **防重叠**：`is_processing` 标志 + `withoutOverlapping()` 双重保障
3. **任务链**：支持多任务按顺序执行，每个任务可设置延迟
4. **有限容错**：`continue_on_failure` 允许在 Wings 不可达时继续执行后续任务，但仅限于网络通信失败这一种异常类型
5. **状态一致性**：关键操作使用数据库事务保证

### 注意事项
1. **精度限制**：每分钟检查一次，不支持秒级调度
2. **结果回调差异**：
   - 电源/命令操作：无结果回调，Wings 执行结果不会回写到 Panel
   - 备份操作：有独立的回调接口（`POST /api/remote/backups/{uuid}/status`），备份完成后 Wings 回写状态
3. **备份异步性**：`DaemonBackupRepository::backup()` 只向 Wings 发起 HTTP POST 触发备份，不等待备份完成；`RunTaskJob` 触发成功后立即继续任务链
4. **`continue_on_failure` 作用域极窄**：仅对 `DaemonConnectionException`（Wings 网络通信失败）生效；所有业务逻辑异常（备份超限、频率超限、锁定等）无论此标志如何都会终止任务链
5. **服务器异常状态绕过 continue_on_failure**：当 `server.status != null`（被暂停/重装）时，直接调用 `failed()`，不经过 try-catch，`continue_on_failure` 无法介入
6. **无内置失败通知**：需要自行扩展通知机制（可通过 ActivityLogged 事件或 Queue::failing 事件）
7. **队列依赖**：必须运行队列 worker（`php artisan queue:work`）
8. **备份悬挂风险**：若 Wings 在接受备份指令后崩溃，Backup 记录会处于 `is_successful=false, completed_at=null` 的悬挂状态，需 `PruneOrphanedBackupsCommand` 清理
9. **任务上限**：每个 Schedule 最多 10 个任务（可通过 `PTERODACTYL_PER_SCHEDULE_TASK_LIMIT` 配置）
10. **备份限制**：服务器 `backup_limit = 0` 时不能创建备份任务
11. **手动执行与队列执行的失败记录不对称**：
    - 队列路径（定时调度）：失败写入 `failed_jobs` 表，但前端无感知；可通过 `queue:retry` 重试
    - 手动路径（用户触发）：失败不写 `failed_jobs`，但前端收到 HTTP 500 错误；无法通过 `queue:retry` 重试
12. **`dispatchNow` 不会自动调用 `failed()`**：手动执行路径依赖 `ProcessScheduleService:73-83` 的显式 `failed()` 补调（修复 issue #2550）
13. **分发前的 `failed()` 调用无 `failed_jobs` 记录**：`ProcessScheduleService` 中 `only_when_online` 检查失败时，Job 尚未分发，`failed()` 调用不会写入 `failed_jobs` 表，也不记录日志，形成监控盲区
