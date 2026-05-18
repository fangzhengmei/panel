# Panel 排程器任务派发流程分析 - 第三轮（队列重试语义精确对齐）

## 修正说明

前两轮分析中，队列重试语义与实际代码及运行配置存在偏差。本次从以下几个维度精确对齐：

1. `RunTaskJob` 未定义 `$tries` 时的代码层默认行为
2. 容器启动参数 `--tries=3` 对实际尝试次数的影响
3. `failed_jobs` 在异步队列场景下的精确写入时机
4. `dispatchNow` 手动触发失败不进入 `failed_jobs` 的底层原因

---

## 1. 运行时配置全景

### 1.1 队列 Worker 启动参数

**文件**：`.github/docker/supervisord.conf:28`

```ini
[program:queue-worker]
command=/usr/local/bin/php /app/artisan queue:work --queue=high,standard,low --sleep=3 --tries=3
```

**实际生效参数**：
| 参数 | 值 | 说明 |
|------|-----|------|
| `--queue` | high,standard,low | 按优先级处理三个队列 |
| `--sleep` | 3 | 队列空时休眠 3 秒再轮询 |
| `--tries` | 3 | **全局默认最大尝试次数** |

### 1.2 Job 类对比

| Job 类 | 是否定义 `$tries` | `$tries` 值 | 是否定义 `$maxExceptions` |
|--------|-------------------|-------------|--------------------------|
| `RunTaskJob` | ❌ 未定义 | - | ❌ 未定义 |
| `RevokeSftpAccessJob` | ✅ 定义 | `3` | ✅ 定义为 `1` |

---

## 2. Laravel Queue 重试机制精确解析

### 2.1 最大尝试次数的优先级规则

Laravel 队列中，决定任务最大尝试次数的优先级（从高到低）：

```
┌─────────────────────────────────────────────────────────────────┐
│  优先级 1: Job 类的 maxTries() 方法（动态计算）                  │
└─────────────────────────────────┬───────────────────────────────┘
                                  ▼
┌─────────────────────────────────────────────────────────────────┐
│  优先级 2: Job 类的 $tries 属性（静态定义）                      │
│  如：public int $tries = 3;                                     │
└─────────────────────────────────┬───────────────────────────────┘
                                  ▼
┌─────────────────────────────────────────────────────────────────┐
│  优先级 3: queue:work 命令的 --tries 参数                        │
│  如：--tries=3                                                  │
└─────────────────────────────────┬───────────────────────────────┘
                                  ▼
┌─────────────────────────────────────────────────────────────────┐
│  优先级 4: Laravel 框架默认值 = 1                               │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 RunTaskJob 的实际最大尝试次数

**结论**：`RunTaskJob` 实际最大尝试次数 = **3 次**

推导过程：
1. `RunTaskJob` 类中没有定义 `$tries` 属性
2. `RunTaskJob` 类中没有定义 `maxTries()` 方法
3. 所以使用命令行参数 `--tries=3`
4. 任务最多会被尝试 **3 次**（第 1 次执行 + 最多 2 次重试）

### 2.3 尝试次数计数逻辑

Laravel 内部维护一个 `attempts` 计数器：

```
第 1 次执行：attempts = 1
  ↓ 失败
  检查：attempts < maxTries (1 < 3) → 是，可以重试
  ↓
第 2 次执行：attempts = 2
  ↓ 失败
  检查：attempts < maxTries (2 < 3) → 是，可以重试
  ↓
第 3 次执行：attempts = 3
  ↓ 失败
  检查：attempts < maxTries (3 < 3) → 否，不再重试
  ↓
标记为最终失败，写入 failed_jobs 表
```

**重要**：`--tries=3` 意味着 **总共执行 3 次**，不是"重试 3 次"。

---

## 3. failed_jobs 写入时机精确分析

### 3.1 异步队列场景（dispatch + queue:work）

**完整失败处理流程**：

```
任务被 Worker 取出执行
    ↓
┌─ RunTaskJob::handle() 执行 ─────────────────────────────────────┐
│  try {                                                           │
│      执行任务逻辑                                                 │
│  } catch (\Exception $exception) {                               │
│      检查 continue_on_failure 条件                                │
│      不满足 → 重新抛出异常                                        │
│  }                                                               │
└─────────────────────────────────┬───────────────────────────────┘
                                  ↓ 异常抛出
