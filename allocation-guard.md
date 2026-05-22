# 多服务器环境下端口与地址分配机制分析

## 一、核心数据模型

分配（Allocation）是系统中表示 `IP + Port` 组合的实体，用于标识服务器可使用的网络地址。

**数据结构** (`app/Models/Allocation.php:42-134`)：
- `node_id`: 所属节点ID
- `ip`: IP地址
- `port`: 端口号（1024-65535）
- `server_id`: 绑定的服务器ID（NULL表示未分配）
- `ip_alias`: IP别名（可选）
- `notes`: 备注信息

**数据库约束** (`database/schema/mysql-schema.sql:54-57`)：
- 唯一索引：`allocations_node_id_ip_port_unique` - 确保 `(node_id, ip, port)` 组合全局唯一
- 外键约束：`allocations_server_id_foreign` - 带 `ON DELETE SET NULL`，服务器删除时自动回收分配

---

## 二、分配候选筛选机制

候选筛选是从资源池中找到可用分配的过程，分为三层筛选。

### 2.1 节点级筛选 (`FindViableNodesService`)

**位置**：`app/Services/Deployment/FindViableNodesService.php:69-99`

**筛选条件**：
1. **位置过滤**：可按 `location_id` 筛选指定区域的节点
2. **公开节点**：只选择 `public = 1` 的节点（允许自动部署）
3. **资源校验**：
   - 内存：`已用内存 + 需求内存 <= 节点内存 * (1 + memory_overallocate/100)`
   - 磁盘：`已用磁盘 + 需求磁盘 <= 节点磁盘 * (1 + disk_overallocate/100)`

**关键代码**：
```php
->havingRaw('(IFNULL(SUM(servers.memory), 0) + ?) <= (nodes.memory * (1 + (nodes.memory_overallocate / 100)))', [$this->memory])
->havingRaw('(IFNULL(SUM(servers.disk), 0) + ?) <= (nodes.disk * (1 + (nodes.disk_overallocate / 100)))', [$this->disk]);
```

### 2.2 分配级筛选 (`AllocationRepository::getRandomAllocation`)

**位置**：`app/Repositories/Eloquent/AllocationRepository.php:59-99`

**筛选条件**：
1. **基础条件**：`server_id IS NULL`（未被占用）
2. **节点范围**：可指定 `node_id` 列表
3. **端口筛选**：
   - 支持单个端口：`whereIn('port', $ports)`
   - 支持端口区间：`whereBetween('port', [$start, $end])`
4. **专用IP模式**（`dedicated = true`）：
   - 调用 `getDiscardableDedicatedAllocations()` 获取已有服务器使用的 `(node_id, ip)` 组合
   - 排除这些组合，确保该IP下没有其他服务器

**返回策略**：`inRandomOrder()->first()` - 随机返回，保证分配均衡

### 2.3 自动分配服务 (`FindAssignableAllocationService`)

**位置**：`app/Services/Allocations/FindAssignableAllocationService.php:31-112`

**分配策略**：
1. **优先复用**：在同一IP下查找未分配端口（带 `lockForUpdate()` 行锁）
2. **动态创建**：找不到时在配置的端口范围内创建新分配
   - 计算已有端口与配置范围的差集
   - 随机选择可用端口创建
   - 使用 `insertIgnore` 防止重复

---

## 三、占用校验和冲突防护机制

冲突防护采用**多层防御**策略，从数据库层到应用层层层把关。

### 3.1 数据库层：终极防护

**1. 唯一索引约束** (`database/schema/mysql-schema.sql:54`)
```sql
UNIQUE KEY `allocations_node_id_ip_port_unique` (`node_id`,`ip`,`port`)
```
- 任何重复的 `(node_id, ip, port)` 组合都会触发数据库异常
- 这是最后一道防线，确保数据一致性

**2. 外键约束** (`database/schema/mysql-schema.sql:57`)
```sql
CONSTRAINT `allocations_server_id_foreign` FOREIGN KEY (`server_id`) REFERENCES `servers` (`id`) ON DELETE SET NULL
```
- 服务器删除时自动将 `server_id` 置为 NULL，避免悬挂引用

### 3.2 应用层：主动检查

**1. 批量插入防重复** (`EloquentRepository::insertIgnore`, `app/Repositories/Eloquent/EloquentRepository.php:251-277`)
```php
$statement = "insert ignore into $table ($columns) values $parameters";
```
- 批量创建分配时使用 `INSERT IGNORE`，自动跳过已存在的记录
- 不会抛出异常，适合批量导入场景

