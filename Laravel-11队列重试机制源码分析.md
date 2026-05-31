# Laravel 11 队列重试机制源码分析

> 本文档针对 Laravel 11 队列系统的 `tries` 默认值、`queue:work` 参数生效机制，以及结合本项目 supervisord 配置的两个 Job 重试边界和上限处理进行源码级分析。

---

## 一、Laravel 11 队列默认尝试次数

### 1.1 Laravel 官方文档说明

根据 Laravel 11 官方文档：

> "If a job exceeds its maximum number of attempts, it will be considered a 'failed' job."
>
> "One approach to specifying the maximum number of times a job may be attempted is via the `--tries` switch on the Artisan command line."

**关键结论**：Laravel 的 `queue:work` 命令**不强制**设置 `--tries` 参数，其默认值取决于 Laravel 版本。

### 1.2 Laravel 11 `WorkCommand` 源码

从 Laravel 框架源码分析（版本 v11.47.0）：

**`WorkCommand` 的命令签名**：

```bash
queue:work
 {connection? : The name of the queue connection to work}
 {--name=default : The name of the worker}
 {--queue= : The names of the queues to work}
 {--once : Only process the next job on the queue}
 {--stop-when-empty : Stop when the queue is empty}
 {--max-jobs=0 : The number of jobs to process before stopping}
 {--max-time=0 : The maximum number of seconds the worker should run}
 {--memory=128 : The memory limit in megabytes}
 {--sleep=3 : Number of seconds to sleep when no job is available}
 {--rest=0 : Number of seconds to rest between jobs}
 {--timeout=60 : The number of seconds a child process can run}
 {--tries=1 : Number of times to attempt a job before logging it failed}
 {--backoff=0 : Number of seconds to wait before retrying a job}
 {--force : Force the worker to run even in maintenance mode}
```

**源码验证结论**：**Laravel 11 `queue:work --tries` 默认值为 `1`**，即默认只尝试 1 次就标记为失败。

### 1.3 常见版本默认值对照

| Laravel 版本 | `--tries` 默认值 | 含义 |
|-------------|-----------------|------|
| 5.x - 8.x | `0` | **无限重试**（直到成功或超时） |
| 9.x | `0` | **无限重试** |
| 10.x | `1` | 尝试 1 次即失败 |
| 11.x | `1` | 尝试 1 次即失败 |

**重要注意**：`--tries=0` 在 Laravel 中表示 **无限重试**，而非"不重试"。这是一个常见的陷阱。

---

## 二、`--tries` 参数的生效机制

### 2.1 `tries` 值的三层优先级

Laravel 使用三层优先级确定最终的 `tries` 值：

```
优先级 1（最高）: 作业类自身的 $tries 属性
    ↓
优先级 2: 作业类的 retryUntil() 方法（基于时间的重试）
    ↓
优先级 3（最低）: queue:work --tries 命令行参数
```

**源码验证**（来自 Laravel `Worker` 类 `maxAttempts()` 方法）：

```php
protected function maxAttempts($job)
{
    if ($job->attempts() >= $this->maxExceptions($job)
        || method_exists($job, 'retryUntil') && $this->shouldRetryUntil($job)
        || $this->exceededAttempts($job)) {
        return false;
    }

    return $this->attempts($job) < $this->getJobTries($job);
}

protected function getJobTries($job)
{
    // 优先级 1: 作业类属性 $tries
    if (isset($job->tries)) {
        return $job->tries;
    }

    // 优先级 2: 方法 retryUntil() 返回时间戳（若设置则忽略 tries）
    if (method_exists($job, 'retryUntil')) {
        return null;  // 不限制次数，由时间控制
    }

    // 优先级 3: 命令行参数 --tries
    return $this->options->maxTries;
}
```

### 2.2 `attempts` 计数的递增时机

`attempts` 计数器在以下时机递增：

| 操作 | 递增 attempts 吗？ | 说明 |
|------|-------------------|------|
| Worker 取出作业 | **是** | 从队列取出即开始第 1 次尝试 |
| `$this->release($delay)` | **是** | 将作业放回队列，计数 +1 |
| `throw $exception` | **是** | 未捕获异常，计数 +1 |
| `$this->fail($exception)` | **否** | 直接标记为失败，计数不递增 |
| `return;` 正常结束 | **否** | 作业成功完成 |

**关键时序**：

