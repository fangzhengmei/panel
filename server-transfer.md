# 游戏服务器迁移调度系统分析

## 一、整体架构概述

本系统实现了游戏服务器从源节点（旧节点）到目标节点（新节点）的完整迁移调度流程。系统采用 **Panel（控制面板）+ Wings（守护节点）** 双层架构，Panel 负责调度决策与状态管理，Wings 负责实际的文件同步与服务器生命周期管理。

迁移流程分为五个核心阶段：

```
用户发起迁移 → 状态前置校验 → 分配占位 → 启动节点间同步 → 状态回填/回滚
```

---

## 二、核心数据模型

### ServerTransfer 模型 [`app/Models/ServerTransfer.php:26`]

| 字段 | 类型 | 说明 |
|------|------|------|
| `server_id` | int | 关联的服务器 ID |
| `old_node` / `new_node` | int | 源节点 / 目标节点 ID |
| `old_allocation` / `new_allocation` | int | 主分配 ID（IP+端口） |
| `old_additional_allocations` | array\|null | 源节点附加分配列表 |
| `new_additional_allocations` | array\|null | 目标节点附加分配列表 |
| `successful` | bool\|null | 迁移结果：null=进行中，true=成功，false=失败 |
| `archived` | bool | 是否已归档 |

**关键关联设计：**

```php
// Server 模型中的关联定义，只会返回进行中的迁移
public function transfer(): HasOne
{
    return $this->hasOne(ServerTransfer::class)
        ->whereNull('successful')
        ->orderByDesc('id');
}
```

> **设计意图：** 通过 `whereNull('successful')` 过滤，确保 `$server->transfer` 始终返回**当前正在进行**的迁移记录，自动排除历史记录。

---

## 三、迁移启动流程（分配占位阶段）

### 入口：Admin 控制器 [`app/Http/Controllers/Admin/Servers/ServerTransferController.php:38`]

#### 3.1 前置校验（三层防护）

**第一层：参数合法性校验**
```php
$validatedData = $request->validate([
    'node_id' => 'required|exists:nodes,id',
    'allocation_id' => 'required|bail|unique:servers|exists:allocations,id',
    'allocation_additional' => 'nullable',
]);
```

**第二层：目标节点资源可行性检查** [`app/Repositories/Eloquent/NodeRepository.php:142`]
```php
$node = $this->nodeRepository->getNodeWithResourceUsage($node_id);
if (!$node->isViable($server->memory, $server->disk)) {
    // 资源不足，拒绝迁移
}
```

`isViable()` 算法 [`app/Models/Node.php:242`]：
```php
public function isViable(int $memory, int $disk): bool
{
    $memoryLimit = $this->memory * (1 + ($this->memory_overallocate / 100));
    $diskLimit = $this->disk * (1 + ($this->disk_overallocate / 100));
    return ($this->sum_memory + $memory) <= $memoryLimit 
        && ($this->sum_disk + $disk) <= $diskLimit;
}
```

> **关键点：** 支持超售（overallocate）配置，允许节点资源按比例超额分配。

**第三层：服务器状态校验** [`app/Models/Server.php:409`]
```php
public function validateTransferState()
{
    if (
        !$this->isInstalled()            // 未安装完成
        || $this->status === self::STATUS_RESTORING_BACKUP  // 正在恢复备份
        || !is_null($this->transfer)      // 正在迁移中
    ) {
        throw new ServerStateConflictException($this);
    }
}
```

> **注意：** 与 `validateCurrentState()` 不同，`validateTransferState()` **允许已暂停（suspended）的服务器进行迁移**。

#### 3.2 数据库事务与分配占位

整个启动过程在数据库事务内执行，确保原子性：

