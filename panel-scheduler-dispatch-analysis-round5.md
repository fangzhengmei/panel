# Panel 排程器任务派发流程分析 - 第五轮（驱动差异与适用前提校准）

## 修正说明

在第四轮分析的基础上，补充**结论适用前提**与**驱动差异说明**，明确区分：
- 哪些结论依赖 **Redis 队列实现** 和当前 `queue:work --tries=3` 参数
- 哪些是 **Laravel 队列通用行为**
- 对 **database/sqs** 等非 Redis 驱动给出差异点或不适用点

本文档保留第四轮的完整时序链和核心结论，并在开头和相关章节补充驱动差异说明。

---

## 0. 结论适用前提与驱动差异总览

### 0.1 本项目当前运行配置

| 配置项 | 值 | 来源 |
|--------|-----|------|
| 队列驱动 | Redis | `config/queue.php:15` + `.env` |
| Worker 启动参数 | `--queue=high,standard,low --sleep=3 --tries=3` | `.github/docker/supervisord.conf:28` |
| retry_after | 90 秒 | `config/queue.php:67` |

**重要**：本分析文档中的以下内容**仅适用于当前配置**（Redis 驱动 + --tries=3）：
- 时序链中 attempts 的值（1、2、3）
- retry_after 的 Lua 脚本实现细节
- 精确的重试延迟控制
- 重复执行的幂等性风险

### 0.2 通用行为 vs 驱动特定行为

| 行为分类 | Laravel 通用行为（所有驱动） | Redis 特定 | Database 特定 | SQS 特定 |
|---------|----------------------------|-----------|--------------|----------|
| `--tries` 参数优先级 | ✅ 所有驱动适用 | - | - | - |
| `$tries` 属性优先级 | ✅ 所有驱动适用 | - | - | - |
| attempts 计数机制 | ✅ 所有驱动适用（但递增时机不同） | Lua 脚本中递增 | SELECT + UPDATE 时递增 | SQS 服务端管理 |
| retry_after 机制 | ❌ 概念通用，实现不同 | Redis zset + Lua | MySQL `reserved_at` 字段 | SQS VisibilityTimeout |
| `failed_jobs` 写入时机 | ✅ 所有驱动适用 | - | - | - |
| `dispatchNow` 行为 | ✅ 所有驱动适用 | - | - | - |
| `continue_on_failure` 逻辑 | ✅ 所有驱动适用 | - | - | - |
| 精确延迟重试 | ❌ 驱动差异 | ✅ 精确到毫秒 | ❌ ±1~3 秒偏差 | ❌ 最小 60 秒 |
| 重复执行风险 | ❌ 驱动差异 | ✅ 高（90秒超时） | ✅ 中（数据库锁） | ✅ 低（SQS 托管） |

---

## 1. Laravel Queue attempts 计数的底层机制

### 1.1 Redis 队列取出任务的 Lua 脚本（本项目当前使用）

Laravel Redis 队列在取出任务时执行以下 Lua 脚本：

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

**Redis 特定结论**：
- `attempts` 计数存储在任务的 payload 中，随任务一起序列化
- **每次从队列取出任务时，`attempts` 立即 +1**
- 无论任务是首次执行、失败后重试、还是 retry_after 超时重新入队，只要被取出就递增
- `attempts` 永远不会重置，直到任务被删除或标记为最终失败

### 1.2 Database 队列的 attempts 递增（差异点）

**不适用本项目，但值得了解**：

Database 驱动使用 `jobs` 表的 `attempts` 字段，在 SELECT + UPDATE 原子操作中递增：

```sql
-- 简化的 SQL 逻辑
SELECT * FROM jobs WHERE queue = ? AND available_at <= NOW() ORDER BY id ASC LIMIT 1 FOR UPDATE;
UPDATE jobs SET attempts = attempts + 1, reserved_at = NOW() WHERE id = ?;
```

**差异点**：
- attempts 存储在数据库字段中，不是 payload 中
- 递增时机相同（取出时 +1）
- 但受事务隔离级别影响，高并发下可能出现死锁

### 1.3 SQS 队列的 attempts 递增（差异点）

**不适用本项目，但值得了解**：

SQS 驱动不依赖本地 attempts 计数，而是由 AWS SQS 服务端管理 `ApproximateReceiveCount`：

**差异点**：
- attempts 计数由 SQS 服务端维护
- Laravel 只能读取，不能修改
- 最大接收次数由 SQS 队列配置，不是 Laravel 控制
- 超过最大接收次数后，消息自动移到死信队列（DLQ）

### 1.4 attempts 与 maxTries 的判断逻辑（通用）

Worker 进程中的判断逻辑（所有驱动通用）：

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

### 2.1 运行配置回顾（本项目当前）

| 配置项 | 值 | 来源 |
|--------|-----|------|
| `--tries` | 3 | supervisord.conf:28 |
| `retry_after` | 90 | config/queue.php:67 |

### 2.2 统一时序链（正常失败重试场景）（Redis 驱动特定）

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