┌─ Laravel Queue Worker 捕获异常 ─────────────────────────────────┐
│                                                                 │
│  1. 检查是否可以重试：                                           │
│     if ($job->attempts() < $maxTries) {                         │
│         // 可以重试                                              │
│         $job->release();  // 放回队列，稍后重试                  │
│         // 不写入 failed_jobs                                    │
│     } else {                                                     │
│         // 已达最大尝试次数，标记为最终失败                       │
│         调用 $job->failed($exception);                           │
│         写入 failed_jobs 表                                      │
│         从队列中删除任务                                          │
│     }                                                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 failed_jobs 写入的精确条件

**只有同时满足以下所有条件时，才会写入 failed_jobs 表**：

1. ✅ 任务是通过异步队列执行的（经过 queue worker）
2. ✅ 任务执行过程中抛出了未被捕获的异常
3. ✅ 任务的尝试次数 **已达到** 最大尝试次数（`attempts() >= maxTries`）
4. ❌ 任务没有被手动 `release()` 放回队列

**每次重试时**：
- 不会写入 `failed_jobs`
- 会调用 `$job->release()` 将任务放回队列
- `attempts` 计数器 +1
- 等待下次被取出执行

**最终失败时**（第 3 次失败）：
- 调用 `RunTaskJob::failed()` 进行状态清理
- 将任务完整信息（包括异常堆栈）写入 `failed_jobs` 表
- 从队列中永久删除任务

### 3.3 failed_jobs 表记录的内容

写入 `failed_jobs` 表的字段包括：

| 字段 | 内容 |
|------|------|
| `id` | 自增 ID |
| `uuid` | 唯一标识（Laravel 11 使用 UUID） |
| `connection` | 队列连接名（如 `redis`） |
| `queue` | 队列名（如 `standard`） |
| `payload` | 序列化的完整 Job 对象（包括 Task 模型、manualRun 标志） |
| `exception` | 完整的异常堆栈信息 |
| `failed_at` | 失败时间戳 |

---

## 4. dispatchNow 不写入 failed_jobs 的底层原因

### 4.1 dispatch vs dispatchNow 的执行路径对比

```
┌─────────────────────────────────────────────────────────────────────┐
│                      dispatch() 异步路径                            │
├─────────────────────────────────────────────────────────────────────┤
│  1. 将 Job 序列化后放入 Redis 队列                                  │
│  2. 立即返回，不等待执行                                            │
│  3. 队列 Worker 进程独立取出任务执行                                │
│  4. Worker 进程中包含完整的异常处理逻辑：                            │
│     - 计数 attempts                                                │
│     - 判断是否可以重试                                              │
│     - 调用 failed() 方法                                           │
│     - 写入 failed_jobs 表                                          │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                    dispatchNow() 同步路径                           │
├─────────────────────────────────────────────────────────────────────┤
│  1. 直接在当前 PHP 进程中调用 $job->handle()                        │
│  2. 绕过整个队列系统                                                │
│  3. 没有 attempts 计数逻辑                                          │
│  4. 没有重试判断逻辑                                                │
│  5. 异常直接向上抛出，不经过 Worker 的失败处理流程                  │
│  6. **不会自动调用 failed() 方法**（Laravel 已知行为）              │
│  7. **不会写入 failed_jobs 表**                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 4.2 代码中的特殊处理

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

**代码注释明确说明了这个 Laravel 行为差异**：
- `dispatchNow()` 不会自动调用 `failed()`
- 必须手动调用 `$job->failed($exception)` 来确保状态清理
- 但即使手动调用了 `failed()`，也 **不会触发 failed_jobs 写入**
- 因为写入逻辑在 Queue Worker 中，不在当前执行路径

### 4.3 为什么 dispatchNow 不写入 failed_jobs

**根本原因**：`failed_jobs` 写入是 **队列 Worker 进程** 的职责，不是 Job 类本身的职责。

```
写入 failed_jobs 的逻辑在：
  vendor/laravel/framework/src/Illuminate/Queue/Worker.php
    └─ failJob() 方法
        ├─ 调用 $job->failed()
        └─ 写入 failed_jobs 表

dispatchNow 执行路径：
  app/Services/Schedules/ProcessScheduleService.php
    └─ 直接调用 $job->handle()
        └─ 异常抛出 → 手动调用 $job->failed()
            └─ 只做状态清理，不调用 Worker::failJob()
                └─ 不写入 failed_jobs
```

---

## 5. RunTaskJob 失败场景的完整生命周期

### 5.1 场景 1：定时触发（异步队列）+ 首次失败

```
1. ProcessRunnableCommand 触发
   ↓
2. ProcessScheduleService::handle()
   事务更新状态，dispatch RunTaskJob 到队列
   ↓
3. Queue Worker 取出任务（attempts = 1）
   ↓
4. RunTaskJob::handle() 执行，抛出异常
   ↓