```php
$this->connection->transaction(function () use (...) {
    // 1. 创建迁移记录
    $transfer = new ServerTransfer();
    $transfer->server_id = $server->id;
    $transfer->old_node = $server->node_id;
    $transfer->new_node = $node_id;
    $transfer->old_allocation = $server->allocation_id;
    $transfer->new_allocation = $allocation_id;
    $transfer->old_additional_allocations = [...];
    $transfer->new_additional_allocations = $additional_allocations;
    $transfer->save();

    // 2. 预占目标节点分配（关键！）
    $this->assignAllocationsToServer($server, $node_id, $allocation_id, $additional_allocations);

    // 3. 生成 JWT 认证令牌（15分钟有效期）
    $token = $this->nodeJWTService
        ->setExpiresAt(CarbonImmutable::now()->addMinutes(15))
        ->setSubject($server->uuid)
        ->handle($transfer->newNode, $server->uuid, 'sha256');

    // 4. 通知源 Wings 节点启动迁移
    $this->daemonTransferRepository->setServer($server)->notify($transfer->newNode, $token);
});
```

**分配占位机制** [`app/Http/Controllers/Admin/Servers/ServerTransferController.php:97`]：
```php
private function assignAllocationsToServer(Server $server, int $node_id, int $allocation_id, array $additional_allocations)
{
    $allocations = array_merge($additional_allocations, [$allocation_id]);
    $unassigned = $this->allocationRepository->getUnassignedAllocationIds($node_id);
    
    $updateIds = [];
    foreach ($allocations as $allocation) {
        if (in_array($allocation, $unassigned)) {
            $updateIds[] = $allocation;
        }
    }

    if (!empty($updateIds)) {
        $this->allocationRepository->updateWhereIn('id', $updateIds, ['server_id' => $server->id]);
    }
}
```

> **核心设计：** 将目标节点的端口分配预先绑定到该服务器，`server_id` 字段作为"占位标记"，防止迁移期间被其他服务器占用。此设计巧妙地复用了现有 `allocations` 表结构，无需新增锁表。

#### 3.3 JWT 认证设计 [`app/Services/Nodes/NodeJWTService.php:63`]

迁移令牌包含：
- 签发者：Panel 域名
- 受众：目标节点连接地址
- 主题：服务器 UUID
- 有效期：15 分钟
- 签名算法：HMAC-SHA256，使用目标节点的守护密钥签名

> **安全要点：** 源节点使用此令牌向目标节点认证，确保只有合法的迁移请求才能被目标节点接受。

---

## 四、节点间通信（文件同步阶段）

### DaemonTransferRepository [`app/Repositories/Wings/DaemonTransferRepository.php:19`]

```php
public function notify(Node $targetNode, Plain $token): void
{
    $this->getHttpClient()->post(sprintf('/api/servers/%s/transfer', $this->server->uuid), [
        'json' => [
            'server_id' => $this->server->uuid,
            'url' => $targetNode->getConnectionAddress() . '/api/transfers',
            'token' => 'Bearer ' . $token->toString(),
            'server' => [
                'uuid' => $this->server->uuid,
                'start_on_completion' => false,  // 迁移完成后不自动启动
            ],
        ],
    ]);
}
```

**Wings 节点间的文件同步流程（推断）：**

1. **源节点** 收到通知后，挂起服务器（停止运行）
2. **源节点** 使用提供的 `url` 和 `token` 连接 **目标节点**
3. 两节点间直接建立数据通道，同步服务器文件（快照/增量）
4. 同步完成后，**源节点**或**目标节点**向 Panel 回调结果

> **注意：** Panel 仅负责触发和协调，**不参与实际的文件传输**，大文件流量在节点间直接流动，避免 Panel 成为瓶颈。

---

## 五、状态回填与回滚保险

### 回调接口设计 [`app/Http/Controllers/Api/Remote/Servers/ServerTransferController.php`]

#### 5.1 成功回调 `success()` [`app/Http/Controllers/Api/Remote/Servers/ServerTransferController.php:63`]

**权限控制：** 只有 **目标节点（newNode）** 才能报告成功

```php
if (! $node->is($transfer->newNode)) {
    throw new HttpForbiddenException(...);
}
```

