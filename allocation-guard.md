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

## 五、三者协作关系

### 5.1 整体流程

```
候选筛选 → 占用校验 → 分配绑定 → 使用中 → 回收释放
    ↓          ↓          ↓                    ↓
1. 节点筛选   1. DB唯一键  1. 事务保护        1. 外键自动置空
2. 分配筛选   2. 应用层检查  2. 行锁防并发      2. 主动置空 server_id
3. 专用IP检查  3. 行锁      3. 更新 server_id   3. 清空 notes
```

### 5.2 典型场景协作

#### 场景1：自动部署服务器

**流程**：
1. `ServerCreationService::configureDeployment()` 调用 `FindViableNodesService` 筛选满足资源的节点
2. 调用 `AllocationSelectionService::handle()` → `AllocationRepository::getRandomAllocation()` 筛选可用分配
3. 在事务中创建服务器并调用 `storeAssignedAllocations()` 批量更新 `server_id`
4. 若Daemon创建失败，调用 `ServerDeletionService` 回滚，外键自动置空分配

**涉及文件**：
- `app/Services/Servers/ServerCreationService.php:56-60, 86-94, 116-128`

#### 场景2：客户端自助添加分配

**流程**：
1. `NetworkAllocationController::store()` 检查分配限额
2. `FindAssignableAllocationService::handle()`:
   - 使用 `lockForUpdate()` 锁定查询，防止并发
   - 找到现有未分配 → 直接更新 `server_id`
   - 找不到 → 计算可用端口差集 → `insertIgnore` 创建 → 更新 `server_id`

**涉及文件**：
- `app/Http/Controllers/Api/Client/Servers/NetworkAllocationController.php:95-112`
- `app/Services/Allocations/FindAssignableAllocationService.php:31-53`

#### 场景3：修改服务器构建（增减分配）

**流程**：
1. `BuildModificationService::processAllocations()` 处理 `add_allocations` 和 `remove_allocations`
2. 添加分配：查询 `server_id IS NULL` 的分配，更新 `server_id`
3. 移除分配：检查主分配保护，更新 `server_id = NULL` 并清空 `notes`
4. 整个过程在事务中执行

**涉及文件**：
- `app/Services/Servers/BuildModificationService.php:82-130`

### 5.3 关键协作点

| 机制 | 筛选阶段 | 校验阶段 | 回收阶段 |
|------|---------|---------|---------|
| **数据库约束** | - | 唯一索引、外键 | ON DELETE SET NULL |
| **应用层检查** | whereNull('server_id') | whereNull('server_id') | server_id 非空检查 |
| **并发控制** | - | lockForUpdate() | - |
| **事务保护** | - | 创建/修改事务 | 删除事务 |
| **数据清理** | - | - | 清空 notes |

---

## 六、设计亮点与潜在风险

### 设计亮点

1. **多层防御**：数据库唯一索引 + 应用层检查 + 行锁，确保万无一失
2. **自动回收**：外键 `ON DELETE SET NULL` 确保服务器删除时分配自动释放
3. **随机分配**：`inRandomOrder()` 避免端口集中在某一区域
4. **幂等操作**：`insertIgnore` 支持重复调用，适合批量场景
5. **资源隔离**：专用IP模式确保敏感应用独占IP

### 潜在风险与注意事项

1. **长事务风险**：`FindAssignableAllocationService` 中的 `lockForUpdate()` 可能导致锁等待
2. **差集计算性能**：`array_diff(range($start, $end), $ports->toArray())` 在端口范围大时可能有性能问题
3. **随机分配冲突**：高并发下 `inRandomOrder()` + `first()` 可能导致多次重试
4. **Notes 泄露**：回收时必须清空 `notes`，否则可能携带前一服务器的敏感信息

### 关键代码溯源

| 功能 | 文件位置 | 行号 |
|------|---------|------|
| 节点资源筛选 | `FindViableNodesService.php` | 74-86 |
| 分配随机获取 | `AllocationRepository.php` | 59-98 |
| 自动分配逻辑 | `FindAssignableAllocationService.php` | 31-112 |
| 插入忽略重复 | `EloquentRepository.php` | 251-277 |
| 构建修改分配 | `BuildModificationService.php` | 82-130 |
| 服务器删除回收 | `ServerDeletionService.php` | 59-85 |
| 数据库唯一键 | `mysql-schema.sql` | 54 |
| 外键置空约束 | `mysql-schema.sql` | 57 |
