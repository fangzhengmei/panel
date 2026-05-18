# Panel 排程器任务派发流程分析 - 第四轮（retry_after 精确对齐）

## 修正说明

第三轮分析中 `retry_after` 部分存在两处关键偏差，本次基于 Laravel 11 队列源码精确修正：

1. **错误结论 1**：retry_after 触发重试时 `attempts` 计数器会重置
   **修正**：`attempts` 永远不会重置，每次从队列取出都会递增

2. **错误结论 2**：retry_after 可能导致实际执行次数超过 `--tries=3`
   **修正**：`--tries=3` 是硬性上限，`attempts >= maxTries` 时立即标记失败

---

## 1. Laravel Queue attempts 计数的底层机制

### 1.1 Redis 队列取出任务的 Lua 脚本

Laravel Redis 队列在取出任务时执行以下 Lua 脚本（来自源码）：

```lua
local job = redis.call('lpop', KEYS[1])
local reserved = false
if(job ~= false) then
    reserved = cjson.decode(job)
    reserved['attempts'] = reserved['attempts'] + 1  -- 关键：取出时立即 +1
    reserved = cjson.encode(reserved)
    redis.call('zadd', KEYS[2], ARGV[1], reserved)
end
return {job, reserved}
```

**核心结论**：
- `attempts` 计数存储在任务的 payload 中，随任务一起序列化
- **每次从队列取出任务时，`attempts` 立即 +1**
- 无论任务是首次执行、失败后重试、还是 retry_after 超时重新入队，只要被取出就递增
- `attempts` 永远不会重置，直到任务被删除或标记为最终失败

### 1.2 attempts 与 maxTries 的判断逻辑

Worker 进程中的判断逻辑（简化）：

```php
// 取出任务，此时 attempts 已经 +1
$job = $this->getNextJob($queue);

try {
    $job->fire();  // 执行 RunTaskJob::handle()
} catch (\Throwable $e) {
    // 检查是否可以重试
    if ($job->attempts() < $maxTries) {
        $job->release();  // 放回队列，下次取出时 attempts 再次 +1
    } else {
        $this->failJob($job, $e);  // 标记失败，写入 failed_jobs
    }
}
```

---

## 2. retry_after、--tries=3、failed_jobs 的完整时序链

### 2.1 运行配置回顾

| 配置项 | 值 | 来源 |
|--------|-----|------|
| `--tries` | 3 | supervisord.conf:28 |
| `retry_after` | 90 | config/queue.php:67 |

### 2.2 统一时序链（正常失败重试场景）

```
时间轴 →
│
├─ T0: 任务被放入队列（payload: attempts=0）
│
├─ T1: Worker 取出任务
│      Lua 脚本执行: attempts = 0 + 1 = 1
│      任务标记为 "reserved"，90秒超时
│      ↓
│      RunTaskJob::handle() 执行，抛出异常
│      ↓
│      Worker 判断: attempts(1) < maxTries(3) → 可以重试
│      ↓
│      $job->release() 放回队列
│
├─ T2: Worker 再次取出任务（release 后立即或延迟后）
│      Lua 脚本执行: attempts = 1 + 1 = 2
│      任务标记为 "reserved"，90秒超时
│      ↓
│      RunTaskJob::handle() 执行，再次抛出异常
│      ↓
│      Worker 判断: attempts(2) < maxTries(3) → 可以重试
│      ↓
│      $job->release() 放回队列
│
├─ T3: Worker 第三次取出任务
│      Lua 脚本执行: attempts = 2 + 1 = 3
│      任务标记为 "reserved"，90秒超时
│      ↓
│      RunTaskJob::handle() 执行，第三次抛出异常
│      ↓
│      Worker 判断: attempts(3) < maxTries(3) → 否
│      ↓
│      调用 RunTaskJob::failed() 清理状态
│      ↓
│      写入 failed_jobs 表
│      ↓
│      从队列中删除任务
│
└─ 任务结束
```

**关键观察**：
- `attempts` 从 0 开始，每次取出 +1
- 第一次执行：`attempts=1`
- 第二次执行：`attempts=2`
- 第三次执行：`attempts=3` → 达到上限，失败

### 2.3 retry_after 超时场景的时序链