**2. 查询条件过滤**
所有分配查询都显式加上 `whereNull('server_id')` 条件：
- `AllocationRepository.php:26, 61`
- `BuildModificationService.php:94`
- `FindAssignableAllocationService.php:44`

**3. 请求验证层**
- `StoreServerRequest.php:111, 120` - 验证 `allocation_id` 和 `allocation_additional` 未被占用
- `ServerFormRequest.php:39, 51` - 管理后台创建服务器时的验证

### 3.3 并发防护

**1. 行级锁** (`FindAssignableAllocationService.php:42, 106`)
```php
$server->node->allocations()
    ->lockForUpdate()  // 加排他锁
    ->where('ip', $server->allocation->ip)
    ->whereNull('server_id')
    ->first();
```
- 防止并发请求同时分配到同一个端口

**2. 事务保护**
- `ServerCreationService.php:86-94` - 服务器创建使用5次重试的事务
- `BuildModificationService.php:36-58` - 构建修改在事务中执行
- 创建失败时自动回滚，调用 `ServerDeletionService` 清理

### 3.4 业务规则校验

**参数合法性检查** (`AssignmentService.php:40-108`)：
- CIDR范围：`/25` 到 `/32`（防止过大的IP段）
- 端口范围：`1024` 到 `65535`
- 端口范围大小限制：单次最多 `1000` 个端口
- 端口格式：单个数字或 `start-end` 格式

**主分配保护** (`BuildModificationService.php:108-115`)：
- 删除主分配前必须有替代分配
- 优先使用新添加的分配作为主分配

---

## 四、回收策略

回收策略确保分配能够安全、完整地释放回资源池。

### 4.1 服务器删除时自动回收

**位置**：`app/Services/Servers/ServerDeletionService.php:59-85`

**回收流程**：
1. 数据库外键 `ON DELETE SET NULL` 自动将 `server_id` 置为 NULL
2. 应用层显式清空 `notes` 字段，防止信息泄露：
```php
$server->allocations()->update(['notes' => null]);
```

### 4.2 手动解除分配

**位置**：`app/Services/Servers/BuildModificationService.php:103-129`

**回收操作**：
```php
->update([
    'notes' => null,
    'server_id' => null,
]);
```

**保护逻辑**：
- 计算 `array_diff($remove_allocations, $add_allocations)`，避免先删后加的冲突
- 主分配被删除时自动切换到新添加的分配

### 4.3 客户端自助解除

**位置**：`app/Http/Controllers/Api/Client/Servers/NetworkAllocationController.php:119-142`

**前置检查**：
1. 必须设置 `allocation_limit`（防止无限删除）
2. 不能删除主分配

**回收操作**：同样将 `server_id` 和 `notes` 置为 NULL

### 4.4 服务器迁移时回收

**位置**：`app/Http/Controllers/Api/Remote/Servers/ServerTransferController.php:86, 124`

在迁移失败或完成时，将源服务器的分配置空：
```php
Allocation::query()->whereIn('id', $allocations)->update(['server_id' => null]);
```

### 4.5 彻底删除分配

**位置**：`app/Services/Allocations/AllocationDeletionService.php:24-31`

**前置检查**：
```php
if (!is_null($allocation->server_id)) {
    throw new ServerUsingAllocationException(trans('exceptions.allocations.server_using'));
}
```
- 只有未绑定服务器的分配才能从数据库中彻底删除
- 防止误删正在使用的分配

---

## 五、三者联动链路完整时序

### 5.1 回收释放：资源何时回到可选集合

**关键机制**：事务可见性 + 外键约束执行时机

```
时间轴 →
  │
  ├─ [T0] 事务开始（ServerDeletionService::handle）
  │    ├─ 1. 调用 Wings Daemon 删除服务器
  │    ├─ 2. 删除关联数据库（database.delete）
  │    └─ 3. 应用层清空 notes: UPDATE allocations SET notes = NULL WHERE server_id = ?
  │
  ├─ [T1] 执行 server.delete
  │    └─ 数据库触发外键约束 ON DELETE SET NULL
  │       └─ allocations.server_id 自动变为 NULL（仅当前事务可见）
  │
  ├─ [T2] 事务 COMMIT
  │    ├─ 行锁释放
  │    └─ 变更全局可见 → 此时其他事务的 whereNull('server_id') 才能看到
  │
  └─ [T3] 资源正式回归可选集合
       └─ 可被 AllocationRepository::getRandomAllocation() 查询到
```

