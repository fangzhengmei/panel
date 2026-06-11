# Pterodactyl Panel 自动放置算法代码解析

## 一、整体架构与处理路径

服务器创建流程涉及两条主要路径：**手动指定节点**与**自动部署（Auto-Deployment）**。当用户选择自动部署时，系统会依次经过以下核心服务层：

```
请求入口（Admin UI / Application API）
    ↓
1. 表单/请求验证（ServerFormRequest / StoreServerRequest）
    ↓
2. ServerCreationService::handle()
    ├─ 如果是自动部署 → configureDeployment()
    │   ├─ FindViableNodesService → 筛选满足容量的节点
    │   └─ AllocationSelectionService → 在候选节点中选择端口分配
    ├─ 数据库事务：创建 Server 记录、绑定 Allocation、存储 Egg 变量
    └─ DaemonServerRepository::create() → 下发给 Wings Daemon
    ↓
3. 失败回退：若 Wings 连接失败 → ServerDeletionService 清理数据库记录
```

### 关键文件索引

| 文件 | 功能 |
|------|------|
| [FindViableNodesService.php](file:///d:/fz/0508-3/solo-dogfeeding/code/210-panel/app/Services/Deployment/FindViableNodesService.php) | 节点容量筛选核心算法 |
| [AllocationSelectionService.php](file:///d:/fz/0508-3/solo-dogfeeding/code/210-panel/app/Services/Deployment/AllocationSelectionService.php) | Allocation 选择（端口/IP分配） |
| [ServerCreationService.php](file:///d:/fz/0508-3/solo-dogfeeding/code/210-panel/app/Services/Servers/ServerCreationService.php) | 服务器创建编排，部署配置，失败回退 |
| [Node.php](file:///d:/fz/0508-3/solo-dogfeeding/code/210-panel/app/Models/Node.php) | Node 模型（含 isViable() 容量校验方法） |
| [AllocationRepository.php](file:///d:/fz/0508-3/solo-dogfeeding/code/210-panel/app/Repositories/Eloquent/AllocationRepository.php) | getRandomAllocation() 随机分配算法 |
| [NodeRepository.php](file:///d:/fz/0508-3/solo-dogfeeding/code/210-panel/app/Repositories/Eloquent/NodeRepository.php) | getUsageStats() 使用率统计 |
| [BuildModificationService.php](file:///d:/fz/0508-3/solo-dogfeeding/code/210-panel/app/Services/Servers/BuildModificationService.php) | Allocation 增减、default 回退 |
| [DeploymentObject.php](file:///d:/fz/0508-3/solo-dogfeeding/code/210-panel/app/Models/Objects/DeploymentObject.php) | 部署参数对象（locations / ports / dedicated） |
| [StoreServerRequest.php](file:///d:/fz/0508-3/solo-dogfeeding/code/210-panel/app/Http/Requests/Api/Application/Servers/StoreServerRequest.php) | API 层部署参数解析（getDeploymentObject） |
| [DaemonServerRepository.php](file:///d:/fz/0508-3/solo-dogfeeding/code/210-panel/app/Repositories/Wings/DaemonServerRepository.php) | Wings Daemon 通信（POST /api/servers） |
| [ServerDeletionService.php](file:///d:/fz/0508-3/solo-dogfeeding/code/210-panel/app/Services/Servers/ServerDeletionService.php) | 失败回退时清理资源 |
| [NodeDeploymentController.php](file:///d:/fz/0508-3/solo-dogfeeding/code/210-panel/app/Http/Controllers/Api/Application/Nodes/NodeDeploymentController.php) | API 部署节点查询端点 |

---

## 二、入口层：请求验证与部署参数解析

### 2.1 Admin UI 路径（CreateServerController）

Admin 面板通过 [CreateServerController::store()](file:///d:/fz/0508-3/solo-dogfeeding/code/210-panel/app/Http/Controllers/Admin/Servers/CreateServerController.php#L70-L83) 处理表单：

```php
// 第 78 行：直接调用 creationService，未传 DeploymentObject
$server = $this->creationService->handle($data);
```

Admin UI 使用 `auto_deploy` 标志字段（在 [ServerFormRequest::withValidator()](file:///d:/fz/0508-3/solo-dogfeeding/code/210-panel/app/Http/Requests/Admin/ServerFormRequest.php#L26-L57) 中验证）：

- **`auto_deploy = false`**（默认）：`node_id` 与 `allocation_id` 必选，手动指定
- **`auto_deploy = true`**：不需要 `node_id`/`allocation_id`，由自动部署逻辑接管

### 2.2 Application API 路径（ServerController）

通过 [StoreServerRequest::getDeploymentObject()](file:///d:/fz/0508-3/solo-dogfeeding/code/210-panel/app/Http/Requests/Api/Application/Servers/StoreServerRequest.php#L138-L150) 解析 `deploy` 字段：

```php
$object = new DeploymentObject();
$object->setDedicated($this->input('deploy.dedicated_ip', false));
$object->setLocations($this->input('deploy.locations', []));
$object->setPorts($this->input('deploy.port_range', []));
```

部署参数结构体 [DeploymentObject](file:///d:/fz/0508-3/solo-dogfeeding/code/210-panel/app/Models/Objects/DeploymentObject.php)：

| 字段 | 类型 | 含义 |
|------|------|------|
| `dedicated` | bool | 是否独占 IP（dedicated_ip） |
| `locations` | int[] | 候选位置 ID 数组 |
| `ports` | string[] | 端口或端口范围（如 `25565` 或 `25565-25570`） |

---

## 三、第一阶段：FindViableNodesService — 节点容量筛选

### 3.1 SQL 查询结构

[FindViableNodesService::handle()](file:///d:/fz/0508-3/solo-dogfeeding/code/210-panel/app/Services/Deployment/FindViableNodesService.php#L69-L99) 是自动放置算法的核心，通过单条 SQL 完成筛选：

```php
$query = Node::query()->select('nodes.*')
    ->selectRaw('IFNULL(SUM(servers.memory), 0) as sum_memory')
    ->selectRaw('IFNULL(SUM(servers.disk), 0) as sum_disk')
    ->leftJoin('servers', 'servers.node_id', '=', 'nodes.id')
    ->where('nodes.public', 1);                              // 条件①：public=1 的节点

if (!empty($this->locations)) {
    $query = $query->whereIn('nodes.location_id', $this->locations);  // 条件②：位置过滤
}

$results = $query->groupBy('nodes.id')
    // 条件③：内存容量校验
    ->havingRaw('(IFNULL(SUM(servers.memory), 0) + ?) 
                <= (nodes.memory * (1 + (nodes.memory_overallocate / 100)))', [$this->memory])
    // 条件④：磁盘容量校验
    ->havingRaw('(IFNULL(SUM(servers.disk), 0) + ?) 
                <= (nodes.disk * (1 + (nodes.disk_overallocate / 100)))', [$this->disk]);
```

### 3.2 容量计算公式（标准）

**节点容量上限 = 基础容量 × (1 + overallocate 百分比)**

```
内存上限 = nodes.memory × (1 + nodes.memory_overallocate / 100)
磁盘上限 = nodes.disk   × (1 + nodes.disk_overallocate / 100)
```

**已使用量 = 该节点下所有 servers 表记录的 memory/disk 字段之和**（通过 LEFT JOIN + SUM 聚合）。

**判定条件**：`已使用量 + 新增服务器请求量 ≤ 节点容量上限`

### 3.3 overallocate（超售）值的语义

在 [Node.php](file:///d:/fz/0508-3/solo-dogfeeding/code/210-panel/app/Models/Node.php#L107-L109) 验证规则中：

| `memory_overallocate` 值 | 含义 |
|-------------------------|------|
| `-1` | **无限超售**：公式 `(1 + (-1/100)) = 0.99`，但在 SQL 中实际表现为上限非常宽松。⚠️ 该值在 `numeric|min:-1` 中允许，具体语义依赖面板文档 |
| `0`  | **禁止超售**：上限 = 100% 基础容量 |
| `50` | **允许超售 50%**：上限 = 基础容量 × 1.5 |
| `100`| **允许超售 100%**：上限 = 基础容量 × 2.0 |

> **注意**：SQL 使用的是 `1 + (overallocate / 100)` 而非 `1 + (overallocate / 100 * sign)`，当 `overallocate = -1` 时上限会略低于基础容量（99%），这是一个需要注意的边界行为。

### 3.4 模型层辅助校验：Node::isViable()

在 [Node.php::isViable()](file:///d:/fz/0508-3/solo-dogfeeding/code/210-panel/app/Models/Node.php#L242-L249) 中提供了相同算法的 PHP 版本，用于非 SQL 场景（如单节点检查）：

```php
public function isViable(int $memory, int $disk): bool
{
    $memoryLimit = $this->memory * (1 + ($this->memory_overallocate / 100));
    $diskLimit   = $this->disk   * (1 + ($this->disk_overallocate   / 100));

    return ($this->sum_memory + $memory) <= $memoryLimit
        && ($this->sum_disk   + $disk)   <= $diskLimit;
}
```

### 3.5 结果输出

- 如果结果集为空 → 抛出 **[NoViableNodeException](file:///d:/fz/0508-3/solo-dogfeeding/code/210-panel/app/Exceptions/Service/Deployment/NoViableNodeException.php)**（用户可翻译消息 `exceptions.deployment.no_viable_nodes`）
- 结果中每条 Node 记录会额外携带 `sum_memory` 与 `sum_disk` 动态属性（通过 selectRaw 注入）
- 支持分页：`handle($perPage, $page)`，默认不分页

> **⚠️ 重要**：此阶段结果**未按资源使用率排序**，仅为满足条件的节点集合（SQL 默认按主键或 groupBy 顺序）。「资源最少节点优先」的排序在当前代码中**并未实现**——选择哪个节点实际是由下一个阶段的 Allocation 随机选择间接决定的。

---

## 四、第二阶段：AllocationSelectionService — 端口/IP 分配

### 4.1 选择流程

[AllocationSelectionService::handle()](file:///d:/fz/0508-3/solo-dogfeeding/code/210-panel/app/Services/Deployment/AllocationSelectionService.php#L85-L94) 将 FindViableNodesService 返回的节点 ID 列表传入 AllocationRepository：

```php
$allocation = $this->repository->getRandomAllocation(
    $this->nodes,      // 上一阶段筛选出的节点 ID 数组
    $this->ports,      // 用户指定的端口范围
    $this->dedicated   // 是否独占 IP
);
```

### 4.2 AllocationRepository::getRandomAllocation()

[AllocationRepository::getRandomAllocation()](file:///d:/fz/0508-3/solo-dogfeeding/code/210-panel/app/Repositories/Eloquent/AllocationRepository.php#L59-L99) 算法步骤：

```php
$query = Allocation::query()->whereNull('server_id');  // 条件①：未被分配

if (!empty($nodes)) {
    $query->whereIn('node_id', $nodes);                 // 条件②：节点范围限制
}

if (!empty($ports)) {
    $query->where(function (Builder $inner) use ($ports) {
        // 条件③：端口匹配（单端口 IN + 端口范围 BETWEEN）
        foreach ($ports as $port) {
            if (is_array($port)) {
                $inner->orWhereBetween('port', $port);  // [25565, 25570]
            } else {
                $whereIn[] = $port;                     // 单个端口
            }
        }
        if (!empty($whereIn)) $inner->orWhereIn('port', $whereIn);
    });
}

if ($dedicated) {
    $discard = $this->getDiscardableDedicatedAllocations($nodes);
    if (!empty($discard)) {
        // 条件④：独占 IP 过滤 — 排除已有服务器的 (node_id, ip) 组合
        $query->whereNotIn(
            $this->getBuilder()->raw('CONCAT_WS("-", node_id, ip)'),
            $discard
        );
    }
}

return $query->inRandomOrder()->first();  // ⚠️ 随机取第一条
```

### 4.3 关键机制：如何间接实现「资源最少优先」

由于 `inRandomOrder()` 是**完全随机**选择，代码本身没有实现「资源最少节点优先」。但：

1. **FindViableNodesService 结果集的大小**决定了每个节点被选中的概率
2. 节点中**空闲 Allocation 数量多**意味着被抽中的概率更高（因为在 allocations 表中有更多条记录参与抽奖）
3. 在实际使用中，「资源少的节点」通常创建的服务器少，空闲 Allocation 也更充足——这形成了一种**间接的、统计意义上的弱均衡**

> **扩容建议**：如果希望严格按 `sum_memory + sum_disk` 最小优先，需要在 FindViableNodesService::handle() 后增加 `ORDER BY (sum_memory + sum_disk) ASC`，并在 AllocationSelectionService 中优先选择第一个节点，而非全节点随机。

### 4.4 失败条件

- 如果 `getRandomAllocation()` 返回 `null` → 抛出 **[NoViableAllocationException](file:///d:/fz/0508-3/solo-dogfeeding/code/210-panel/app/Exceptions/Service/Deployment/NoViableAllocationException.php)**
- 这意味着：节点有容量，但所有满足条件的 IP/端口都被占用了

---

## 五、容量使用量的计算口径

### 5.1 NodeRepository::getUsageStats() 可视化统计

[NodeRepository::getUsageStats()](file:///d:/fz/0508-3/solo-dogfeeding/code/210-panel/app/Repositories/Eloquent/NodeRepository.php#L22-L49) 在管理后台显示节点使用量，使用相同的计算口径：

```php
$maxUsage = $node->{$key};
if ($node->{$key . '_overallocate'} > 0) {
    $maxUsage = $node->{$key} * (1 + ($node->{$key . '_overallocate'} / 100));
}
$percent = ($value / $maxUsage) * 100;
```

信号灯阈值（self::THRESHOLD_PERCENTAGE_LOW / MEDIUM）：
- `≤ LOW` → **green**（健康）
- `(LOW, MEDIUM]` → **yellow**（注意）
- `> MEDIUM` → **red**（告警）

### 5.2 使用量包含哪些服务器？

SQL 通过 `LEFT JOIN servers ON servers.node_id = nodes.id` 聚合——**所有 node_id 匹配的 servers 记录都计入**，不论：
- 服务器状态（installing / suspended / install_failed 都占用容量）
- 服务器是否正在被删除（软删除除外）

这意味着**暂停（suspended）的服务器仍占用容量配额**，不会因为无法运行而释放资源。

---

## 六、失败回退（Fallback）与事务保证

### 6.1 ServerCreationService 的事务+回滚结构

[ServerCreationService::handle()](file:///d:/fz/0508-3/solo-dogfeeding/code/210-panel/app/Services/Servers/ServerCreationService.php#L52-L107) 采用「**先持久化，后调用 Wings，失败则清理**」的策略：

```php
// 步骤 1：数据库事务创建服务器（重试 5 次）
$server = $this->connection->transaction(function () use ($data, $eggVariableData) {
    $server = $this->createModel($data);           // INSERT servers
    $this->storeAssignedAllocations($server, $data);  // UPDATE allocations SET server_id
    $this->storeEggVariables($server, $eggVariableData);  // INSERT server_variables
    return $server;
}, 5);

// 步骤 2：调用 Wings Daemon
try {
    $this->daemonServerRepository->setServer($server)->create($startOnCompletion);
} catch (DaemonConnectionException $exception) {
    // ⚠️ 失败回退：强制删除已写库的服务器
    $this->serverDeletionService->withForce()->handle($server);
    throw $exception;
}
```

### 6.2 ServerDeletionService 清理细节

[ServerDeletionService::handle()](file:///d:/fz/0508-3/solo-dogfeeding/code/210-panel/app/Services/Servers/ServerDeletionService.php#L43-L86)：

```php
try {
    $this->daemonServerRepository->setServer($server)->delete();  // 尝试告诉 Wings 删除
} catch (DaemonConnectionException $exception) {
    // withForce=true 时：忽略 Daemon 404，不忽略其他错误
    if (!$this->force && $exception->getStatusCode() !== 404) {
        throw $exception;
    }
}

$this->connection->transaction(function () use ($server) {
    // 清理关联数据库（含 force 降级策略：删不掉 host 上的 DB 就只删面板记录）
    foreach ($server->databases as $database) { ... }
    // 清空 allocations.notes（防止备注泄露）
    $server->allocations()->update(['notes' => null]);
    // 删除 servers 记录 → allocations.server_id 将通过外键/模型层被置空
    $server->delete();
});
```

### 6.3 Daemon 通信端点

[DaemonServerRepository::create()](file:///d:/fz/0508-3/solo-dogfeeding/code/210-panel/app/Repositories/Wings/DaemonServerRepository.php#L42-L56) 发送：

```
POST {node.getConnectionAddress()}/api/servers
{
    "uuid": "<server-uuid>",
    "start_on_completion": true/false
}
```

Wings 收到后会从 Panel 拉取完整配置（包括资源限制、挂载、启动命令等），所以此处不传细节。

---

## 七、与现有 Allocation 的相互影响

### 7.1 Allocation 生命周期

| 阶段 | allocations.server_id | 说明 |
|------|----------------------|------|
| 创建（AssignmentService） | `NULL` | 由 [AssignmentService::handle()](file:///d:/fz/0508-3/solo-dogfeeding/code/210-panel/app/Services/Allocations/AssignmentService.php) 批量导入，CIDR / 端口范围校验 |
| 分配给服务器 | `= server.id` | `storeAssignedAllocations()` → `UPDATE allocations SET server_id = ? WHERE id IN (?)` |
| 服务器删除 | → `NULL` | 通过 ServerDeletionService 事务级联 |
| Build 修改（增/删 Allocation） | 切换 | [BuildModificationService::processAllocations()](file:///d:/fz/0508-3/solo-dogfeeding/code/210-panel/app/Services/Servers/BuildModificationService.php#L82-L130) |

### 7.2 Default Allocation 回退机制

在修改服务器 Allocation（Build 页面）时，[BuildModificationService::processAllocations()](file:///d:/fz/0508-3/solo-dogfeeding/code/210-panel/app/Services/Servers/BuildModificationService.php#L103-L129) 有一层重要的**安全回退**：

```php
if (!empty($data['remove_allocations'])) {
    foreach ($data['remove_allocations'] as $allocation) {
        // 若用户要删除默认分配（default allocation）
        if ($allocation === ($data['allocation_id'] ?? $server->allocation_id)) {
            if (empty($freshlyAllocated)) {
                // ❌ 没有新增 Allocation 可替代 → 拒绝操作
                throw new DisplayException(
                    'You are attempting to delete the default allocation 
                     for this server but there is no fallback allocation to use.'
                );
            }
            // ✅ 自动回退到第一个新增的 Allocation 作为新 default
            $data['allocation_id'] = $freshlyAllocated;
        }
    }
}
```

### 7.3 自动部署对 Allocation 池的影响

自动部署流程使用 `getRandomAllocation()` 的 SQL 满足：

- **`server_id IS NULL`**：仅从空闲池选取
- **`whereIn('node_id', $nodes)`**：限定在容量达标节点
- **dedicated_ip 过滤**：通过 `CONCAT_WS(node_id, ip)` 排除已有服务器占用的整段 IP——即：同一 IP 下只要有一个端口被分配，整个 IP 对 dedicated 请求不可用
- **`inRandomOrder()`**：避免端口热点（但会让节点选择也变得随机）

### 7.4 Allocation 与容量计算的独立性

**重要区别**：
- **FindViableNodesService** 使用 `servers.memory / servers.disk` 聚合 → 关注**资源使用量**
- **AllocationSelectionService** 使用 `allocations` 表 → 关注**IP/端口可用性**

两者是**独立**的。存在理论边界情况：节点资源充足但所有 Allocation 都被占用 → `NoViableAllocationException`；或有空闲端口但容量不足 → `NoViableNodeException`。两种异常会分别抛出，不会自动交叉降级。

---

## 八、扩容建议：实现「资源最少节点优先」

如前所述，当前实现是**随机均衡**而非**资源最少优先**。若需改造，建议在以下两处修改：

### 改造点 1：FindViableNodesService 增加排序

在 [FindViableNodesService.php](file:///d:/fz/0508-3/solo-dogfeeding/code/210-panel/app/Services/Deployment/FindViableNodesService.php) 的 `$query->groupBy('nodes.id')` 之后增加：

```php
->orderByRaw('(IFNULL(SUM(servers.memory), 0) / (nodes.memory * (1 + (nodes.memory_overallocate / 100))) 
             + IFNULL(SUM(servers.disk), 0)   / (nodes.disk   * (1 + (nodes.disk_overallocate   / 100)))) ASC');
```

该排序按「**内存使用率 + 磁盘使用率**」之和升序排列，综合负载最低的节点排在最前。

### 改造点 2：AllocationSelectionService 优先从前若干节点选择

在 [AllocationSelectionService.php](file:///d:/fz/0508-3/solo-dogfeeding/code/210-panel/app/Services/Deployment/AllocationSelectionService.php) 中，将全部节点一次性传入改为：先尝试前 N（如前 3 个）节点，如果没有合适 Allocation，再放宽到全部节点。

```php
// 伪代码思路
$tier1 = array_slice($this->nodes, 0, 3);
$allocation = $this->repository->getRandomAllocation($tier1, $this->ports, $this->dedicated);
if (!$allocation) {
    $allocation = $this->repository->getRandomAllocation($this->nodes, $this->ports, $this->dedicated);
}
```

这样既优先填满最空闲的节点，又保证不会因头部节点端口耗尽而阻塞部署。

---

## 九、异常体系与诊断

| 异常类 | 触发场景 | 含义 |
|--------|---------|------|
| [NoViableNodeException](file:///d:/fz/0508-3/solo-dogfeeding/code/210-panel/app/Exceptions/Service/Deployment/NoViableNodeException.php) | FindViableNodesService 返回空 | 所有位置的 public 节点容量都不足 |
| [NoViableAllocationException](file:///d:/fz/0508-3/solo-dogfeeding/code/210-panel/app/Exceptions/Service/Deployment/NoViableAllocationException.php) | getRandomAllocation() 返回 null | 有容量但 IP/端口池耗尽 |
| DaemonConnectionException | Wings 无法访问 / 返回非 2xx | 下发失败，触发 ServerDeletionService 回退 |
| ValidationException (auto_deploy=false 时) | node_id / allocation_id 不存在或已占用 | 手动指定参数不合法 |
| CidrOutOfRangeException / PortOutOfRangeException | 创建 Allocation 池时 | 管理员导入 CIDR 或端口范围错误 |

排查自动部署失败的推荐顺序：
1. 查看异常类型 → 定位是「容量问题」还是「端口问题」
2. 容量问题 → 在 `nodes` 表检查 `memory / disk / memory_overallocate / disk_overallocate / public` 字段
3. 端口问题 → 在 `allocations` 表按 `node_id` 查询 `server_id IS NULL` 的记录数
4. Wings 通信问题 → 检查 Wings 日志、节点 `fqdn` / `scheme` / `daemonListen` 配置，以及防火墙