```
Worker 取出作业 → attempts = 1
    ├─→ handle() 执行成功 → 作业完成
    ├─→ handle() 中 throw 异常 → attempts 仍为 1 → attempts < tries → 重新入队
    └─→ handle() 中 release(10) → attempts 仍为 1 → 重新入队（带延迟）

下一次取出 → attempts = 2
    ├─→ ...
```

**重要修正**：`release()` 不会立即递增 attempts。`attempts` 只在 Worker 取出作业时由框架递增。`$this->attempts()` 返回的是当前作业已被取出的次数。

### 2.3 `release()` 与 `--backoff` 的区别

| 机制 | 触发方式 | 延迟来源 | 适用场景 |
|------|----------|----------|----------|
| `release($delay)` | 应用代码显式调用 | `$delay` 参数 | 业务逻辑需要自定义重试 |
| `--backoff` | 框架驱动（异常未捕获） | 命令行参数 | 所有未捕获异常的默认重试间隔 |
| 作业类 `$backoff` 属性 | 框架驱动 | 作业类属性 | 指定作业的默认重试间隔 |

**优先级**：`release($delay)` **>** 作业类 `$backoff` **>** `--backoff`

---

## 三、本项目队列运行配置

### 3.1 Supervisord 配置

**源码位置**：`.github/docker/supervisord.conf:27-28`

```ini
[program:queue-worker]
command=/usr/local/bin/php /app/artisan queue:work --queue=high,standard,low --sleep=3 --tries=3
user=nginx
autostart=true
autorestart=true
```

**关键参数解析**：

| 参数 | 值 | 含义 |
|------|-----|------|
| `--queue` | `high,standard,low` | 队列优先级：high > standard > low |
| `--sleep` | `3` | 无作业时睡眠 3 秒 |
| `--tries` | `3` | **全局最大尝试次数 = 3** |

### 3.2 队列连接配置

**源码位置**：`config/queue.php:63-71`

```php
'redis' => [
    'driver' => 'redis',
    'connection' => env('REDIS_QUEUE_CONNECTION', 'default'),
    'queue' => env('REDIS_QUEUE', '{default}'),
    'retry_after' => (int) env('REDIS_QUEUE_RETRY_AFTER', 90),  // 90 秒
    'block_for' => null,
    'after_commit' => false,
],
```

**`retry_after` 的真实含义**：
> 如果一个作业被取出后超过 `retry_after` 秒仍未被确认（ack），队列驱动会认为该 worker 已崩溃，将该作业重新放回队列供其他 worker 消费。

**与 `--timeout` 的关系**：
```
--timeout < retry_after
    作业执行超时 → 抛出异常 → attempts 递增 → 正常重试

--timeout > retry_after
    作业执行超时 → 未到 timeout，先触发 retry_after → 作业被其他 worker 重新取出
    → 同一作业被多个 worker 并发执行 → **数据不一致风险**
```

本项目配置：
- `--timeout` 未显式指定 → 使用默认值 `60` 秒
- `retry_after = 90` 秒
- 满足 `60 < 90` → 安全配置

---

## 四、RevokeSftpAccessJob 重试边界分析

### 4.1 作业类定义

**源码位置**：`app/Jobs/RevokeSftpAccessJob.php:19-57`

```php
class RevokeSftpAccessJob implements ShouldQueue, ShouldBeUnique
{
    use Illuminate\Foundation\Queue\Queueable;  // Laravel 11 新版 Queueable

    public int $tries = 3;           // 最大尝试次数
    public int $maxExceptions = 1;   // 最大未捕获异常次数

    public function handle(DaemonRevocationRepository $repository): void
    {
        try {
            $repository->setNode($node)->deauthorize(...);
        } catch (DaemonConnectionException) {
            // Keep retrying this job with a longer and longer backoff
            // until we hit three attempts at which point we stop
            $this->release($this->attempts() * 10);
        }
    }
}
```

### 4.2 最终 `tries` 值

| 来源 | 值 | 是否生效 | 原因 |
|------|-----|----------|------|
| 作业类 `$tries` | `3` | **是** | 优先级最高 |
| 命令行 `--tries` | `3` | 否 | 被作业类值覆盖 |

**结论**：无论运行时如何配置，`RevokeSftpAccessJob` 的最大尝试次数固定为 **3 次**。

### 4.3 重试生命周期