**关键代码溯源**：
- 外键约束执行：`database/schema/mysql-schema.sql:57` - `ON DELETE SET NULL` 在 DELETE 语句执行时触发
- 事务提交点：`app/Services/Servers/ServerDeletionService.php:59-85` - `$this->connection->transaction()`
- 可见性规则：MySQL InnoDB 默认 `REPEATABLE READ` 隔离级别，事务提交前变更对其他事务不可见

**不同回收场景的时序差异**：

| 回收场景 | 触发时机 | 可见性延迟 | 数据清理 |
|---------|---------|-----------|---------|
| 服务器删除 | 事务 COMMIT 后 | 事务提交即可见 | 外键自动置空 + 应用层清空 notes |
| 手动解除分配 | UPDATE 语句执行后 | 事务提交即可见 | 应用层置空 server_id + 清空 notes |
| 客户端自助解除 | UPDATE 语句执行后 | 事务提交即可见 | 应用层置空 server_id + 清空 notes |
| 服务器迁移 | UPDATE 语句执行后 | 事务提交即可见 | 仅置空 server_id，不清空 notes |

### 5.2 占用校验：下一次分配时怎样拦截冲突

**五层拦截机制**（按执行顺序）：

```
分配请求 →
  │
  ├─ [第一层] 请求验证层（Request Validation）
  │    位置：app/Http/Requests/Admin/ServerFormRequest.php:33-55
  │    逻辑：Rule::exists('allocations', 'id')->whereNull('server_id')
  │    时机：控制器方法执行前
  │    失败：返回 422 ValidationException
  │
  ├─ [第二层] 查询条件过滤（Application Layer）
  │    位置：AllocationRepository.php:61, BuildModificationService.php:94
  │    逻辑：WHERE server_id IS NULL
  │    时机：筛选候选分配时
  │    失败：返回空集合，抛出 NoViableAllocationException
  │
  ├─ [第三层] 行级锁（Concurrency Control）
  │    位置：FindAssignableAllocationService.php:42, 106
  │    逻辑：SELECT ... FOR UPDATE (lockForUpdate)
  │    时机：选中候选分配后，更新前
  │    失败：阻塞等待锁释放，超时抛出 PDOException
  │
  ├─ [第四层] UPDATE 条件校验（Atomic Update）
  │    位置：三处 UPDATE 语句，实际逻辑不一致
  │
  │    【情况1】ServerCreationService::storeAssignedAllocations()
  │    代码：UPDATE allocations SET server_id = ? WHERE id IN (?, ?, ...)
  │    注意：❌ **没有** AND server_id IS NULL 条件！并发时会直接覆盖！
  │    位置：app/Services/Servers/ServerCreationService.php:180-182
  │
  │    【情况2】BuildModificationService::processAllocations()
  │    代码：UPDATE allocations SET server_id = ?, notes = NULL 
  │          WHERE node_id = ? AND id IN (?, ...) AND server_id IS NULL
  │    注意：✅ 有条件，但代码未检查影响行数，并发抢占成功时静默失败
  │    位置：app/Services/Servers/BuildModificationService.php:90-100
  │
  │    【情况3】FindAssignableAllocationService::handle()
  │    代码：先 WHERE server_id IS NULL 查询出模型，再 $model->update()
  │    注意：✅ 通过查询条件间接保证原子性，lockForUpdate() 防止并发
  │    位置：app/Services/Allocations/FindAssignableAllocationService.php:40-50
  │
  └─ [第五层] 数据库唯一索引（Last Resort）
       位置：database/schema/mysql-schema.sql:54
       逻辑：UNIQUE KEY (node_id, ip, port)
       时机：INSERT 或 UPDATE 时
       失败：抛出 PDOException (SQLSTATE[23000]: Duplicate entry)
```

**典型拦截流程示例**（自动部署场景）：

```
请求 POST /api/application/servers
  │
  ├─ StoreServerRequest::withValidator() 执行
  │    └─ Rule::exists()->whereNull('server_id') → 第一层拦截
  │
  ├─ ServerCreationService::handle()
  │    ├─ configureDeployment()
  │    │   ├─ FindViableNodesService → 筛选节点
  │    │   └─ AllocationSelectionService::handle()
  │    │       └─ AllocationRepository::getRandomAllocation()
  │    │           └─ WHERE server_id IS NULL → 第二层拦截
  │    │
  │    └─ 事务开始（尝试5次）
  │        ├─ createModel() → 创建服务器记录
  │        └─ storeAssignedAllocations()
  │            └─ UPDATE allocations SET server_id = ?
  │                WHERE id IN (...) → 第四层隐含校验
  │
  └─ 成功返回
```