```
时间轴 →
│
├─ T0: 任务被放入队列（payload: attempts=0）
│
├─ T1: Worker A 取出任务
│      Lua 脚本执行: attempts = 0 + 1 = 1
│      任务标记为 "reserved"，超时时间 T1+90
│      ↓
│      RunTaskJob::handle() 开始执行
│      ↓
│      任务执行时间很长（如备份大文件）
│
├─ T1+90: 达到 retry_after 超时时间
│      Worker B（或下次轮询）执行 migrateExpiredJobs
│      Lua 脚本将任务从 "reserved" 集合移回主队列
│      注意：此时 payload 中的 attempts 仍为 1
│
├─ T1+95: Worker B 取出同一任务
│      Lua 脚本执行: attempts = 1 + 1 = 2  ← 继续递增，不重置
│      ↓
│      现在有两个 Worker 同时在执行同一个任务！
│      Worker A: 仍在执行（attempts 逻辑上是 1）
│      Worker B: 开始执行（attempts 逻辑上是 2）
│
└─ 注意：这就是 retry_after 可能导致重复执行的原因，但 attempts 计数仍在正常递增
```

### 2.4 retry_after 与 tries 的关系结论

**修正前错误结论**：retry_after 可能导致实际执行次数超过 `--tries=3`，因为 attempts 会重置

**修正后正确结论**：
- `attempts` 永远不会重置，每次取出都递增
- `--tries=3` 是硬性上限，无论重试原因是什么
- retry_after 可能导致**同一时刻有多个 Worker 执行同一任务**（重复执行）
- 但每个任务实例的 attempts 计数独立递增，最终都会受 `--tries=3` 限制
- 因此，retry_after 不会导致 attempts 超过 tries，只会导致**重复执行**（这是一个独立的幂等性问题）

---

## 3. 失败路径的精确边界

### 3.1 异步 Worker 失败路径（dispatch + queue:work）

```
异常抛出点: RunTaskJob::handle()
    ↓
Laravel Queue Worker 捕获异常
    ↓
┌─ Worker 内部逻辑 ─────────────────────────────────────────────────┐
│                                                                   │
│  1. 获取 maxTries（优先级:  Job $tries → --tries=3 → 默认 1）     │
│  2. if ($job->attempts() < $maxTries) {                            │
│        $job->release();        // 放回队列，不写 failed_jobs      │
│     } else {                                                       │
│        $this->failJob($job, $e);  // 写 failed_jobs                │
│     }                                                              │
│                                                                   │
│  failJob() 内部:                                                   │
│    ├─ 调用 Job::failed() 方法（RunTaskJob::failed()）              │
│    ├─ 触发 JobFailed 事件                                          │
│    └─ 写入 failed_jobs 表                                          │
│                                                                   │
└───────────────────────────────────────────────────────────────────┘
```

**进入此路径的充要条件**：
1. 任务通过 `dispatch()` 派发
2. 任务由 `queue:work` 进程执行
3. `handle()` 方法抛出未被捕获的异常

### 3.2 dispatchNow 同步失败路径

```
异常抛出点: RunTaskJob::handle()
    ↓
ProcessScheduleService 捕获异常（因为在 try-catch 中调用 dispatchNow）
    ↓
┌─ ProcessScheduleService 内部逻辑 ────────────────────────────────┐
│                                                                   │
│  try {                                                             │
│      $this->dispatcher->dispatchNow($job);                         │
│  } catch (\Exception $exception) {                                 │
│      $job->failed($exception);  // 手动调用，只做状态清理          │
│      throw $exception;         // 继续向上抛出                     │
│  }                                                                 │
│                                                                   │
└───────────────────────────────────────────────────────────────────┘
    ↓
异常抛出到 HTTP 层，返回 500 响应
```

**关键区别**：
- 不经过 Worker 进程
- 不调用 `Worker::failJob()` 方法
- **不写入 failed_jobs 表**
- attempts 计数器不生效（因为没有队列取出操作）

### 3.3 两条路径的边界对比表

| 操作 | 异步 Worker 路径 | dispatchNow 同步路径 |
|------|-----------------|----------------------|
| 异常被谁捕获 | Worker 进程 | ProcessScheduleService |
| attempts 计数 | 生效，每次取出 +1 | 不生效 |
| 重试逻辑 | 自动重试（attempts < maxTries） | 不重试 |
| 调用 Job::failed() | Worker 自动调用 | 必须手动调用 |
| 写入 failed_jobs | 最终失败时写入 | 永远不写入 |
| 异常返回方式 | 静默处理，记录日志 | 抛出到上层，返回 HTTP 500 |
| 任务最终状态 | 删除或重试中 | 立即结束 |

---

## 4. continue_on_failure 对失败路径的影响

### 4.1 代码逻辑回顾

```php
// RunTaskJob.php:74-80
catch (\Exception $exception) {
    if (!($this->task->continue_on_failure && $exception instanceof DaemonConnectionException)) {
        throw $exception;
    }
    // 不抛出，静默继续
}
```

### 4.2 对失败路径的影响

