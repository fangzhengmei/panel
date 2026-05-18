# Panel 排程器任务派发流程分析 - 第二轮修正

## 修正说明

第一轮分析中存在几处与实际代码不符的地方，本次修正重点澄清：
1. `retry_after` 与失败任务记录的实际关系
2. `tasks_log` 表在当前实现中的真实地位
3. 手动触发 `execute` 接口与 `is_active`、`manualRun` 的完整交互分支

---

## 1. tasks_log 的真实状态

### 1.1 代码审计结果

通过全代码库搜索 `TaskLog` 或 `tasks_log`，结果显示：

| 位置 | 用途 | 实际操作 |
|------|------|----------|
| `app/Models/TaskLog.php` | 模型定义 | 仅定义，无业务代码引用 |
| `database/migrations/2016_02_27_163447_add_tasks_log_table.php` | 数据库迁移 | 2016 年创建，后续无修改 |
| `database/schema/mysql-schema.sql` | 数据库schema | 表结构定义 |
| 其他业务代码 | 无 | **零引用** |

### 1.2 结论

**`tasks_log` 表完全没有进入主执行链，是一个遗留的未使用功能。**

- 创建于 2016 年的早期版本
- 当前排程系统（2017年重构后）没有任何代码写入或读取该表
- 表结构包含 `response` 字段，设计意图是记录任务执行结果，但从未实现
- 实际的任务执行状态完全通过 `tasks.is_queued` 和 `schedules.is_processing` 字段追踪

---

## 2. 手动触发 execute 接口的完整分支

### 2.1 接口入口

**文件**：`app/Http/Controllers/Api/Client/Servers/ScheduleController.php:144-151`

```php
public function execute(TriggerScheduleRequest $request, Server $server, Schedule $schedule): JsonResponse
{
    $this->service->handle($schedule, true);  // 第二个参数 $now = true
    Activity::event('server:schedule.execute')->subject($schedule)->log();
    return new JsonResponse([], JsonResponse::HTTP_ACCEPTED);
}
```

### 2.2 $now 参数的传递链

```
ScheduleController::execute()
    ↓ $now = true
ProcessScheduleService::handle($schedule, $now = true)
    ↓ $manualRun = $now
RunTaskJob::__construct($task, $manualRun = true)
```

### 2.3 is_active 与 manualRun 的交互逻辑

**文件**：`app/Jobs/Schedule/RunTaskJob.php:41-46`

```php
// Do not process a task that is not set to active, unless it's been manually triggered.
if (!$this->task->schedule->is_active && !$this->manualRun) {
    $this->markTaskNotQueued();
    $this->markScheduleComplete();
    return;
}
```

**真值表**：

| is_active | manualRun | 执行结果 | 说明 |
|-----------|-----------|----------|------|
| true | false | 执行 | 正常定时触发 |
| true | true | 执行 | 手动触发已启用的排程 |
| false | false | 跳过 | 已禁用的排程不会被定时触发 |
| false | true | **执行** | 手动触发可以绕过 is_active 限制 |

### 2.4 关键设计意图

手动触发的设计目的是：
- 允许用户立即测试排程，无需等待 Cron 时间
- 即使排程被禁用（`is_active = false`），也可以手动执行进行调试
- 这是一个"调试/紧急执行"通道，不受正常启用状态限制

---

## 3. 队列异常后的重试行为 - 完整澄清

### 3.1 Laravel Queue 重试机制基础

RunTaskJob 的类定义：

```php
class RunTaskJob implements ShouldQueue
{
    use Queueable;
    use DispatchesJobs;
    use SerializesModels;

    // 注意：没有定义 $tries 属性
    // 注意：没有定义 retryUntil() 方法
    // 注意：没有定义 backoff() 方法
}
```

**Laravel Queue 默认行为**（没有 `$tries` 时）：
- 任务只会被尝试执行 **1 次**
- 抛出异常即视为失败
- 不会自动重试

### 3.2 retry_after 的真实作用

**配置文件**：`config/queue.php:64-71`

```php
'redis' => [
    'driver' => 'redis',
    'retry_after' => (int) env('REDIS_QUEUE_RETRY_AFTER', 90),
],
```

**`retry_after` 不是业务重试机制，而是"僵尸任务"保护机制：**