### 5.3 并发抢占：失败时真实的异常路径、重试机制和回滚触发点

**并发冲突真实场景**：两个请求同时分配同一个端口

```
请求 A (T0)                          请求 B (T1)
   │                                    │
   ├─ 事务开始                          ├─ 事务开始
   ├─ SELECT allocation WHERE          │
   │  server_id IS NULL → 找到端口25565 │
   │                                    ├─ SELECT allocation WHERE
   │                                    │  server_id IS NULL → 也找到端口25565
   │                                    │
   │ 【情况1：使用 lockForUpdate】       │
   ├─ lockForUpdate() → 加锁            ├─ lockForUpdate() → ❌ 阻塞等待
   ├─ UPDATE server_id = A.id            │  （MySQL 默认等待50秒）
   ├─ 事务 COMMIT → 锁释放               │
   │                                    ├─ 获得锁，重新读取
   │                                    ├─ 发现 server_id 已变为 A.id
   │                                    └─ 若有 WHERE server_id IS NULL 条件
   │                                       → 影响行数 = 0，静默失败
   │                                       → 无 WHERE 条件 → 直接覆盖！
   │
   │ 【情况2：无 lockForUpdate】
   ├─ UPDATE server_id = A.id            ├─ UPDATE server_id = B.id
   │  (无 WHERE server_id IS NULL)       │  (无 WHERE server_id IS NULL)
   ├─ 事务 COMMIT                        ├─ 事务 COMMIT
   └─ 成功                              └─ ❗ 后提交的覆盖先提交的！
                                        （无异常，数据不一致）
```

**重试机制：当前实现的真实边界**

1. **事务自动重试**（`ServerCreationService.php:86`）
   ```php
   $server = $this->connection->transaction(function () use ($data, $eggVariableData) {
       // ... 业务逻辑 ...
   }, 5); // 重试5次
   ```
   - ✅ Laravel 事务 `$attempts` 参数只捕获**死锁（错误码 1213）**
   - ❌ **不捕获**锁等待超时（错误码 1205）或其他 PDOException
   - ❌ 不处理 UPDATE 影响行数为 0 的业务逻辑失败
   - 每次重试都会重新开始整个事务，重新筛选分配

2. **死锁异常处理**（`app/Exceptions/Handler.php:137-139`）
   ```php
   if ($connections->transactionLevel()) {
       $connections->rollBack(0); // 强制回滚到最外层
   }
   ```
   - 全局异常处理器检查未完成事务
   - 任何异常未被捕获时，强制回滚所有层级
   - 确保数据库连接状态干净，避免悬挂事务

3. **创建失败补偿回滚**（`ServerCreationService.php:100-104`）
   ```php
   try {
       $this->daemonServerRepository->setServer($server)->create(...);
   } catch (DaemonConnectionException $exception) {
       $this->serverDeletionService->withForce()->handle($server);
       throw $exception;
   }
   ```
   - ✅ Wings Daemon 创建失败时，显式调用 `ServerDeletionService` 回滚
   - ✅ 服务器删除触发外键 `ON DELETE SET NULL`，自动释放分配
   - ✅ `withForce()` 忽略 Daemon 错误，确保数据一致性
   - ❌ 但此回滚仅发生在事务**提交后**，不是事务内回滚

4. **并发更新失败检测：当前实现的缺陷**
   - `BuildModificationService` 中 UPDATE 虽有 `WHERE server_id IS NULL` 条件
   - ❌ **代码未检查影响行数**，并发抢占成功时影响行数=0，但静默失败
   - `ServerCreationService` 中 UPDATE **没有** `WHERE server_id IS NULL` 条件
   - ❌ 并发时会直接覆盖已分配的 allocation，无任何异常
   - `FindAssignableAllocationService` 中有 `lockForUpdate()` + 查询条件
   - ✅ 这是三处中唯一能正确防止并发覆盖的

**真实的异常路径**：
```
并发抢占 → 静默失败/数据覆盖 → 无异常抛出
          ↗            ↖
    有WHERE条件     无WHERE条件
  (影响行数=0)   (直接覆盖)
  无异常抛出    无异常抛出

只有以下情况会抛出异常：
→ 死锁（错误码1213）→ 触发Laravel事务重试
→ 锁等待超时（错误码1205）→ 抛出PDOException，不重试
→ 唯一键冲突（错误码23000）→ 插入重复分配时抛出
```