**关键观察（Redis 驱动）**：
- `attempts` 从 0 开始，每次取出 +1
- 第一次执行：`attempts=1`
- 第二次执行：`attempts=2`
- 第三次执行：`attempts=3` → 达到上限，失败

### 2.3 Database 驱动时序链差异

**不适用本项目**：

Database 驱动的 retry_after 不是 Lua 脚本实现，而是通过 `reserved_at` 字段判断：

```
T0: 放入队列: attempts=0, reserved_at=null
T1: Worker 取出: UPDATE SET attempts=1, reserved_at=NOW()
    执行失败 → release: UPDATE SET reserved_at=null, available_at=NOW()
T2: Worker 取出: UPDATE SET attempts=2, reserved_at=NOW()
    ...
```

**差异点**：
- retry_after 逻辑：Worker 轮询时检查 `reserved_at < NOW() - retry_after`
- 没有原子性保证，可能出现两个 Worker 同时取出同一任务
- 重试延迟可能有 ±3 秒偏差（受 --sleep 参数影响）

### 2.4 SQS 驱动时序链差异

**不适用本项目**：

SQS 使用 VisibilityTimeout 机制，完全由 AWS 服务端管理：

```
T0: 发送消息到 SQS
T1: Worker 接收消息 → SQS 设置 VisibilityTimeout=90s
    执行失败 → 不删除消息
T1+90s: VisibilityTimeout 到期，消息重新变为可见
T2: 同一或其他 Worker 再次接收消息
    ...
T6: 第 3 次接收后仍失败 → Laravel 标记为最终失败
```

**差异点**：
- retry_after 等价于 SQS 的 VisibilityTimeout
- attempts 计数由 SQS 维护（ApproximateReceiveCount）
- 超过 SQS 队列的 `maxReceiveCount` 后，消息自动移到死信队列
- 最小 VisibilityTimeout 是 60 秒，不能更小

### 2.5 retry_after 超时场景的时序链（Redis 驱动特定）

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

### 2.6 retry_after 与 tries 的关系结论（通用 + 驱动特定）

| 结论 | 通用/特定 | 说明 |
|------|-----------|------|
| attempts 永远不会重置 | ✅ 通用 | 所有驱动都不会重置 attempts |
| tries 是硬性上限 | ✅ 通用 | 所有驱动都会在 attempts >= maxTries 时标记失败 |
| retry_after 可能导致重复执行 | ❌ 特定 | Redis: 高风险；Database: 中风险；SQS: 低风险 |
| retry_after 不影响 tries 限制 | ✅ 通用 | 即使触发 retry_after，最终仍受 tries 限制 |

**修正前错误结论**：retry_after 可能导致实际执行次数超过 `--tries=3`，因为 attempts 会重置

**修正后正确结论**：
- `attempts` 永远不会重置，每次取出都递增
- `--tries=3` 是硬性上限，无论重试原因是什么
- retry_after 可能导致**同一时刻有多个 Worker 执行同一任务**（重复执行）
- 但每个任务实例的 attempts 计数独立递增，最终都会受 `--tries=3` 限制
- 因此，retry_after 不会导致 attempts 超过 tries，只会导致**重复执行**（这是一个独立的幂等性问题）

---

## 3. 失败路径的精确边界

### 3.1 异步 Worker 失败路径（dispatch + queue:work）（通用）

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

**进入此路径的充要条件**（通用）：
1. 任务通过 `dispatch()` 派发
2. 任务由 `queue:work` 进程执行
3. `handle()` 方法抛出未被捕获的异常

### 3.2 dispatchNow 同步失败路径（通用）

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

**关键区别（通用，与驱动无关）**：
- 不经过 Worker 进程
- 不调用 `Worker::failJob()` 方法
- **不写入 failed_jobs 表**
- attempts 计数器不生效（因为没有队列取出操作）

### 3.3 两条路径的边界对比表（通用）

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

## 4. continue_on_failure 对失败路径的影响（通用）

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

### 4.2 对失败路径的影响（通用，与驱动无关）

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

## 5. 关键结论汇总（标注适用范围）

### 5.1 attempts 计数行为（通用）

| 场景 | attempts 变化 | 适用范围 |
|------|--------------|----------|
| 任务首次放入队列 | `attempts = 0` | 所有驱动 |
| 每次从队列取出（无论原因） | `attempts++` | 所有驱动（但递增时机不同） |
| 失败后 release 放回队列 | 不变化，下次取出时 +1 | 所有驱动 |
| retry_after 超时重新入队 | 不变化，下次取出时 +1 | Redis/Database；SQS 由服务端管理 |
| 任务完成或最终失败 | 停止计数，任务被删除 | 所有驱动 |

### 5.2 --tries=3 的准确含义（通用）

- **不是**：重试 3 次，共执行 4 次
- **而是**：总共最多执行 3 次（首次 + 最多 2 次重试）
- attempts 值分别为：1、2、3
- `attempts >= 3` 时立即标记为最终失败
- **适用范围**：所有驱动，但 attempts 递增时机不同