| 条件 | 异常是否抛出 | 是否进入 Worker 失败处理 | 是否写入 failed_jobs | 是否继续后续任务 |
|------|-------------|------------------------|---------------------|-----------------|
| `continue_on_failure=true` + `DaemonConnectionException` | ❌ 不抛出 | ❌ 不进入 | ❌ 不写入 | ✅ 继续 |
| `continue_on_failure=false` + 任何异常 | ✅ 抛出 | ✅ 进入 | 达到 maxTries 后写入 | ❌ 停止 |
| `continue_on_failure=true` + 其他异常 | ✅ 抛出 | ✅ 进入 | 达到 maxTries 后写入 | ❌ 停止 |

**设计意图**：
- 只容忍网络连接异常（守护进程暂时不可达）
- 业务逻辑错误必须终止，避免级联错误
- 网络异常被静默捕获，Worker 认为任务成功，不会重试也不会记录失败

---

## 5. 关键结论汇总（最终版）

### 5.1 attempts 计数行为

| 场景 | attempts 变化 |
|------|--------------|
| 任务首次放入队列 | `attempts = 0` |
| 每次从队列取出（无论原因） | `attempts++` |
| 失败后 release 放回队列 | 不变化，下次取出时 +1 |
| retry_after 超时重新入队 | 不变化，下次取出时 +1 |
| 任务完成或最终失败 | 停止计数，任务被删除 |

### 5.2 --tries=3 的准确含义

- **不是**：重试 3 次，共执行 4 次
- **而是**：总共最多执行 3 次（首次 + 最多 2 次重试）
- attempts 值分别为：1、2、3
- `attempts >= 3` 时立即标记为最终失败

### 5.3 retry_after 的真实作用

| 认知点 | 错误认知 | 正确认知 |
|--------|---------|---------|
| retry_after 是重试机制 | 是 | 否，是"僵尸任务"保护 |
| retry_after 会重置 attempts | 是 | 否，attempts 持续递增 |
| retry_after 可能超过 tries | 是 | 否，tries 是硬性上限 |
| retry_after 可能导致重复执行 | 否 | 是（幂等性问题） |

### 5.4 failed_jobs 写入条件

**必须同时满足**：
1. ✅ 任务通过异步队列执行（经过 Worker 进程）
2. ✅ `handle()` 抛出未被捕获的异常
3. ✅ `attempts() >= maxTries`（即已达到最大尝试次数）
4. ❌ 不是 `continue_on_failure` 捕获的 `DaemonConnectionException`

### 5.5 dispatchNow 的边界

**dispatchNow 永远不会**：
- 写入 failed_jobs 表
- 触发重试逻辑
- 使 attempts 计数器递增
- 经过 Worker 进程的失败处理流程

**dispatchNow 只会**：
- 同步执行 `handle()` 方法
- 需要手动调用 `failed()` 进行状态清理
- 将异常直接抛出给调用者

---

## 6. 完整失败生命周期对比

### 6.1 异步队列 + 最终失败（3 次尝试）

```
1. 首次执行: attempts=1 → 失败 → release
2. 二次执行: attempts=2 → 失败 → release
3. 三次执行: attempts=3 → 失败 → 达到上限
   ↓
   调用 RunTaskJob::failed()
   ↓
   写入 failed_jobs 表
   ↓
   任务删除
```

### 6.2 异步队列 + continue_on_failure 捕获

```
1. 首次执行: attempts=1 → 异常被捕获 → 不抛出
   ↓
   Worker 认为成功 → 不重试
   ↓
   不写入 failed_jobs
   ↓
   继续执行后续任务
```

### 6.3 异步队列 + retry_after 超时

```
1. Worker A 取出: attempts=1 → 开始执行
2. 执行超过 90 秒 → retry_after 触发
3. 任务被移回主队列（attempts 仍为 1）
4. Worker B 取出: attempts=2 → 开始执行
   ↓
   现在两个 Worker 同时执行同一任务（重复执行）
   ↓
   各自独立判断重试逻辑，最终都会受 tries=3 限制
```

### 6.4 dispatchNow 手动触发

```
1. 直接调用 handle() → 失败
2. 手动调用 failed() 清理状态
3. 异常抛出 → HTTP 500
4. 不写入 failed_jobs
5. 不重试
```

---

## 7. 潜在风险（基于精确行为）

1. **幂等性风险**：retry_after 超时可能导致任务被重复执行，电源/命令任务可能执行多次
2. **无失败通知**：continue_on_failure 捕获的异常完全静默，用户无法感知
3. **手动触发无记录**：dispatchNow 失败不写入 failed_jobs，问题排查困难
4. **长任务风险**：执行时间超过 90 秒的任务（如大备份）容易触发 retry_after 重复执行
5. **状态不一致**：retry_after 导致重复执行时，任务状态可能不一致