**成功处理事务：**
```php
$server = $this->connection->transaction(function () use ($server, $transfer) {
    // 1. 释放源节点的所有分配
    $allocations = array_merge([$transfer->old_allocation], $transfer->old_additional_allocations);
    Allocation::query()->whereIn('id', $allocations)->update(['server_id' => null]);
    
    // 2. 更新服务器归属到新节点和新分配
    $server->update([
        'allocation_id' => $transfer->new_allocation,
        'node_id' => $transfer->new_node,
    ]);
    
    // 3. 标记迁移成功
    $server->transfer->update(['successful' => true]);
    
    return $server;
});

// 4. 异步清理：通知源节点删除服务器文件
try {
    $this->daemonServerRepository
        ->setServer($server)
        ->setNode($transfer->oldNode)
        ->delete();
} catch (DaemonConnectionException $exception) {
    Log::warning($exception, ['transfer_id' => $server->transfer->id]);
}
```

> **容错设计：** 源节点删除失败**不影响迁移结果**，仅记录日志。这是一种"最终一致性"设计，宁可源节点残留垃圾文件，也不让迁移失败。

#### 5.2 失败回调 `failure()` [`app/Http/Controllers/Api/Remote/Servers/ServerTransferController.php:38`]

**权限控制：源节点和目标节点均可报告失败**

```php
// 任一节点都可报告失败
if (! $node->is($transfer->newNode) && ! $node->is($transfer->oldNode)) {
    throw new HttpForbiddenException(...);
}

return $this->processFailedTransfer($transfer);
```

**失败回滚逻辑 `processFailedTransfer()`** [`app/Http/Controllers/Api/Remote/Servers/ServerTransferController.php:118`]：
```php
protected function processFailedTransfer(ServerTransfer $transfer): JsonResponse
{
    $this->connection->transaction(function () use (&$transfer) {
        // 1. 标记迁移失败
        $transfer->forceFill(['successful' => false])->saveOrFail();
        
        // 2. 释放目标节点的预占分配（核心回滚）
        $allocations = array_merge([$transfer->new_allocation], $transfer->new_additional_allocations);
        Allocation::query()->whereIn('id', $allocations)->update(['server_id' => null]);
    });
    
    return new JsonResponse([], Response::HTTP_NO_CONTENT);
}
```

> **回滚保险的核心：** 预占的分配在失败时被释放，恢复可用状态。服务器本身仍停留在源节点，用户无感知。

---

## 六、并发与状态保护机制

### 6.1 迁移期间的操作封锁

**SuspensionService 中的保护** [`app/Services/Servers/SuspensionService.php:41`]：
```php
if (!is_null($server->transfer)) {
    throw new ConflictHttpException(
        'Cannot toggle suspension status on a server that is currently being transferred.'
    );
}
```

**Server 模型的全局状态检查** [`app/Models/Server.php:390`]：
```php
public function validateCurrentState()
{
    if (
        $this->isSuspended()
        || $this->node->isUnderMaintenance()
        || !$this->isInstalled()
        || $this->status === self::STATUS_RESTORING_BACKUP
        || !is_null($this->transfer)  // 迁移中封锁所有状态变更
    ) {
        throw new ServerStateConflictException($this);
    }
}
```

**ServerDetailsController 中的双节点权限** [`app/Http/Controllers/Api/Remote/Servers/ServerDetailsController.php:48`]：
```php
$valid = $transfer
    ? $node->id === $transfer->old_node || $node->id === $transfer->new_node
    : $node->id === $server->node_id;
```

> **设计意图：** 迁移期间，源节点和目标节点都有权限获取服务器配置，确保两节点都能正确完成同步流程。

### 6.2 测试验证的安全边界

从 `ServerTransferControllerTest.php` 可验证以下安全规则：

| 场景 | 源节点 | 目标节点 | 第三方节点 |
|------|--------|----------|------------|
| 报告成功 | ❌ 禁止 | ✅ 允许 | ❌ 禁止 |
| 报告失败 | ✅ 允许 | ✅ 允许 | ❌ 禁止 |

---

## 七、回滚保险体系总结

系统设计了多层回滚保险机制：

### 第一层：事务原子性
- 迁移启动阶段使用数据库事务，任何一步失败都会自动回滚
- 迁移记录创建和分配占位在同一事务中

