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

### 10.1 迁移预占时的静默跳过

**问题：** `ServerTransferController::assignAllocationsToServer()` 中，如果目标分配已被占用，代码只是跳过而不报错。这可能导致迁移成功后服务器缺少预期的分配。

**建议：** 在预占阶段验证所有请求的分配均为空闲，若有已被占用者应提前失败。

### 10.2 分配更新无乐观锁

**问题：** `storeAssignedAllocations()` 直接 `UPDATE allocations SET server_id = ? WHERE id IN (...)`，不检查 `server_id` 是否仍为 NULL。虽然前置验证步骤通常能捕获冲突，但在高并发场景下存在 TOCTOU（Time-of-check to time-of-use）竞态。

**建议：** 将 UPDATE 条件改为 `WHERE id IN (...) AND server_id IS NULL`，并通过影响行数验证是否全部更新成功。

### 10.3 自动部署的随机选择无排序保证

**问题：** `AllocationRepository::getRandomAllocation()` 使用 `inRandomOrder()->first()`，在高并发场景下多个请求可能选中同一分配后产生冲突。

**建议：** 使用 `SELECT ... FOR UPDATE SKIP LOCKED`（MySQL 8.0+）或 `lockForUpdate()` 实现行级锁定后再选择。

### 10.4 迁移成功后旧节点清理的容错

**问题：** 迁移成功后删除旧节点上的实例如果失败，仅记录日志。旧节点上可能残留"幽灵实例"。

**建议：** 增加定期清理任务或管理员工具，检测并清理无对应 Panel 记录的 Wings 实例。

### 10.5 端口范围无节点级差异化配置

**问题：** 用户自服务自动分配的端口范围是全局配置（`PTERODACTYL_CLIENT_ALLOCATIONS_RANGE_START/END`），不同节点无法设置不同范围。

**建议：** 将端口范围配置下沉到节点级别，支持每个节点定义自己的可用端口池。

### 10.6 节点容量检查的实时性

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