1. 当队列 worker 取出一个任务时，会在 Redis 中设置一个 "保留" 标记
2. 如果 worker 进程崩溃（例如被 OOM killer 杀死、服务器断电），这个任务会一直处于"处理中"状态
3. `retry_after` 定义了一个超时时间：**90 秒后如果任务还没标记完成，就认为 worker 已经死亡**
4. 此时队列会将该任务**重新放回队列**，让其他 worker 尝试执行

**这是一种异常恢复机制，不是设计上的重试逻辑。**

### 3.3 异常抛出与失败记录的完整流程

```
任务开始执行
    ↓
RunTaskJob::handle() 被调用
    ↓
┌─ 执行任务逻辑 ───────────────────────────────────────────────────────┐
│  try {                                                              │
│      switch ($this->task->action) { ... }                           │
│  } catch (\Exception $exception) {                                  │
│      // 检查是否可以忽略错误继续                                     │
│      if (!($this->task->continue_on_failure                         │
│           && $exception instanceof DaemonConnectionException)) {    │
│          throw $exception;  // ← 重新抛出异常                       │
│      }                                                              │
│      // 否则，忽略错误继续执行后续任务                               │
│  }                                                                  │
└─────────────────────────────────────────────────────────────────────┘
    ↓ （如果异常被重新抛出）
Laravel Queue 捕获异常
    ↓
1. 调用 RunTaskJob::failed($exception) 方法
    ↓
2. 将任务记录到 failed_jobs 表
    ├─ 字段：connection、queue、payload、exception、failed_at
    └─ payload 包含序列化的任务对象（Task 模型、manualRun 标志等）
    ↓
3. **不会重试**（因为没有设置 $tries）
    ↓
任务结束
```

### 3.4 failed() 方法的实际作用

**文件**：`app/Jobs/Schedule/RunTaskJob.php:89-93`

```php
public function failed(?\Exception $exception = null)
{
    $this->markTaskNotQueued();    // task.is_queued = false
    $this->markScheduleComplete(); // schedule.is_processing = false, 更新 last_run_at
}
```

**重要**：`failed()` 方法只做状态清理，不做：
- 不记录异常到 tasks_log（该表未使用）
- 不触发重试
- 不通知用户
- 只确保排程不会卡在 `is_processing = true` 状态

### 3.5 continue_on_failure 的边界条件

**文件**：`app/Jobs/Schedule/RunTaskJob.php:74-80`

```php
catch (\Exception $exception) {
    if (!($this->task->continue_on_failure && $exception instanceof DaemonConnectionException)) {
        throw $exception;
    }
    // 否则，静默继续
}
```

**可继续执行的充要条件**（必须同时满足）：
1. ✅ `task.continue_on_failure = true`
2. ✅ 异常类型 **精确** 是 `DaemonConnectionException`
3. ❌ 其他异常（如 `InvalidArgumentException`、`TooManyBackupsException` 等）都会导致失败

**设计意图**：
- 只容忍网络连接问题（守护进程暂时不可达）
- 业务逻辑错误必须终止执行，避免级联错误

---

## 4. 失败任务的生命周期

### 4.1 正常定时触发的失败流程

```
1. ProcessRunnableCommand 每分钟运行
   查询：is_active=true AND is_processing=false AND next_run_at<=NOW()
    ↓
2. ProcessScheduleService::handle()
   事务：is_processing=true, next_run_at=下次时间, task.is_queued=true
    ↓
3. 派发 RunTaskJob 到队列
    ↓
4. 任务执行失败，抛出异常
    ↓
5. Laravel 调用 failed() 方法
   is_queued=false, is_processing=false, last_run_at=now
    ↓
6. 写入 failed_jobs 表
    ↓
7. 排程等待下一次 Cron 触发
   （不会自动重试本次失败的任务）
```

### 4.2 手动触发的失败流程

```
1. 用户调用 POST /api/client/servers/{server}/schedules/{schedule}/execute
    ↓
2. ProcessScheduleService::handle($schedule, $now = true)
   事务：is_processing=true, next_run_at=下次时间, task.is_queued=true
    ↓
3. 同步执行 dispatchNow($job) （不经过队列）
    ↓
4. 任务执行失败，抛出异常
    ↓
5. ProcessScheduleService 手动调用 $job->failed()
   （因为 dispatchNow 不会自动调用 failed()）
    ↓
6. 异常继续向上抛出，返回 HTTP 500 给用户
    ↓
7. **不会**写入 failed_jobs 表
   （因为没有经过队列 worker，Laravel 的失败记录逻辑不触发）
```