### 第二层：分配占位与释放
- 启动时预占目标节点分配 → 防止并发占用
- 失败时释放目标节点分配 → 恢复资源
- 成功时释放源节点分配 → 清理旧资源

### 第三层：异步清理容忍
- 成功后源节点文件删除是异步非阻塞的
- 删除失败仅记录日志，不影响迁移结果
- 属于"向前恢复"而非"向后回滚"

### 第四层：状态封锁
- 迁移期间服务器状态为"迁移中"，禁止其他状态变更
- 防止用户在迁移过程中执行暂停、删除等操作

### 第五层：双向失败报告
- 源节点和目标节点均可触发失败回滚
- 无论哪一侧发现问题，都能及时终止并回滚

---

## 八、代码优化建议

### 建议 1：迁移失败后的源节点清理

当前 `processFailedTransfer()` 只释放了目标节点的分配，但**目标节点上可能已经同步了部分文件**，建议增加对目标节点的清理通知：

```php
protected function processFailedTransfer(ServerTransfer $transfer): JsonResponse
{
    $this->connection->transaction(function () use (&$transfer) {
        $transfer->forceFill(['successful' => false])->saveOrFail();
        
        $allocations = array_merge([$transfer->new_allocation], $transfer->new_additional_allocations);
        Allocation::query()->whereIn('id', $allocations)->update(['server_id' => null]);
    });

    // 【新增】尝试清理目标节点上的部分同步文件
    try {
        $this->daemonServerRepository
            ->setServer($transfer->server)
            ->setNode($transfer->newNode)
            ->delete();
    } catch (DaemonConnectionException $exception) {
        Log::warning('Failed to clean up target node after failed transfer', [
            'transfer_id' => $transfer->id,
            'error' => $exception->getMessage(),
        ]);
    }
    
    return new JsonResponse([], Response::HTTP_NO_CONTENT);
}
```

### 建议 2：增加迁移超时机制

当前迁移记录的 `successful` 字段为 `null` 表示进行中，但没有超时处理。如果 Wings 节点宕机，迁移将永远卡在"进行中"状态。建议：

1. 增加 `started_at` 字段记录迁移开始时间
2. 增加定时任务检查超时迁移（如超过 2 小时），自动标记为失败并回滚

### 建议 3：`transfer()` 关联的潜在问题

```php
// 当前实现
public function transfer(): HasOne
{
    return $this->hasOne(ServerTransfer::class)
        ->whereNull('successful')
        ->orderByDesc('id');
}
```

**问题：** 如果一个服务器有多个失败的迁移记录（`successful=false`），然后再创建新的迁移，`whereNull('successful')` 会正确返回最新的进行中迁移。但如果某次迁移异常终止（`successful` 仍为 `null`），则后续迁移永远无法创建。

**优化：** 在 `validateTransferState()` 中增加超时检查，或者在创建新迁移前自动将旧的"僵尸"迁移标记为失败。

---

## 九、总结

### 架构亮点
1. **职责分离清晰**：Panel 管状态，Wings 管数据
2. **分配占位设计巧妙**：复用 `allocation.server_id` 字段实现分布式锁
3. **回滚机制完善**：从事务原子性到最终一致性的多层防护
4. **安全边界明确**：严格的节点权限控制，防止恶意回调
5. **失败容错设计**：非关键路径失败不阻塞主流程

### 核心文件索引
| 模块 | 文件路径 | 关键行 |
|------|---------|--------|
| 数据模型 | `app/Models/ServerTransfer.php` | 26 |
| 迁移启动 | `app/Http/Controllers/Admin/Servers/ServerTransferController.php` | 38 |
| 节点通信 | `app/Repositories/Wings/DaemonTransferRepository.php` | 19 |
| 成功回调 | `app/Http/Controllers/Api/Remote/Servers/ServerTransferController.php` | 63 |
| 失败回滚 | `app/Http/Controllers/Api/Remote/Servers/ServerTransferController.php` | 118 |
| 状态校验 | `app/Models/Server.php` | 409 |
| 资源检查 | `app/Models/Node.php` | 242 |