---

## 六、三者协作关系

### 6.1 整体流程

```
候选筛选 → 占用校验 → 分配绑定 → 使用中 → 回收释放
    ↓          ↓          ↓                    ↓
1. 节点筛选   1. 请求验证  1. 事务保护        1. 外键自动置空
2. 分配筛选   2. 查询过滤  2. ⚠️ 仅部分场景加行锁  2. 主动置空 server_id
3. 专用IP检查  3. ⚠️ 仅部分场景加行锁  3. ⚠️ 三处UPDATE逻辑不一致  3. ⚠️ 部分场景不清空 notes
             4. ⚠️ 仅两处有UPDATE条件  4. 事务提交可见
             5. 唯一索引    ⚠️ 无影响行数检查
```

### 6.2 典型场景协作

#### 场景1：自动部署服务器

**完整时序**：
```
T0  请求验证层：Rule::exists()->whereNull('server_id') → 第一层拦截
T1  FindViableNodesService → 节点筛选（内存/磁盘/位置）
T2  AllocationSelectionService::handle()
    └─ AllocationRepository::getRandomAllocation()
       └─ WHERE server_id IS NULL → 第二层拦截
T3  事务开始（第1次尝试，最多5次死锁重试）
T4    createModel() → INSERT servers
T5    storeAssignedAllocations()
      └─ UPDATE allocations SET server_id = ? WHERE id IN (...)
         ⚠️  注意：没有 AND server_id IS NULL 条件！并发时可能覆盖！
T6  事务 COMMIT → 变更可见（无显式行锁，依赖请求层验证+事务重试）
T7  调用 Wings Daemon 创建服务器
    ├─ 成功 → 返回
    └─ 失败 → ServerDeletionService::withForce()->handle() → 补偿回滚
         （删除服务器 → 外键自动置空分配）
```

**涉及文件**：
- `app/Services/Servers/ServerCreationService.php:56-60, 86-94, 100-104, 116-128, 180-182`

#### 场景2：客户端自助添加分配

**完整时序**：
```
T0  NetworkAllocationController::store()
    └─ Activity::event()->transaction()
T1    lockForUpdate() 计数校验 → 限额检查
T2    FindAssignableAllocationService::handle()
        ├─ lockForUpdate() → WHERE server_id IS NULL → 第二+三层拦截
        ├─ 找到 → UPDATE server_id = ? → 第四层校验
        └─ 找不到 → 差集计算 → insertIgnore → UPDATE
T3  事务 COMMIT → 锁释放
T4  返回
```

**涉及文件**：
- `app/Http/Controllers/Api/Client/Servers/NetworkAllocationController.php:95-112`
- `app/Services/Allocations/FindAssignableAllocationService.php:31-53`

#### 场景3：修改服务器构建（增减分配）

**完整时序**：
```
T0  事务开始
T1    processAllocations()
      ├─ 添加：WHERE server_id IS NULL → UPDATE server_id = ?
      ├─ 移除：主分配保护检查 → UPDATE server_id = NULL, notes = NULL
      └─ 冲突处理：array_diff(remove, add) 避免先删后加
T2    更新 server.allocation_id（如果主分配变更）
T3    服务器模型 saveOrFail()
T4  事务 COMMIT
T5  同步到 Wings Daemon（失败仅记录日志，不回滚）
```

**涉及文件**：
- `app/Services/Servers/BuildModificationService.php:82-130`

### 6.3 关键协作点

| 机制 | 筛选阶段 | 校验阶段 | 回收阶段 |
|------|---------|---------|---------|
| **数据库约束** | - | 唯一索引、外键 | ON DELETE SET NULL |
| **应用层检查** | whereNull('server_id') | whereNull('server_id')<br>⚠️ 三处UPDATE逻辑不一致 | server_id 非空检查 |
| **并发控制** | - | lockForUpdate()<br>⚠️ 仅自动分配和限额检查使用 | - |
| **事务保护** | - | 创建/修改事务（仅死锁重试5次）<br>⚠️ 锁等待超时不重试 | 删除事务 |
| **数据清理** | - | - | 清空 notes<br>⚠️ 服务器迁移不清空 |
| **可见性规则** | 读已提交快照 | 事务内立即可见 | 提交后全局可见 |
| **失败回滚** | - | 异常时自动回滚<br>⚠️ 静默失败不回滚 | 外键自动置空 |