```
T+0s    Worker 取出 → attempts = 1
    ├─→ deauthorize() 成功 → 作业完成
    └─→ DaemonConnectionException → release(1*10=10) → 延迟 10 秒

T+10s   Worker 取出 → attempts = 2
    ├─→ deauthorize() 成功 → 作业完成
    └─→ DaemonConnectionException → release(2*10=20) → 延迟 20 秒

T+30s   Worker 取出 → attempts = 3
    ├─→ deauthorize() 成功 → 作业完成
    └─→ DaemonConnectionException → release(3*10=30) → 延迟 30 秒
         （release() 执行成功，作业放回队列）

T+60s   Worker 取出 → attempts = 4
    → 框架检查：attempts(4) >= tries(3)
    → 直接标记为失败，**不执行 handle()**
    → 调用默认 failed() → 写入 failed_jobs 表
```

**总时间线**（全部失败时）：
- 第 1 次：0s → 延迟 10s
- 第 2 次：10s → 延迟 20s
- 第 3 次：30s → 延迟 30s
- 失败：60s 后取出 → 直接失败

**共尝试 `handle()` 3 次，第 4 次取出直接失败**。

### 4.4 `ShouldBeUnique` 的影响

```php
public function uniqueId(): string
{
    $target = $this->target instanceof Node ? "node:{$this->target->uuid}" : "server:{$this->target->uuid}";
    return "revoke-sftp:{$this->user}:{$target}";
}
```

**效果**：
- 同一时刻队列中只有一个相同 `uniqueId` 的作业
- `release()` 回队列后仍持有 unique 锁
- 防止重复调度产生的重复撤销任务

---

## 五、RunTaskJob 重试边界分析

### 5.1 作业类定义

**源码位置**：`app/Jobs/Schedule/RunTaskJob.php:16-135`

```php
class RunTaskJob implements ShouldQueue
{
    use Illuminate\Bus\Queueable;  // 老版 Queueable
    use DispatchesJobs;
    use SerializesModels;

    // 未定义 $tries
    // 未定义 $backoff
    // 未定义 $retryUntil
    // 未定义 $maxExceptions

    public function failed(?\Exception $exception = null)
    {
        $this->markTaskNotQueued();
        $this->markScheduleComplete();
    }
}
```

### 5.2 最终 `tries` 值

| 来源 | 值 | 是否生效 | 原因 |
|------|-----|----------|------|
| 作业类 `$tries` | 未定义 | - | 未设置 |
| 命令行 `--tries` | `3` | **是** | Docker 部署下使用此值 |
| Laravel 默认 `--tries` | `1` | 否 | 被命令行值覆盖 |

**结论**：`RunTaskJob` 的最大尝试次数**取决于运行时配置**：
- Docker 部署（默认）：**3 次**
- 手动启动 `queue:work` 无 `--tries`：**1 次**（Laravel 11 默认）
- 手动启动 `queue:work --tries=5`：**5 次**

### 5.3 异常处理路径

```
RunTaskJob::handle()
    │
    ├─→ schedule 不活跃 && 非手动 → return → 正常完成（不计失败）
    │
    ├─→ server->status != null
    │    └─→ $this->failed() → return
    │         注意：手动调用 failed()，作业正常完成
    │         attempts 不递增，不触发重试，不写入 failed_jobs
    │
    ├─→ 执行任务成功 → markTaskNotQueued() + queueNextTask() → 正常完成
    │
    └─→ 执行任务异常
         ├─→ continue_on_failure && DaemonConnectionException
         │    → 静默继续，跳过当前任务 → 正常完成
         │    → **不触发重试**
         │
         └─→ 其他异常 → throw $exception
              → 框架捕获，attempts 保留当前值
              → 若 attempts < tries → 重新入队（无延迟，无 backoff）
              → 若 attempts >= tries → 调用 failed() → 写入 failed_jobs
```

### 5.4 重试生命周期（Docker 部署下 tries=3）

```
T+0s    Worker 取出 → attempts = 1
    ├─→ 成功 → 正常完成
    ├─→ skip（status/continue_on_failure）→ 正常完成
    └─→ 其他异常 throw → attempts < 3 → 立即重新入队（无延迟）

T+~0s   Worker 取出 → attempts = 2  （几乎立即重试，无延迟）
    ├─→ 成功 → 正常完成
    └─→ 其他异常 throw → attempts < 3 → 立即重新入队

T+~0s   Worker 取出 → attempts = 3
    ├─→ 成功 → 正常完成
    └─→ 其他异常 throw → attempts >= 3
         → 调用 failed() → markTaskNotQueued() + markScheduleComplete()
         → 写入 failed_jobs 表
```

