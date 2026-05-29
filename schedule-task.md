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

### 8. 失败通知机制

**重要发现**：当前代码中 **没有内置的失败通知机制**（如邮件、Webhook、Discord 等）。

#### 现有失败处理方式：
1. **日志记录**：异常信息写入 Laravel 日志
2. **控制台输出**：命令行执行时输出错误信息
3. **任务终止**：失败后默认终止后续任务执行（除非设置 `continue_on_failure`）

#### 可能的扩展点：
- 可通过监听 Laravel 的 `JobFailed` 事件实现自定义通知
- 可通过修改 `RunTaskJob::failed()` 方法添加通知逻辑
- 可通过 Activity Log 系统（`Activity` Facade）扩展通知

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
4. **容错**：支持单个任务失败后继续执行后续任务
5. **状态一致性**：关键操作使用数据库事务保证

### 注意事项
1. **精度限制**：每分钟检查一次，不支持秒级调度
2. **无结果回调**：Wings 执行结果（如命令输出、备份状态）不会回写到 Panel
3. **无失败通知**：需要自行扩展通知机制
4. **队列依赖**：必须运行队列 worker（`php artisan queue:work`）
5. **时间同步**：服务器时间需准确，否则 Cron 计算会有偏差