### 6.4 端到端完整链路时序

```
回收链路                                分配链路
──────────                              ┌───────────────┐
│ T0 事务开始                          │ 请求到达      │
│ T1 清空 notes UPDATE                 │ 层验证        │
│ T2 删除 server                       │ 节点筛选      │
│    → 外键 ON DELETE SET NULL         │ 分配筛选      │
│      (仅当前事务可见)                │ 事务开始      │
│ T3 事务 COMMIT                       │ 行锁加锁      │
│    → 锁释放                          │ UPDATE 校验   │
│    → 变更全局可见                    │ 事务 COMMIT   │
│ T4 资源回归可选集合 ◄───────────────► 可被查询到     │
└──────────                              └───────────────┘

并发冲突场景：
请求 A: SELECT → 加锁 → UPDATE → COMMIT
请求 B: SELECT → 阻塞 → 获得锁 → 发现已变更 → 回滚 → 重试
```

---

## 七、设计亮点与潜在风险

### 设计亮点

1. **多层防御**：5层拦截（请求验证→查询过滤→行锁→UPDATE条件→唯一索引），整体形成防护网
2. **自动回收**：外键 `ON DELETE SET NULL` + 应用层双保险，确保服务器删除时分配自动释放
3. **随机分配**：`inRandomOrder()` 避免端口集中在某一区域，减少碎片
4. **幂等操作**：`insertIgnore` 支持重复调用，适合批量场景
5. **资源隔离**：专用IP模式确保敏感应用独占IP
6. **死锁重试**：服务器创建支持5次事务重试，应对死锁（仅错误码1213）
7. **全局回滚**：异常处理器强制回滚未完成事务，防止悬挂锁

### 潜在风险（经代码验证的真实缺陷）

1. **⚠️ 并发覆盖风险**：`ServerCreationService::storeAssignedAllocations()` 的 UPDATE 没有 `AND server_id IS NULL` 条件，高并发下可能直接覆盖已分配的端口，无任何异常
   - 位置：`app/Services/Servers/ServerCreationService.php:180-182`
   - 影响：管理员从后台创建服务器时存在并发冲突窗口

2. **⚠️ 静默失败风险**：`BuildModificationService::processAllocations()` 的 UPDATE 虽有条件，但未检查影响行数，并发抢占成功时（影响行数=0）静默失败
   - 位置：`app/Services/Servers/BuildModificationService.php:100`
   - 影响：用户以为分配成功了，实际分配被别人抢走了

3. **⚠️ 重试机制不完整**：Laravel 事务重试（`$attempts = 5`）仅捕获死锁（错误码1213），不捕获锁等待超时（错误码1205）或业务逻辑失败

4. **长事务风险**：`FindAssignableAllocationService` 中的 `lockForUpdate()` 可能导致锁等待，高并发下需注意事务长度

5. **差集计算性能**：`array_diff(range($start, $end), $ports->toArray())` 在端口范围大于10000时可能有性能问题

6. **Notes 泄露**：服务器迁移场景仅置空 `server_id`，未清空 `notes`，可能携带前一服务器的敏感信息

7. **事务隔离级别**：MySQL 默认 `REPEATABLE READ` 可能导致幻读，极端并发下需注意

### 关键代码溯源

| 功能 | 文件位置 | 行号 |
|------|---------|------|
| 节点资源筛选 | `FindViableNodesService.php` | 74-86 |
| 分配随机获取 | `AllocationRepository.php` | 59-98 |
| 自动分配逻辑（带行锁） | `FindAssignableAllocationService.php` | 31-112 |
| 插入忽略重复 | `EloquentRepository.php` | 251-277 |
| 构建修改分配（带WHERE条件，无行数检查） | `BuildModificationService.php` | 82-130 |
| 服务器删除回收 | `ServerDeletionService.php` | 59-85 |
| 服务器创建分配（⚠️ 无WHERE条件） | `ServerCreationService.php` | 180-182 |
| 死锁重试（仅错误码1213） | `ServerCreationService.php` | 86 |
| 创建失败补偿回滚 | `ServerCreationService.php` | 100-104 |
| 全局异常强制回滚 | `Handler.php` | 137-139 |
| 请求验证层 | `ServerFormRequest.php` | 33-55 |
| 行锁加锁 | `FindAssignableAllocationService.php` | 42, 106 |
| 数据库唯一键 | `mysql-schema.sql` | 54 |
| 外键置空约束 | `mysql-schema.sql` | 57 |
