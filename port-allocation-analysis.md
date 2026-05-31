# 端口与地址分配机制——冲突避免策略代码分析报告

> 分析对象：Pterodactyl Panel（`180-panel`）  
> 分析日期：2026-05-31  
> 涉及核心模块：Allocation Model / Node Model / Server Model / ServerTransfer Model / Deployment Services / Wings Daemon Repositories

---

## 目录

1. [总体架构概览](#1-总体架构概览)
2. [分配策略的存储模型](#2-分配策略的存储模型)
3. [节点容量约束关系](#3-节点容量约束关系)
4. [新建实例——端口/地址分配流程与冲突避免](#4-新建实例端口地址分配流程与冲突避免)
5. [实例迁移——端口/地址再分配与冲突检测](#5-实例迁移端口地址再分配与冲突检测)
   5.5 [迁移状态机与时序详解——目标分配被占用后的走向](#55-迁移状态机与时序详解目标分配被占用后的走向)
6. [实例释放——端口/地址回收机制](#6-实例释放端口地址回收机制)
7. [运行时分配变更——Build Modification 与用户自服务](#7-运行时分配变更build-modification-与用户自服务)
8. [面板与守护层之间的协议](#8-面板与守护层之间的协议)
9. [冲突检测与回滚路径汇总](#9-冲突检测与回滚路径汇总)
10. [风险点与改进建议](#10-风险点与改进建议)

---

## 1. 总体架构概览

本系统采用 **Panel–Wings** 双层架构：

| 层次 | 职责 | 技术栈 |
|------|------|--------|
| **Panel（面板层）** | 资源元数据管理、分配决策、持久化状态机 | Laravel (PHP)，MySQL |
| **Wings（守护层）** | 容器生命周期、端口实际绑定、文件系统操作 | Go，Docker API |

关键设计原则：**Panel 是分配的唯一决策者（Single Source of Truth），Wings 仅执行 Panel 下发的配置**。Wings 在启动服务器时会从 Panel 拉取最新配置（sync），因此即使 Panel→Wings 的推送失败，下次服务器启动也会自动收敛到一致状态。

---

## 2. 分配策略的存储模型

### 2.1 allocations 表——核心存储

```sql
CREATE TABLE allocations (
  id             INT UNSIGNED NOT NULL AUTO_INCREMENT,
  node_id        INT UNSIGNED NOT NULL,
  ip             VARCHAR(191) NOT NULL,
  ip_alias       TEXT DEFAULT NULL,
  port           MEDIUMINT UNSIGNED NOT NULL,
  server_id      INT UNSIGNED DEFAULT NULL,    -- 占用标记
  notes          VARCHAR(191) DEFAULT NULL,
  created_at     TIMESTAMP NULL,
  updated_at     TIMESTAMP NULL,
  PRIMARY KEY (id),
  UNIQUE KEY allocations_node_id_ip_port_unique (node_id, ip, port),
  CONSTRAINT allocations_node_id_foreign  FOREIGN KEY (node_id)   REFERENCES nodes(id)    ON DELETE CASCADE,
  CONSTRAINT allocations_server_id_foreign FOREIGN KEY (server_id) REFERENCES servers(id) ON DELETE SET NULL
);
```

**设计要点：**

| 字段 | 语义 | 冲突避免作用 |
|------|------|-------------|
| `(node_id, ip, port)` UNIQUE | 同一节点同一IP的同一端口只能存在一条记录 | **数据库级硬约束**，杜绝重复注册 |
| `server_id DEFAULT NULL` | 空值=空闲，非空=被占用 | **占用标记位**，所有分配查询的过滤条件 |
| `server_id → servers.id ON DELETE SET NULL` | 服务器删除时自动释放分配 | **级联回收**，无需手动清理 |

### 2.2 servers 表——默认分配绑定

```sql
allocation_id  INT UNSIGNED NOT NULL,
UNIQUE KEY servers_allocation_id_unique (allocation_id),
CONSTRAINT servers_allocation_id_foreign FOREIGN KEY (allocation_id) REFERENCES allocations(id)
```

`servers.allocation_id` 是 1:1 唯一约束，确保一个分配只能是一个服务器的默认分配。

### 2.3 server_transfers 表——迁移状态记录

```sql
CREATE TABLE server_transfers (
  id                       INT UNSIGNED NOT NULL AUTO_INCREMENT,
  server_id                INT UNSIGNED NOT NULL,
  successful               TINYINT(1) DEFAULT NULL,    -- NULL=进行中, true=成功, false=失败
  old_node                 INT UNSIGNED NOT NULL,
  new_node                 INT UNSIGNED NOT NULL,
  old_allocation           INT UNSIGNED NOT NULL,
  new_allocation           INT UNSIGNED NOT NULL,
  old_additional_allocations LONGTEXT DEFAULT NULL,     -- JSON数组
  new_additional_allocations LONGTEXT DEFAULT NULL,     -- JSON数组
  archived                 TINYINT(1) NOT NULL DEFAULT 0,
  ...
);
```

`successful = NULL` 表示迁移正在进行中，用于 **Server.validateTransferState()** 判断是否存在活跃迁移。

### 2.4 nodes 表——容量定义

```sql
memory                INT UNSIGNED NOT NULL,
memory_overallocate   INT NOT NULL DEFAULT 0,   -- 百分比
disk                  INT UNSIGNED NOT NULL,
disk_overallocate     INT NOT NULL DEFAULT 0,   -- 百分比
```

### 2.5 insertIgnore——批量注册时的幂等保障

`AllocationRepository::insertIgnore()`（底层调用 `EloquentRepository::insertIgnore()`）使用 SQL `INSERT IGNORE` 语句，当遇到 `(node_id, ip, port)` 唯一键冲突时静默跳过而非报错。这在管理员批量添加端口段时避免因已有端口而中断整个操作。

**源码位置：** `app/Services/Allocations/AssignmentService.php:103`

---

## 3. 节点容量约束关系

### 3.1 端口范围约束

端口范围由 `AssignmentService` 的常量定义：

| 常量 | 值 | 含义 |
|------|----|------|
| `PORT_FLOOR` | 1024 | 最小可用端口（排除系统保留端口） |
| `PORT_CEIL` | 65535 | 最大可用端口 |
| `PORT_RANGE_LIMIT` | 1000 | 单次批量添加端口数量上限 |
| `CIDR_MAX_BITS` | 25 | CIDR最大子网（更多主机） |
| `CIDR_MIN_BITS` | 32 | CIDR最小子网（单IP） |

用户自服务自动分配的端口范围由配置文件控制：

```php
// config/pterodactyl.php
'allocations' => [
    'enabled'     => env('PTERODACTYL_CLIENT_ALLOCATIONS_ENABLED', false),
    'range_start' => env('PTERODACTYL_CLIENT_ALLOCATIONS_RANGE_START'),
    'range_end'   => env('PTERODACTYL_CLIENT_ALLOCATIONS_RANGE_END'),
],
```

### 3.2 内存与磁盘约束

`FindViableNodesService::handle()` 构建的 SQL：

```sql
SELECT nodes.*,
       IFNULL(SUM(servers.memory), 0) AS sum_memory,
       IFNULL(SUM(servers.disk), 0)   AS sum_disk
FROM nodes
LEFT JOIN servers ON servers.node_id = nodes.id
WHERE nodes.public = 1
GROUP BY nodes.id
HAVING (sum_memory + ?) <= (nodes.memory * (1 + nodes.memory_overallocate / 100))
   AND (sum_disk + ?)   <= (nodes.disk * (1 + nodes.disk_overallocate / 100))
```

**约束公式：**
```
已分配内存 + 新增内存 ≤ 节点内存 × (1 + 超分配百分比/100)
已分配磁盘 + 新增磁盘 ≤ 节点磁盘 × (1 + 超分配百分比/100)
```

`memory_overallocate` / `disk_overallocate` 默认值为 0（不允许超分配），设为 -1 表示无限制。

**源码位置：** `app/Services/Deployment/FindViableNodesService.php:74-86`

### 3.3 分配数量约束（allocation_limit）

`servers.allocation_limit` 字段限制单个服务器可拥有的最大分配数。在用户自服务添加分配时检查：

```php
// NetworkAllocationController::store()
if ($server->allocations()->lockForUpdate()->count() >= $server->allocation_limit) {
    throw new DisplayException('Cannot assign additional allocations: limit reached.');
}
```

**源码位置：** `app/Http/Controllers/Api/Client/Servers/NetworkAllocationController.php:98`

---

## 4. 新建实例端口/地址分配流程与冲突避免

### 4.1 手动指定分配

当管理员直接指定 `allocation_id` 时：

1. `Server::$validationRules` 中 `allocation_id` 的验证规则为 `required|bail|unique:servers|exists:allocations,id`——确保该分配尚未被任何服务器作为默认分配
2. `ServerCreationService::handle()` 自动从分配推导 `node_id`：如果 `node_id` 为空则从 `Allocation::findOrFail($allocation_id)->node_id` 获取
3. 事务内调用 `storeAssignedAllocations()` 批量设置 `server_id`

```php
// ServerCreationService::storeAssignedAllocations()
Allocation::query()->whereIn('id', $records)->update(['server_id' => $server->id]);
```

**冲突避免：**
- `servers.allocation_id` 的 UNIQUE 约束保证同一分配不会被两个服务器同时作为默认分配
- `allocations.server_id` 的更新在数据库事务内完成
- 若分配已被占用（`server_id` 非 NULL），UPDATE 语句仍会覆盖——**依赖前置验证而非乐观锁**

### 4.2 自动部署（DeploymentObject）

当传入 `DeploymentObject` 时，分配流程如下：

```
ServerCreationService::handle()
  └─ configureDeployment()
       ├─ FindViableNodesService::handle()        → 按内存/磁盘过滤可行节点
       └─ AllocationSelectionService::handle()     → 从可行节点中选择分配
            └─ AllocationRepository::getRandomAllocation()
                 ├─ WHERE server_id IS NULL         → 排除已占用
                 ├─ WHERE node_id IN (...)          → 限定节点
                 ├─ WHERE port IN / BETWEEN         → 限定端口
                 └─ 专用IP过滤（dedicated mode）    → 排除已有其他服务器的IP
```

**专用IP（Dedicated IP）模式：** 当 `DeploymentObject.dedicated = true` 时，`getDiscardableDedicatedAllocations()` 查询所有已有服务器绑定的 `(node_id, ip)` 组合，然后在选择分配时排除这些 IP 上的所有端口，确保该服务器独占一个 IP。

**源码位置：** `app/Repositories/Eloquent/AllocationRepository.php:41-54`

### 4.3 回滚路径

```php
// ServerCreationService::handle() — 第86-107行
$server = $this->connection->transaction(function () use ($data, $eggVariableData) {
    $server = $this->createModel($data);
    $this->storeAssignedAllocations($server, $data);
    $this->storeEggVariables($server, $eggVariableData);
    return $server;
}, 5);  // 5次死锁重试

try {
    $this->daemonServerRepository->setServer($server)->create(...);
} catch (DaemonConnectionException $exception) {
    $this->serverDeletionService->withForce()->handle($server);  // 回滚：强制删除
    throw $exception;
}
```

**回滚策略：**
- Panel侧数据库操作在事务内完成，失败自动回滚
- Wings创建失败时，调用 `ServerDeletionService::withForce()` 强制删除——包括释放分配、删除数据库记录
- `withForce()` 使得即使 Wings 删除也失败（如 404），Panel 侧数据仍会被清理

---

## 5. 实例迁移端口/地址再分配与冲突检测

### 5.1 迁移发起

**源码位置：** `app/Http/Controllers/Admin/Servers/ServerTransferController.php`

```
transfer()
  ├─ 验证 allocation_id: required|bail|unique:servers|exists:allocations,id
  ├─ Node.isViable(memory, disk)  → 检查目标节点容量
  ├─ Server.validateTransferState() → 检查是否已在迁移/未安装
  └─ 事务内:
       ├─ 创建 ServerTransfer 记录（successful=NULL）
       ├─ assignAllocationsToServer() → 预占新分配
       └─ DaemonTransferRepository::notify() → 通知源节点
```

**预占机制（关键冲突避免）：** `assignAllocationsToServer()` 在迁移开始时就将目标节点的新分配绑定到 `server_id`，防止其他实例在迁移过程中抢走这些分配：

```php
private function assignAllocationsToServer(Server $server, int $node_id, int $allocation_id, array $additional_allocations) {
    $unassigned = $this->allocationRepository->getUnassignedAllocationIds($node_id);
    $updateIds = [];
    foreach ($allocations as $allocation) {
        if (!in_array($allocation, $unassigned)) continue;  // 跳过已被占用的
        $updateIds[] = $allocation;
    }
    if (!empty($updateIds)) {
        $this->allocationRepository->updateWhereIn('id', $updateIds, ['server_id' => $server->id]);
    }
}
```

**注意：** 如果请求的分配已被占用，这里只是**静默跳过**而非报错——这是一个潜在问题（见第10节）。

### 5.2 迁移成功

**源码位置：** `app/Http/Controllers/Api/Remote/Servers/ServerTransferController::success()`

由**新节点**的 Wings 调用，Panel 执行：

```php
$this->connection->transaction(function () use ($server, $transfer) {
    // 1. 释放旧分配
    Allocation::query()->whereIn('id', $old_allocations)->update(['server_id' => null]);
    
    // 2. 更新服务器指向新节点和新默认分配
    $server->update([
        'allocation_id' => $transfer->new_allocation,
        'node_id' => $transfer->new_node,
    ]);
    
    // 3. 标记迁移成功
    $server->transfer->update(['successful' => true]);
});

// 4. 通知旧节点删除实例
$this->daemonServerRepository->setServer($server)->setNode($transfer->oldNode)->delete();
```

### 5.3 迁移失败

**源码位置：** `app/Http/Controllers/Api/Remote/Servers/ServerTransferController::failure()`

源节点或目标节点均可报告失败：

```php
protected function processFailedTransfer(ServerTransfer $transfer): JsonResponse {
    $this->connection->transaction(function () use (&$transfer) {
        $transfer->forceFill(['successful' => false])->saveOrFail();
        
        // 释放新节点上预占的分配
        $allocations = array_merge([$transfer->new_allocation], $transfer->new_additional_allocations);
        Allocation::query()->whereIn('id', $allocations)->update(['server_id' => null]);
    });
    return new JsonResponse([], Response::HTTP_NO_CONTENT);
}
```

**回滚路径：** 迁移失败时，**只释放新节点上的分配**，旧节点上的分配绑定不受影响——服务器继续在旧节点运行。

### 5.4 迁移中的状态隔离

`Server::validateCurrentState()` 在每次用户操作前检查是否存在活跃迁移：

```php
public function validateCurrentState() {
    if (!is_null($this->transfer)) {  // transfer关系查询 successful IS NULL
        throw new ServerStateConflictException($this);
    }
}
```

`ServerStateConflictException` 返回 HTTP 409 Conflict，消息为 "This server is currently being transferred to a new machine, please try again later."

### 5.5 迁移状态机与时序详解——目标分配被占用后的走向

**核心发现：Wings notify 在数据库事务内调用。** 如果 HTTP 调用失败抛出异常，整个事务回滚——包括 ServerTransfer 记录和分配预占。这是一个"全部成功或全部失败"的原子操作边界。

#### 5.5.1 完整时序与状态分支

```
初始状态（server_transfers 无 successful IS NULL 记录）
    │
    │ [管理员发起迁移]
    ▼
┌──────────────────────────────────────────────────────────────┐
│  前置验证（事务外）                                           │
│  1. allocation_id: unique:servers → 检查是否是其他服务器的默认分配 │
│  2. allocation_id: exists:allocations → 检查分配记录是否存在     │
│  3. Node.isViable(memory, disk) → 目标节点容量检查              │
│  4. Server.validateTransferState() → 无活跃迁移                 │
└──────────────────────────────┬─────────────────────────────────┘
                               │
                  ┌────────────┴────────────┐
                  │ 验证失败                  │ 验证通过
                  ▼                           ▼
             重定向+错误提示          ┌──────────────────────────────────┐
             【可直接重试】           │  启动数据库事务                │
                                     │  ┌────────────────────────────┐ │
                                     │  │ 1. INSERT server_transfers │ │
                                     │  │    successful = NULL       │ │
                                     │  └────────────────────────────┘ │
                                     │  ┌────────────────────────────┐ │
                                     │  │ 2. assignAllocationsToServ…│ │
                                     │  │    查询 getUnassignedAlloc… │ │
                                     │  │    不在列表中的 → 静默跳过   │ │
                                     │  │    在列表中的 → UPDATE      │ │
                                     │  │      server_id = server.id  │ │
                                     │  └────────────────────────────┘ │
                                     │  ┌────────────────────────────┐ │
                                     │  │ 3. 生成 JWT (15分钟过期)   │ │
                                     │  └────────────────────────────┘ │
                                     │  ┌────────────────────────────┐ │
                                     │  │ 4. Wings notify()          │ │
                                     │  │    POST /api/servers/{uuid}/│ │
                                     │  │    transfer                │ │
                                     │  │    ⚠️ HTTP 请求在事务内!    │ │
                                     │  └────────────────────────────┘ │
                                     └─────────────────┬──────────────┘
                                                       │
                                      ┌────────────────┴─────────────────┐
                                      │ Wings 异常（网络/5xx/超时等）      │ Wings 200 OK
                                      ▼                                    ▼
                                  事务自动回滚                          事务提交
                                  ────────────                      ──────────────────
                                  • ServerTransfer 回滚（无记录）         • ServerTransfer 持久化
                                  • 分配 UPDATE 回滚（server_id 复原）    • 部分/全部分配已绑定 server_id
                                  • JWT 作废（未被 Wings 接收）          • 服务器进入"迁移中"状态
                                  • 【可直接重试】                         • Server::transfer() 非空
                                                                           │
                                                                           ▼
                                                                   Wings 侧执行迁移流程
                                                                   （归档 → 传输 → 恢复）
                                                                           │
                                                      ┌────────────────────┴────────────────────┐
                                                      │ 迁移失败（任何阶段）                        │ 迁移成功
                                                      ▼                                           ▼
                                          POST /api/remote/servers/{uuid}/transfer/failure    POST /api/remote/servers/{uuid}/transfer/success
                                          ──────────────────────────────────────────────      ────────────────────────────────────────────────
                                          1. 认证：仅新旧节点可上报失败                              1. 认证：仅新节点可上报成功
                                          2. 事务内：                                                  2. 事务内：
                                             ServerTransfer.successful = false                            释放旧分配（server_id = NULL）
                                          3. 事务内：                                                  更新 server.node_id = new_node
                                             释放新节点预占分配：                                        更新 server.allocation_id = new_allocation
                                             new_allocation + new_additional_allocations                ServerTransfer.successful = true
                                             → server_id = NULL                                     3. 事务外：
                                          4. 返回 204 No Content                                         尝试 DELETE 旧节点上的实例（异常仅记录日志）
                                          • 旧分配仍绑定服务器（状态不变）                                • 返回 204 No Content
                                          • 预占已清理                                                     旧节点上可能残留"幽灵实例"
                                          • 【可直接重试】                                               • 【需要人工检查：旧节点清理结果】
```

#### 5.5.2 关键时序点的原子性分析

| 操作 | 原子边界 | 失败影响 |
|------|---------|---------|
| ServerTransfer INSERT | 数据库事务 | 回滚，无残留 |
| assignAllocationsToServer | 数据库事务 | 回滚，分配占用状态恢复 |
| Wings notify | **在事务内同步调用** | 抛出异常 → 事务回滚，全部复原 |
| 事务 COMMIT | 数据库事务 | 以上所有变更持久化 |
| 迁移失败回调 | 独立数据库事务 | 只清理新节点预占，旧节点不受影响 |
| 迁移成功回调 | 独立数据库事务 | 切换服务器绑定，随后的旧节点删除是"尽力而为" |

**⚠️ 注意：事务内发起 HTTP 请求是反模式。** 如果 Wings 接收了请求但响应超时，会出现：
- Panel 侧事务回滚（认为迁移未发起）
- Wings 侧已收到迁移指令并执行
→ **造成脑裂**，需要人工干预。

#### 5.5.3 目标分配被占用的四种场景与可重试性

根据 `assignAllocationsToServer()` 的静默跳过逻辑，目标分配被占用时走向分四种：

**场景 A：默认分配（new_allocation）被占用**

```php
$unassigned = $this->allocationRepository->getUnassignedAllocationIds($node_id);
if (!in_array($allocation_id, $unassigned)) {
    continue;  // 静默跳过！
}
```

- 状态走向：事务提交 → ServerTransfer 存在但默认分配未预占 → Wings 拉取配置时发现该分配 server_id ≠ 本服务器 → 迁移失败回调 → 释放新分配（但本来就没预占）→ successful = false
- **可直接重试：是**
- **判断依据：** 失败回调已执行（successful = false），新分配预占已清理，旧分配完好

**场景 B：部分附加分配被占用（默认分配空闲）**

- 状态走向：部分附加分配预占成功，部分跳过 → 事务提交 → Wings 收到迁移指令 → 实际可用分配比预期少 → 可能成功也可能失败（取决于 Egg 是否强制需要所有端口）
- **可直接重试：需先检查 ServerTransfer.successful**
  - successful = NULL：进行中，**不可重试**，等待回调或超时
  - successful = false：已失败，**可重试**
  - successful = true：已成功但分配缺失，**需要人工补充分配**
- **判断依据：** `Server::transfer()` 关系是否非空（查询 successful IS NULL）

**场景 C：Wings notify 超时后目标分配被他人抢占**

- 状态走向：Panel 侧事务回滚（认为未发起）→ 管理员重试 → 第二次验证时发现分配已被占用 → 验证失败
- **可直接重试：是（换其他分配）**
- **判断依据：** 无 ServerTransfer 记录，可直接重新选择分配

**场景 D：迁移成功后旧节点删除失败**

```php
// success() 回调中
try {
    $this->daemonServerRepository->setServer($server)->setNode($transfer->oldNode)->delete();
} catch (DaemonConnectionException $exception) {
    Log::warning($exception, ['transfer_id' => $server->transfer->id]);
}
```

- 状态走向：Panel 侧已全部切换成功 → 旧节点 Wings 删除请求失败（仅记日志）→ 旧节点上残留"幽灵实例"
- **可直接重试：否**
- **需要人工干预：** 检查旧节点实例状态，手动删除
- **判断依据：** ServerTransfer.successful = true，但旧节点 Wings 上仍有该 UUID 的实例

#### 5.5.4 可重试 vs 需人工干预的判断矩阵

| 状态（ServerTransfer + 分配绑定） | 可直接重试？ | 判断依据 | 操作建议 |
|----------------------------------|------------|---------|---------|
| **无 ServerTransfer 记录** | ✅ 是 | 上次事务回滚或从未发起 | 重新选择分配后重试 |
| **successful = NULL + 所有新分配 server_id = 目标服务器** | ❌ 否 | 迁移正在 Wings 侧进行 | 等待回调（最长约30分钟，取决于 Wings 超时设置） |
| **successful = NULL + 部分新分配 server_id 为 NULL** | ⚠️ 谨慎 | 预占不完整，可能 Wing 侧也会失败 | 建议等待失败回调，不要强制重试 |
| **successful = false** | ✅ 是 | 失败回调已完成清理 | 更换分配后重试 |
| **successful = true + 旧节点实例已删除** | ✅ 已完成 | 迁移全流程成功 | 无需操作 |
| **successful = true + 旧节点实例仍存在** | ❌ 否 | Panel 侧成功但 Wings 侧残留 | 人工登录旧节点手动删除实例 |
| **successful = NULL + 超过1小时无回调** | ⚠️ 需人工 | Wings 可能崩溃或网络中断，状态未知 | 检查两个节点的实例状态，手动设置 successful = false 并清理预占 |

#### 5.5.5 超时无回调的手动恢复步骤

当 `successful = NULL` 超过合理时间（如 1 小时）且无任何回调时：

1. 检查源节点 Wings：`GET /api/servers/{uuid}` 是否存在，状态是否在迁移中
2. 检查目标节点 Wings：`GET /api/servers/{uuid}` 是否存在
3. 若两端都不存在或只有源节点存在：
   - 手动执行 `UPDATE server_transfers SET successful = false WHERE id = ?`
   - 手动释放新节点分配：`UPDATE allocations SET server_id = NULL WHERE id IN (新分配列表)`
4. 若目标节点已存在：
   - 视为成功，手动执行成功回调的 SQL 更新
   - 手动从源节点删除实例

---

## 6. 实例释放端口/地址回收机制

**源码位置：** `app/Services/Servers/ServerDeletionService.php`

```
handle(Server $server)
  ├─ Wings删除实例（允许404）
  └─ 事务内:
       ├─ 逐个删除关联数据库
       ├─ allocations()->update(['notes' => null])  → 清空备注
       └─ $server->delete()  → 删除服务器记录
```

**回收机制核心：** `allocations` 表的 `server_id` 外键定义为 `ON DELETE SET NULL`。当 `servers` 表的记录被删除时，MySQL 自动将关联分配的 `server_id` 设为 NULL，分配回到可用池。

额外地，代码显式清空 `notes`（`allocations()->update(['notes' => null])`），防止旧备注信息泄露到后续分配该端口的服务器。

**强制删除（withForce）：** 当 Wings 通信失败时，`withForce()` 允许忽略守护层错误继续删除 Panel 侧数据。404 响应（实例在 Wings 上不存在）始终被忽略。

---

## 7. 运行时分配变更——Build Modification 与用户自服务

### 7.1 BuildModificationService

**源码位置：** `app/Services/Servers/BuildModificationService.php`

**添加分配：**
```php
Allocation::query()
    ->where('node_id', $server->node_id)      // 必须同节点
    ->whereIn('id', $data['add_allocations'])
    ->whereNull('server_id')                   // 必须空闲
    ->update(['server_id' => $server->id, 'notes' => null]);
```

**移除分配：**
```php
Allocation::query()->where('node_id', $server->node_id)
    ->where('server_id', $server->id)
    ->whereIn('id', array_diff($data['remove_allocations'], $data['add_allocations'] ?? []))
    ->update(['notes' => null, 'server_id' => null]);
```

**默认分配切换保护：** 如果要删除的是当前默认分配，必须有新添加的分配作为回退，否则抛出 `DisplayException`。

**Wings同步：** 变更后调用 `daemonServerRepository->sync()` 推送新配置到 Wings。即使同步失败也仅记录日志，不回滚 Panel 侧变更——因为 Wings 下次启动服务器时会自动拉取最新配置。

### 7.2 用户自服务分配（FindAssignableAllocationService）

**源码位置：** `app/Services/Allocations/FindAssignableAllocationService.php`

```
handle(Server $server)
  ├─ 检查 client_features.allocations.enabled
  ├─ 查找同节点同IP的空闲分配 (lockForUpdate)
  │    └─ 找到 → 直接绑定
  └─ 未找到 → createNewAllocation()
       ├─ 从配置的 range_start/range_end 确定端口范围
       ├─ 查询节点已有端口，计算差集得到可用端口
       ├─ 随机选取一个可用端口
       ├─ AssignmentService::handle() 注册新分配
       └─ lockForUpdate 获取并绑定
```

**冲突避免：** `lockForUpdate()`（SELECT ... FOR UPDATE）在事务内对查询到的分配行加排他锁，防止并发请求同时绑定同一分配。

---

## 8. 面板与守护层之间的协议

### 8.1 通信机制

| 方向 | 协议 | 认证方式 |
|------|------|----------|
| Panel → Wings | HTTPS REST | Bearer Token（Node.daemon_token，AES加密存储） |
| Wings → Panel | HTTPS REST | Node.daemon_token_id + Token |

Panel 通过 `DaemonRepository::getHttpClient()` 构建请求：

```php
'base_uri' => $this->node->getConnectionAddress(),  // scheme://fqdn:daemonListen
'headers'  => [
    'Authorization' => 'Bearer ' . $this->node->getDecryptedKey(),
    'Accept'        => 'application/json',
    'Content-Type'  => 'application/json',
],
```

### 8.2 关键 API 端点

#### Panel → Wings

| 端点 | 方法 | 用途 | 调用时机 |
|------|------|------|----------|
| `/api/servers` | POST | 创建实例 | 新建服务器后 |
| `/api/servers/{uuid}/sync` | POST | 同步配置 | Build修改后 |
| `/api/servers/{uuid}` | DELETE | 删除实例 | 服务器删除/迁移完成后 |
| `/api/servers/{uuid}/reinstall` | POST | 重装实例 | 重装操作 |
| `/api/servers/{uuid}/archive` | POST | 请求归档 | 迁移前打包 |
| `/api/servers/{uuid}/transfer` | POST | 通知迁移 | 迁移发起时 |

#### Wings → Panel（api-remote 路由）

| 端点 | 方法 | 用途 |
|------|------|------|
| `/api/remote/servers` | GET | 批量拉取节点服务器配置 |
| `/api/remote/servers/{uuid}` | GET | 拉取单个服务器配置 |
| `/api/remote/servers/{uuid}/install` | GET/POST | 安装状态上报 |
| `/api/remote/servers/{uuid}/transfer/failure` | POST | 报告迁移失败 |
| `/api/remote/servers/{uuid}/transfer/success` | POST | 报告迁移成功 |
| `/api/remote/servers/reset` | POST | Wings重启后重置异常状态 |

### 8.3 配置下发结构

Wings 拉取服务器配置时获取的核心分配信息（`ServerConfigurationStructureService::returnCurrentFormat()`）：

```json
{
  "allocations": {
    "force_outgoing_ip": false,
    "default": {
      "ip": "1.2.3.4",
      "port": 25565
    },
    "mappings": {
      "1.2.3.4": [25565, 25566, 25567]
    }
  }
}
```

`mappings` 由 `Server::getAllocationMappings()` 生成，按 IP 分组列出所有绑定端口。

### 8.4 最终一致性模型

Panel 与 Wings 之间采用 **最终一致性** 而非强一致性：

1. **Panel 优先写入**：分配变更先持久化到 MySQL
2. **异步推送**：调用 Wings sync，失败仅记录日志
3. **Wings 自愈**：Wings 启动服务器时通过 `/api/remote/servers/{uuid}` 拉取最新配置
4. **Wings 重启恢复**：Wings 进程重启后调用 `/api/remote/servers/reset` 清除异常状态

---

## 9. 冲突检测与回滚路径汇总

| 场景 | 冲突检测机制 | 回滚路径 |
|------|-------------|---------|
| **批量注册端口** | `INSERT IGNORE` + UNIQUE KEY `(node_id, ip, port)` | 静默跳过已存在端口 |
| **手动指定分配创建实例** | `allocation_id UNIQUE on servers` 验证规则 | 事务回滚 |
| **自动部署选择分配** | `WHERE server_id IS NULL` + dedicated IP 过滤 | 无可用节点/分配时抛 NoViableNodeException / NoViableAllocationException |
| **创建实例后Wings失败** | `DaemonConnectionException` 捕获 | `ServerDeletionService::withForce()` 全量清理 |
| **迁移发起——目标分配被占用** | `getUnassignedAllocationIds()` 检查 | 静默跳过已占用分配（⚠️潜在问题） |
| **迁移失败** | Wings 主动上报 `transfer/failure` | 释放新节点预占分配，旧分配不变 |
| **迁移成功** | 新节点Wings上报 `transfer/success` | 释放旧节点分配，从旧节点删除实例 |
| **迁移中状态保护** | `Server::validateCurrentState()` | 返回 HTTP 409，阻止并发操作 |
| **删除实例** | 外键 `ON DELETE SET NULL` | MySQL自动将 `server_id` 置 NULL |
| **添加分配（Build修改）** | `WHERE server_id IS NULL AND node_id = ?` | 只更新空闲分配，已占用的自然被排除 |
| **删除默认分配** | 检查是否有新分配可回退 | 抛出 DisplayException 阻止操作 |
| **用户自服务分配** | `lockForUpdate()` 行级排他锁 | 事务内锁定，防止并发绑定 |
| **Wings同步失败** | `DaemonConnectionException` 捕获 | 仅记录日志，依赖 Wings 下次启动自愈 |

---

## 10. 风险点与改进建议

### 10.1 迁移预占时的静默跳过（高风险）

**问题：** `ServerTransferController::assignAllocationsToServer()` 中，如果目标分配已被占用，代码只是 `continue` 跳过而不报错，且不影响事务提交。

```php
foreach ($allocations as $allocation) {
    if (!in_array($allocation, $unassigned)) {
        continue;  // 静默跳过！无日志、无错误、无回滚
    }
    $updateIds[] = $allocation;
}
```

**后果：**
- ServerTransfer 记录创建成功（`successful = NULL`），迁移进入"进行中"状态
- 但部分或全部新分配实际上没有预占
- Wings 拉取配置时可能发现分配不属于本服务器，触发失败回调
- 如果是默认分配被跳过，Wings 侧必然失败
- 如果是附加分配被跳过，取决于 Egg 是否需要该端口，可能静默成功但实际缺少端口

**根本原因：** 验证规则 `unique:servers` 只检查该分配是否是其他服务器的**默认分配**，不检查它是否被其他服务器作为**附加分配**占用。

**建议：**
1. 在预占阶段验证所有请求的分配 `server_id IS NULL`
2. 若有任何分配已被占用，抛出异常中断事务
3. 错误信息应明确指出哪个分配已被哪个服务器占用

---

### 10.2 事务内发起 HTTP 请求（架构反模式）

**问题：** `ServerTransferController::transfer()` 将 `DaemonTransferRepository::notify()` 放在数据库事务内调用。

**时序风险：**
1. Panel 发起 HTTP POST 到 Wings
2. Wings 接收请求并开始处理
3. 网络超时或 Wings 响应缓慢
4. Panel 侧抛出 `DaemonConnectionException`
5. 数据库事务回滚（Panel 认为迁移未发起）
6. Wings 侧继续执行迁移流程（实际上已开始）

**脑裂状态：**
- Panel：无 ServerTransfer 记录，服务器仍在旧节点
- Wings（新节点）：正在创建实例
- Wings（旧节点）：收到迁移指令可能开始打包
- 新分配：预占已回滚（释放）→ 可能被其他服务器抢走

**建议：**
1. 提交事务后再调用 Wings notify
2. 若 notify 失败，启动补偿流程（后台 Job 异步清理 ServerTransfer 和预占）
3. 增加幂等性：Wings 收到重复迁移指令时检查状态

---

### 10.3 分配更新无乐观锁

**问题：** `storeAssignedAllocations()` 直接 `UPDATE allocations SET server_id = ? WHERE id IN (...)`，不检查 `server_id` 是否仍为 NULL。虽然前置验证步骤通常能捕获冲突，但在高并发场景下存在 TOCTOU（Time-of-check to time-of-use）竞态。

**建议：** 将 UPDATE 条件改为 `WHERE id IN (...) AND server_id IS NULL`，并通过影响行数验证是否全部更新成功。

---

### 10.4 自动部署的随机选择无排序保证

**问题：** `AllocationRepository::getRandomAllocation()` 使用 `inRandomOrder()->first()`，在高并发场景下多个请求可能选中同一分配后产生冲突。

**建议：** 使用 `SELECT ... FOR UPDATE SKIP LOCKED`（MySQL 8.0+）或 `lockForUpdate()` 实现行级锁定后再选择。

---

### 10.5 迁移成功后旧节点清理的容错

**问题：** 迁移成功后删除旧节点上的实例如果失败，仅记录日志。旧节点上可能残留"幽灵实例"。

**建议：** 增加定期清理任务或管理员工具，检测并清理无对应 Panel 记录的 Wings 实例。

---

### 10.6 端口范围无节点级差异化配置

**问题：** 用户自服务自动分配的端口范围是全局配置（`PTERODACTYL_CLIENT_ALLOCATIONS_RANGE_START/END`），不同节点无法设置不同范围。

**建议：** 将端口范围配置下沉到节点级别，支持每个节点定义自己的可用端口池。

---

### 10.7 节点容量检查的实时性

**问题：** `FindViableNodesService` 通过 `SUM(servers.memory)` 计算已分配资源，但这是 Panel 侧的静态值，不考虑 Wings 侧实际运行时的资源使用。

**建议：** 集成 Wings 的实时资源使用数据作为辅助判断。

---

## 附录：核心文件索引

| 文件 | 职责 |
|------|------|
| `app/Models/Allocation.php` | 分配模型，端口/IP/占用状态 |
| `app/Models/Node.php` | 节点模型，容量约束与 `isViable()` |
| `app/Models/Server.php` | 服务器模型，状态验证与分配映射 |
| `app/Models/ServerTransfer.php` | 迁移记录模型 |
| `app/Models/Objects/DeploymentObject.php` | 自动部署参数对象 |
| `app/Services/Allocations/AssignmentService.php` | 批量端口注册（INSERT IGNORE） |
| `app/Services/Allocations/FindAssignableAllocationService.php` | 用户自服务分配（lockForUpdate） |
| `app/Services/Deployment/FindViableNodesService.php` | 可行节点筛选 |
| `app/Services/Deployment/AllocationSelectionService.php` | 自动分配选择 |
| `app/Services/Servers/ServerCreationService.php` | 实例创建（含回滚） |
| `app/Services/Servers/ServerDeletionService.php` | 实例删除（含ON DELETE SET NULL回收） |
| `app/Services/Servers/BuildModificationService.php` | 运行时分配变更 |
| `app/Services/Servers/ServerConfigurationStructureService.php` | Wings配置序列化 |
| `app/Http/Controllers/Admin/Servers/ServerTransferController.php` | 迁移发起 |
| `app/Http/Controllers/Api/Remote/Servers/ServerTransferController.php` | 迁移结果回调 |
| `app/Repositories/Eloquent/AllocationRepository.php` | 分配查询（空闲/专用IP/随机选择） |
| `app/Repositories/Wings/DaemonRepository.php` | Wings通信基座 |
| `app/Repositories/Wings/DaemonServerRepository.php` | Wings实例操作 |
| `app/Repositories/Wings/DaemonTransferRepository.php` | Wings迁移通知 |