5. Worker 判断：1 < 3 → 可以重试
   ↓
6. $job->release()，放回队列
   ↓
7. 不写入 failed_jobs
   ↓
8. 等待下次被取出执行（attempts 将变为 2）
```

### 5.2 场景 2：定时触发（异步队列）+ 第三次失败

```
1. Queue Worker 取出任务（attempts = 3）
   ↓
2. RunTaskJob::handle() 执行，抛出异常
   ↓
3. Worker 判断：3 < 3 → 否，不可重试
   ↓
4. 调用 RunTaskJob::failed()
   - is_queued = false
   - is_processing = false
   - last_run_at = now
   ↓
5. 写入 failed_jobs 表
   - 包含完整异常堆栈
   ↓
6. 从队列中删除任务
   ↓
7. 排程结束，等待下一次 Cron 触发
```

### 5.3 场景 3：手动触发（dispatchNow）+ 失败

```
1. 用户调用 /execute 接口
   ↓
2. ProcessScheduleService::handle($schedule, $now = true)
   事务更新状态
   ↓
3. dispatchNow($job)
   ↓
4. RunTaskJob::handle() 执行，抛出异常
   ↓
5. ProcessScheduleService catch 异常
   ↓
6. 手动调用 $job->failed()
   - is_queued = false
   - is_processing = false
   - last_run_at = now
   ↓
7. 重新抛出异常
   ↓
8. 返回 HTTP 500 给用户
   ↓
9. **不写入 failed_jobs 表**
```

---

## 6. 关键结论汇总表

| 问题 | 结论 | 依据 |
|------|------|------|
| RunTaskJob 未定义 $tries 时的默认尝试次数 | **3 次** | 命令行 `--tries=3` 生效 |
| --tries=3 表示重试几次 | **总共执行 3 次**（1 次初始 + 最多 2 次重试） | Laravel 优先级规则 |
| 异步队列失败时何时写入 failed_jobs | **达到最大尝试次数后** | Worker::failJob() 逻辑 |
| 重试过程中是否写入 failed_jobs | **不写入** | 只有最终失败才写入 |
| dispatchNow 失败是否写入 failed_jobs | **不写入** | 绕过了 Worker 进程的失败处理流程 |
| dispatchNow 失败时是否调用 failed() | **需要手动调用** | 代码注释明确说明，见 issue #2550 |
| continue_on_failure 会阻止写入 failed_jobs 吗 | **会** | 异常被捕获，不会抛出给 Worker |

---

## 7. 代码实现中的隐式行为

### 7.1 continue_on_failure 的副作用

如果任务配置了 `continue_on_failure = true` 且异常是 `DaemonConnectionException`：

```php
catch (\Exception $exception) {
    if (!($this->task->continue_on_failure && $exception instanceof DaemonConnectionException)) {
        throw $exception;
    }
    // 不抛出异常，继续执行
}
```

**结果**：
- 异常被静默捕获，不会抛出给 Worker
- Worker 认为任务执行成功
- **不会重试**
- **不会写入 failed_jobs**
- 继续执行后续任务

这是一个设计决策：网络抖动导致的连接异常，不应该阻止整个排程的执行。

### 7.2 retry_after 与 $tries 的交互

| 机制 | 触发条件 | 行为 | 是否写入 failed_jobs |
|------|---------|------|---------------------|
| `$tries=3` | 任务抛出异常 | 最多尝试 3 次，最终失败后写入 | 最终失败时写入 |
| `retry_after=90` | Worker 崩溃，任务 90 秒未完成 | 重新放回队列，其他 Worker 尝试 | 不会直接触发写入 |

**如果 retry_after 触发了重试**：
- 这会增加实际执行次数（可能超过 $tries=3）
- 因为 retry_after 是从队列层面重新放入，不是 Job 层面的重试
- attempts 计数器会被重置

这是一个潜在的问题：如果任务执行时间超过 90 秒但实际在正常运行，可能会被重复执行。

---

## 8. 最终澄清的认知修正

| 认知点 | 错误认知 | 正确认知（代码事实） |
|--------|---------|---------------------|
| $tries 未定义时 | 默认为 1，只执行 1 次 | 实际为 3 次（来自 --tries=3） |
| --tries=3 | 重试 3 次，共执行 4 次 | 总共执行 3 次（初始 + 2 次重试） |
| 每次失败都写 failed_jobs | 是 | 否，只有最终失败才写 |
| dispatchNow 失败写 failed_jobs | 是 | 否，绕过了 Worker 流程 |
| failed() 方法负责写 failed_jobs | 是 | 否，只做状态清理，写入是 Worker 的职责 |