### 5.3 retry_after 的真实作用（驱动差异大）

| 认知点 | Redis（本项目） | Database | SQS |
|--------|----------------|----------|-----|
| 是重试机制吗 | ❌ 否，是"僵尸任务"保护 | ❌ 否，是"僵尸任务"保护 | ❌ 否，是 VisibilityTimeout |
| 会重置 attempts 吗 | ❌ 否，持续递增 | ❌ 否，持续递增 | ❌ 否，服务端递增 |
| 可能超过 tries 吗 | ❌ 否，tries 是硬性上限 | ❌ 否，tries 是硬性上限 | ❌ 否，受 SQS maxReceiveCount 限制 |
| 可能导致重复执行吗 | ✅ 是（高风险，90秒） | ✅ 是（中风险，数据库锁） | ✅ 是（低风险，AWS 托管） |

### 5.4 failed_jobs 写入条件（通用）

**必须同时满足**：
1. ✅ 任务通过异步队列执行（经过 Worker 进程）
2. ✅ `handle()` 抛出未被捕获的异常
3. ✅ `attempts() >= maxTries`（即已达到最大尝试次数）
4. ❌ 不是 `continue_on_failure` 捕获的 `DaemonConnectionException`

**适用范围**：所有驱动

### 5.5 dispatchNow 的边界（通用）

**dispatchNow 永远不会**：
- 写入 failed_jobs 表
- 触发重试逻辑
- 使 attempts 计数器递增
- 经过 Worker 进程的失败处理流程

**dispatchNow 只会**：
- 同步执行 `handle()` 方法
- 需要手动调用 `failed()` 进行状态清理
- 将异常直接抛出给调用者

**适用范围**：所有驱动

---

## 6. 完整失败生命周期对比（标注适用范围）

### 6.1 异步队列 + 最终失败（3 次尝试）（通用）

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

**适用范围**：所有驱动

### 6.2 异步队列 + continue_on_failure 捕获（通用）

```
1. 首次执行: attempts=1 → 异常被捕获 → 不抛出
   ↓
   Worker 认为成功 → 不重试
   ↓
   不写入 failed_jobs
   ↓
   继续执行后续任务
```

**适用范围**：所有驱动

### 6.3 异步队列 + retry_after 超时（Redis 特定）

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

**适用范围**：Redis 驱动；Database 驱动类似但实现不同；SQS 由 AWS 管理

### 6.4 dispatchNow 手动触发（通用）

```
1. 直接调用 handle() → 失败
2. 手动调用 failed() 清理状态
3. 异常抛出 → HTTP 500
4. 不写入 failed_jobs
5. 不重试
```

**适用范围**：所有驱动

---

## 7. 驱动选择建议（基于本项目场景）

| 考量维度 | Redis（当前） | Database | SQS |
|---------|--------------|----------|-----|
| 部署复杂度 | 需额外部署 Redis | 无需额外组件 | AWS 托管 |
| 性能 | 极高（内存操作） | 中（数据库操作） | 高（AWS 托管） |
| 精确延迟控制 | ✅ 精确 | ❌ ±1~3 秒偏差 | ❌ 最小 60 秒 |
| 重复执行风险 | 高（90秒超时） | 中（数据库锁） | 低（AWS 托管） |
| 监控能力 | 强（Horizon） | 弱（需自行实现） | 强（CloudWatch） |
| 适用场景 | 本项目排程任务（电源/命令/备份） | 小型项目、低并发 | AWS 生态系统项目 |

**本项目选择 Redis 的合理性**：
- 排程任务需要精确的时间控制
- 任务执行时间通常较短（电源操作、命令发送 < 90 秒）
- 备份任务可能超过 90 秒，需要注意幂等性
- Redis 性能足以支撑 Panel 的任务量

---

## 8. 潜在风险（基于精确行为 + 驱动特性）

### 8.1 Redis 驱动特定风险
1. **幂等性风险**：retry_after 超时可能导致任务被重复执行，电源/命令任务可能执行多次
2. **长任务风险**：执行时间超过 90 秒的任务（如大备份）容易触发 retry_after 重复执行
3. **状态不一致**：retry_after 导致重复执行时，任务状态可能不一致

### 8.2 通用风险（所有驱动）
1. **无失败通知**：continue_on_failure 捕获的异常完全静默，用户无法感知
2. **手动触发无记录**：dispatchNow 失败不写入 failed_jobs，问题排查困难

### 8.3 切换到 Database 驱动可能引入的新风险
1. **延迟偏差**：重试延迟可能有 ±1~3 秒偏差，影响精确排程
2. **数据库负载**：高并发下 `jobs` 表可能成为瓶颈
3. **死锁风险**：SELECT + UPDATE 模式在高并发下可能出现死锁

### 8.4 切换到 SQS 驱动可能引入的新风险
1. **最小延迟限制**：VisibilityTimeout 最小 60 秒，无法实现快速重试
2. **成本增加**：SQS 按请求计费
3. **供应商锁定**：绑定 AWS 生态