**关键区别**：
- 定时触发（队列执行）：失败会写入 `failed_jobs` 表
- 手动触发（同步执行）：失败直接返回异常，**不写入** `failed_jobs` 表

---

## 5. retry_after 与 failed_jobs 的关系图

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Laravel Queue Worker                         │
└─────────────────────────────────┬───────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    从 Redis 队列取出 RunTaskJob                     │
│  - 在 Redis 设置 "reserved" 标记                                    │
│  - 启动计时器（关联 retry_after = 90秒）                            │
└─────────────────────────────────┬───────────────────────────────────┘
                                  │
          ┌───────────────────────┴───────────────────────┐
          ▼                                               ▼
┌─────────────────────┐                         ┌─────────────────────┐
│  任务在 90 秒内完成 │                         │  90 秒后任务仍未完成 │
│  - 正常结束或抛出异常 │                         │  - Worker 可能崩溃了  │
└───────────┬─────────┘                         └───────────┬─────────┘
            ▼                                               ▼
┌─────────────────────┐                         ┌─────────────────────┐
│  清除 reserved 标记 │                         │  重新放回队列        │
└───────────┬─────────┘                         │  （其他 worker 尝试）│
            ▼                                   └───────────┬─────────┘
┌─────────────────────┐                                             ▼
│  如果抛出异常：      │                                   ┌─────────────────────┐
│  - 调用 failed()    │                                   │  这是一种异常恢复    │
│  - 写入 failed_jobs │                                   │  不是业务重试逻辑    │
│  - 任务结束         │                                   └─────────────────────┘
└─────────────────────┘
```

---

## 6. 修正后的关键结论汇总

| 问题 | 第一轮分析 | 第二轮修正（实际代码） |
|------|-----------|-----------------------|
| tasks_log 用途 | 任务执行日志 | **未使用，遗留表** |
| retry_after 作用 | 失败后 90 秒自动重试 | **僵尸任务保护，不是业务重试** |
| 失败后是否自动重试 | 不会自动重试 | **确认：不会自动重试（无 $tries）** |
| 手动触发受 is_active 限制 | 未详细说明 | **不受限制，manualRun=true 可绕过** |
| 手动触发失败是否写入 failed_jobs | 未说明 | **不写入，直接返回异常** |
| continue_on_failure 适用范围 | 网络异常时继续 | **仅 DaemonConnectionException，精确匹配** |

---

## 7. 代码实现中的注意点

### 7.1 dispatchNow 的特殊处理

**文件**：`app/Services/Schedules/ProcessScheduleService.php:72-84`

```php
} else {
    // When using dispatchNow the RunTaskJob::failed() function is not called automatically
    // so we need to manually trigger it and then continue with the exception throw.
    //
    // @see https://github.com/pterodactyl/panel/issues/2550
    try {
        $this->dispatcher->dispatchNow($job);
    } catch (\Exception $exception) {
        $job->failed($exception);  // 手动调用 failed()
        throw $exception;
    }
}
```

这是一个已知的 Laravel 行为差异：
- `dispatch()`：队列异步执行，失败时 Laravel 自动调用 `failed()`
- `dispatchNow()`：同步执行，失败时 **不会自动调用** `failed()`，必须手动调用

### 7.2 only_when_online 检查中的异常处理

**文件**：`app/Services/Schedules/ProcessScheduleService.php:57-67`

```php
} catch (\Exception $exception) {
    if (!$exception instanceof DaemonConnectionException) {
        // 非连接异常，调用 failed() 记录异常
        $job->failed($exception);
    }
    // 无论什么异常，都再次调用 failed() 清理状态
    $job->failed();
    return;
}
```

**注意**：这里有两次 `failed()` 调用。第一次带异常参数（如果是非连接异常），第二次不带。但 `failed()` 方法实际上忽略了异常参数，只做状态清理。

---

## 8. 潜在问题（基于实际代码）

1. **`tasks_log` 表冗余**：占用数据库空间，建议清理或实现
2. **无失败通知**：任务失败后只有 `failed_jobs` 表记录，用户无感知
3. **手动触发失败不记录**：手动触发的失败不会进入 `failed_jobs`，问题排查困难
4. **`retry_after` 可能导致重复执行**：如果任务执行超过 90 秒但实际在运行，会被重复执行
5. **无幂等性保证**：如果 `retry_after` 触发了重试，电源/命令任务可能被执行多次