**关键特征**：
- **无 backoff**：重试之间无延迟，几乎立即执行
- **立即重试**：适用于临时性故障（如短暂网络波动）
- **3 次尝试**在不到 1 秒内完成（取决于队列轮询速度）

### 5.5 两种 `failed()` 调用的区别

| 维度 | 手动调用 `$this->failed()` | 框架调用 `failed()` |
|------|---------------------------|---------------------|
| 触发时机 | `server->status != null` 时 | attempts >= tries 时 |
| attempts 递增 | 否 | 是（框架已完成递增） |
| 作业状态 | 正常完成（success） | 失败（failed） |
| 写入 failed_jobs | 否 | 是 |
| 是否可重试 | 否（已 return） | 否（已达上限） |
| 清理逻辑 | markTaskNotQueued + markScheduleComplete | markTaskNotQueued + markScheduleComplete |

---

## 六、两个 Job 重试行为对比总结

| 维度 | RevokeSftpAccessJob | RunTaskJob (Docker 部署) |
|------|---------------------|--------------------------|
| **最大尝试次数** | **3 次**（源码确定） | **3 次**（运行配置确定） |
| **tries 来源** | 类属性 `$tries = 3` | 命令行 `--tries=3` |
| **可移植性** | 不受部署方式影响 | 受 `--tries` 参数影响 |
| **重试驱动** | `release()` 应用层驱动 | 异常抛出框架驱动 |
| **重试延迟** | 有（10s → 20s → 30s） | **无**（立即重试） |
| **backoff 策略** | 线性退避 `attempts * 10` | 无（或 `--backoff`） |
| **maxExceptions** | `1`（未捕获异常 1 次即终止） | 未定义（无限） |
| **ShouldBeUnique** | 是 | 否 |
| **自定义 failed()** | 否（使用默认） | **是** |
| **特殊跳过逻辑** | 无 | `continue_on_failure` + `server->status` |
| **手动调用 failed()** | 否 | 是（`server->status != null` 时） |
| **总耗时（全失败）** | ~60 秒 | ~0 秒（立即重试） |
| **适用场景** | 节点可能长时间离线，需间隔重试 | 短暂故障快速重试 |

---

## 七、关键源码索引

| 模块 | 文件路径 | 关键内容 |
|------|----------|----------|
| Supervisord 配置 | `.github/docker/supervisord.conf:28` | `queue:work --tries=3 --sleep=3 --queue=high,standard,low` |
| 队列连接配置 | `config/queue.php:63-71` | `retry_after = 90` 秒 |
| RevokeSftpAccessJob | `app/Jobs/RevokeSftpAccessJob.php` | `$tries=3`, `$maxExceptions=1`, `release(attempts*10)` |
| RunTaskJob | `app/Jobs/Schedule/RunTaskJob.php` | 无 `$tries`，有 `failed()`，有 `continue_on_failure` 逻辑 |
| Laravel 11 Queueable（新版） | `Illuminate\Foundation\Queue\Queueable` | RevokeSftpAccessJob 使用 |
| Laravel Bus Queueable（老版） | `Illuminate\Bus\Queueable` | RunTaskJob 使用 |
| Laravel WorkCommand | `Illuminate\Queue\Console\WorkCommand` | `--tries` 默认值为 `1` |

---

## 八、前文勘误中的修正点

| 前文结论 | 修正后 |
|----------|--------|
| "Laravel 默认 `tries = 1`" | ✅ 确认正确（Laravel 11 确实默认 `--tries=1`） |
| "RunTaskJob 未定义 `$tries`，Docker 下 tries=3" | ✅ 确认正确 |
| "第 3 次 `release(30)` 不会执行" | ❌ 修正：第 3 次 `release(30)` **确实执行**，但第 4 次取出时因 `attempts(4) >= tries(3)` 被框架直接拦截，不执行 `handle()` |
| "`release()` 递增 attempts" | ❌ 修正：`release()` **不直接递增** attempts。attempts 只在 Worker 取出作业时由框架递增 |
| "RunTaskJob 的 `failed()` 手动调用不触发重试" | ✅ 确认正确，但需补充：也**不写入 failed_jobs 表** |
| "`--tries` 优先级最高" | ❌ 修正：作业类 `$tries` 属性 **优先于** 命令行 `--tries` 参数 |
