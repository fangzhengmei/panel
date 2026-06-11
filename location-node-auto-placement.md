# Pterodactyl Panel 节点与自动放置算法代码解析

## 一、入口层：两条独立路径

Pterodactyl Panel 中创建服务器存在**两条独立且不对称**的路径：**管理端手动创建**与**Application API 自动部署**。两者在容量校验、参数入口、调用链路上都有显著差异。

### 1.1 管理端手动创建路径（Admin UI）

**入口控制器**：[CreateServerController::store()](file:///d:/fz/0508-3/solo-dogfeeding/code/210-panel/app/Http/Controllers/Admin/Servers/CreateServerController.php#L70-L83)

```php
// 第 78 行：仅传 $data，不传 DeploymentObject
$server = $this->creationService->handle($data);
```

**关键事实**：
- 管理端 UI（`new.blade.php`）**没有**「自动部署/Auto Deploy」选项，只有下拉式节点选择 + 端口分配选择
- 虽然 `ServerFormRequest::withValidator()` 中存在 `auto_deploy` 的条件验证逻辑，但 UI 上没有对应输入框，**DeploymentObject 构造完全缺失**
- 手动路径下 `node_id` 和 `allocation_id` 为必填
- **手动创建不做容量校验**：直接进入数据库事务创建服务器，不经过 FindViableNodesService

#### 管理端 auto_deploy 预留校验为何未接入 DeploymentObject

这是一个典型的「**有验证无实现**」的半成品接口。完整分析如下：

**验证层（已接入）**：[ServerFormRequest::withValidator()](file:///d:/fz/0508-3/solo-dogfeeding/code/210-panel/app/Http/Requests/Admin/ServerFormRequest.php#L26-L57) 中：
```php
$validator->sometimes('node_id', 'required|...|exists:nodes,id', function ($input) {
    return !$input->auto_deploy;  // auto_deploy=true 时 node_id 非必填
});
$validator->sometimes('allocation_id', 'required|...', function ($input) {
    return !$input->auto_deploy;  // auto_deploy=true 时 allocation_id 非必填
});
```

**控制器层（未实现）**：[CreateServerController::store()](file:///d:/fz/0508-3/solo-dogfeeding/code/210-panel/app/Http/Controllers/Admin/Servers/CreateServerController.php#L70-L83) 只传了 `$data`：
```php
$server = $this->creationService->handle($data);  // ❌ 没有传第二个参数 DeploymentObject
```

**如果有人绕过前端 UI 构造 `auto_deploy=true` 的 POST 请求**：
1. `ServerFormRequest` 验证通过（node_id / allocation_id 非必填）
2. `CreateServerController` 仍调用 `handle($data)`，不传 DeploymentObject
3. `ServerCreationService::handle()` 中 `if ($deployment instanceof DeploymentObject)` 为 **false** → 跳过 `configureDeployment()`
4. 进入后续逻辑：`if (empty($data['node_id']))` → 为 true → 触发 `Assert::false(empty($data['allocation_id']))`，**因 allocation_id 也为空而抛出 InvalidArgumentException**

所以管理端的 `auto_deploy` 目前只是「让校验规则放松」，并不会真正走自动部署链路。要接入需在 `CreateServerController::store()` 中补充 DeploymentObject 的构造逻辑。

**手动路径调用链**：
```
CreateServerController::store()
    ↓ 表单验证（ServerFormRequest）
ServerCreationService::handle($data)
    ├─ 直接使用传入的 node_id / allocation_id
    ├─ 数据库事务：创建 Server + 绑定 Allocation + 存储 Egg 变量
    └─ DaemonServerRepository::create() → 下发 Wings
        └─ 失败 → ServerDeletionService::withForce() 回退清理
```

### 1.2 Application API 自动部署路径

**入口控制器**：[ServerController::store()](file:///d:/fz/0508-3/solo-dogfeeding/code/210-panel/app/Http/Controllers/Api/Application/Servers/ServerController.php#L56-L63)

```php
// 第 58 行：第二个参数 $request->getDeploymentObject() 触发自动部署
$server = $this->creationService->handle(
    $request->validated(),
    $request->getDeploymentObject()
);
```

**部署参数解析**：[StoreServerRequest::getDeploymentObject()](file:///d:/fz/0508-3/solo-dogfeeding/code/210-panel/app/Http/Requests/Api/Application/Servers/StoreServerRequest.php#L138-L150) 从请求的 `deploy` 字段解析出 [DeploymentObject](file:///d:/fz/0508-3/solo-dogfeeding/code/210-panel/app/Models/Objects/DeploymentObject.php)：

| `deploy` 子字段 | 对应 DeploymentObject 属性 | 含义 |
|----------------|--------------------------|------|
| `deploy.locations` | `$locations` (int[]) | 候选位置 ID，空数组表示全位置 |
| `deploy.dedicated_ip` | `$dedicated` (bool) | 是否独占 IP |
| `deploy.port_range` | `$ports` (string[]) | 端口或端口范围，如 `["25565", "25570-25580"]` |

**验证规则**：`deploy` 与 `allocation.default` 互斥——有 `deploy` 时不需要指定 allocation；没有 `deploy` 时 `allocation.default` 必选。

### 1.3 两条路径的核心差异

| 维度 | 管理端手动创建 | API 自动部署 |
|------|-------------|-----------|
| 入口 | `POST /admin/servers/new` | `POST /api/application/servers` |
| 触发标志 | 无（纯手动） | `deploy` 字段存在 |
| 节点选择 | 手动指定 `node_id` | FindViableNodesService 自动筛选 |
| 端口选择 | 手动指定 `allocation_id` | AllocationSelectionService 随机抽取 |
| 容量校验 | ❌ 不做校验 | ✅ SQL 级 HAVING 校验 |
| 节点可见性 | 所有节点（含 public=0） | 仅 public=1 的节点 |
| 失败回退 | Wings 失败 → force 删除 | Wings 失败 → force 删除 |

---

## 二、第一阶段：FindViableNodesService — 候选节点筛选

[FindViableNodesService::handle()](file:///d:/fz/0508-3/solo-dogfeeding/code/210-panel/app/Services/Deployment/FindViableNodesService.php#L69-L99) 是自动部署的第一关，通过单条 SQL 完成全部筛选。

### 2.1 过滤条件（由严到宽）

```php
$query = Node::query()->select('nodes.*')
    ->selectRaw('IFNULL(SUM(servers.memory), 0) as sum_memory')
    ->selectRaw('IFNULL(SUM(servers.disk), 0) as sum_disk')
    ->leftJoin('servers', 'servers.node_id', '=', 'nodes.id')
    ->where('nodes.public', 1);        // 条件①：仅公开节点

if (!empty($this->locations)) {
    $query->whereIn('nodes.location_id', $this->locations);  // 条件②：位置过滤
}

$results = $query->groupBy('nodes.id')
    // 条件③：内存容量上限
    ->havingRaw('(IFNULL(SUM(servers.memory), 0) + ?) 
                <= (nodes.memory * (1 + (nodes.memory_overallocate / 100)))', [$this->memory])
    // 条件④：磁盘容量上限
    ->havingRaw('(IFNULL(SUM(servers.disk), 0) + ?) 
                <= (nodes.disk * (1 + (nodes.disk_overallocate / 100)))', [$this->disk]);
```

### 2.2 节点配置字段是否参与筛选（完整清单）

FindViableNodesService 是自动部署中**唯一的节点筛选关卡**，以下是所有 Node 字段的参与情况：

| Node 字段 | 是否参与自动部署筛选 | 备注 |
|-----------|-------------------|------|
| `public` | ✅ 参与 | `where('public', 1)`，私有节点直接排除 |
| `location_id` | ✅ 参与 | `whereIn(location_id, ...)`，空数组时全位置 |
| `memory` | ✅ 参与 | 容量上限计算 |
| `disk` | ✅ 参与 | 容量上限计算 |
| `memory_overallocate` | ✅ 参与 | 容量倍率计算（-1 时语义不一致，见 2.3） |
| `disk_overallocate` | ✅ 参与 | 容量倍率计算 |
| `maintenance_mode` | ❌ **不参与** | SQL 中无此条件，维护模式的节点仍可能被选中 |
| `cpu` | ❌ **不参与** | 仅作为服务器参数写入创建时使用 |
| `swap` | ❌ **不参与** | 仅作为服务器参数写入创建时使用 |
| `io` | ❌ **不参与** | 仅作为服务器参数写入创建时使用 |
| `threads` | ❌ **不参与** | 仅作为服务器参数写入创建时使用 |
| `daemon_token` / `daemonListen` | ❌ 不参与 | 用于 Wings 通信，不参与筛选 |
| `behind_proxy` / `scheme` / `fqdn` | ❌ 不参与 | 网络连接配置，不参与筛选 |

> **⚠️ 重要**：`maintenance_mode = true`（维护模式）的节点**仍然**可以被自动部署选中。面板中虽然有 `Node::isUnderMaintenance()` 方法，但 FindViableNodesService 中并未调用或过滤。创建成功后 Wings 是否拒绝启动服务器取决于 Wings 端实现，Panel 的自动放置层不会做维护模式拦截。

### 2.3 容量计算公式

**通用公式**：

```
已用资源 + 新增请求资源 ≤ 节点基础容量 × (1 + overallocate 百分比 / 100)
```

**已用资源**：通过 `LEFT JOIN servers + SUM()` 聚合该节点下所有服务器的 memory/disk 字段之和。**所有状态的服务器都计入**（suspended / installing / install_failed 全部占用配额）。

### 2.4 overallocate 值的完整语义

`memory_overallocate` 与 `disk_overallocate` 的设计意图与代码实现存在**不一致**，需要特别注意。

#### 设计意图（文档/注释层面）

在 [nodes/new.blade.php](file:///d:/fz/0508-3/solo-dogfeeding/code/210-panel/resources/views/admin/nodes/new.blade.php#L124) 注释中明确说明：
> "To disable checking for overallocation enter `-1` into the field."

[MakeNodeCommand.php](file:///d:/fz/0508-3/solo-dogfeeding/code/210-panel/app/Console/Commands/Node/MakeNodeCommand.php#L58) 也提到：
> "-1 will disable checking"

即设计意图是：**`-1` = 禁用容量检查 = 无限超售**。

#### 代码实际行为

| overallocate 值 | FindViableNodesService / Node::isViable() | NodeRepository 统计显示 | 与设计意图是否一致 |
|----------------|------------------------------------------|----------------------|----------------|
| `-1` | 上限 = 基础 × 0.99（更严格） | 上限 = 基础 × 1.0（与 0 相同） | ❌ 不一致，且三处互异 |
| `0`  | 上限 = 基础 × 1.0（不超售） | 上限 = 基础 × 1.0 | ✅ 一致 |
| `50` | 上限 = 基础 × 1.5 | 上限 = 基础 × 1.5 | ✅ 一致 |
| `100` | 上限 = 基础 × 2.0 | 上限 = 基础 × 2.0 | ✅ 一致 |

**三处实现的具体代码**：

1. **容量判断（FindViableNodesService SQL + Node::isViable）**：
   ```php
   // 无条件地使用公式 1 + (overallocate / 100)
   $memoryLimit = $this->memory * (1 + ($this->memory_overallocate / 100));
   ```
   当 overallocate = -1 时，结果为 `0.99 × 基础容量`，相当于「比不超售还严格 1%」。

2. **统计显示（NodeRepository::getUsageStats）**：
   ```php
   if ($node->{$key . '_overallocate'} > 0) {  // 注意是 > 0
       $maxUsage = $node->{$key} * (1 + ($node->{$key . '_overallocate'} / 100));
   }
   // overallocate <= 0 时，maxUsage = 基础容量
   ```
   当 overallocate = -1 时，条件 `> 0` 不成立，max 显示为基础容量（与 0 相同）。

> **⚠️ 重要结论**：`overallocate = -1` 在当前代码中既不是「无限超售」（设计意图），也不是一个固定且自洽的行为——容量判断是 99% 上限，UI 显示是 100% 上限，二者不一致。运维在使用 `-1` 值时需特别谨慎。

### 2.5 筛选结果

- 结果集包含 `sum_memory`、`sum_disk` 两个动态属性（通过 selectRaw 注入）
- 结果**未做排序**，默认按 `GROUP BY` 顺序（通常即节点主键升序）
- 空结果集 → 抛出 [NoViableNodeException](file:///d:/fz/0508-3/solo-dogfeeding/code/210-panel/app/Exceptions/Service/Deployment/NoViableNodeException.php)
- 支持分页（`handle($perPage, $page)`），供 API 列表查询端点使用

---

## 三、部署预览 API 与真实创建链路的不一致

在自动部署流程开始前，调用方通常会先请求「可部署节点预览 API」确认有可用节点。需要特别注意的是：**预览 API 使用的筛选条件与真实创建链路存在显著差异**，这是「预览成功但最终仍失败」的核心原因。

### 3.1 部署预览 API 的入口与参数

**路由**：`GET /api/application/nodes/deployable`（[routes/api-application.php:36](file:///d:/fz/0508-3/solo-dogfeeding/code/210-panel/routes/api-application.php#L36)）

**控制器**：[NodeDeploymentController::__invoke()](file:///d:/fz/0508-3/solo-dogfeeding/code/210-panel/app/Http/Controllers/Api/Application/Nodes/NodeDeploymentController.php#L27-L38)

```php
// 只接收三个参数：location_ids、memory、disk
$nodes = $this->viableNodesService->setLocations($data['location_ids'] ?? [])
    ->setMemory($data['memory'])
    ->setDisk($data['disk'])
    ->handle($request->query('per_page'), $request->query('page'));
```

**请求验证**：[GetDeployableNodesRequest](file:///d:/fz/0508-3/solo-dogfeeding/code/210-panel/app/Http/Requests/Api/Application/Nodes/GetDeployableNodesRequest.php#L7-L16) 规则中：
```php
'page' => 'integer',
'memory' => 'required|integer|min:0',
'disk'   => 'required|integer|min:0',
'location_ids' => 'array',
'location_ids.*' => 'integer',
```

**没有** `dedicated_ip`、**没有** `port_range` 参数。

### 3.2 两阶段筛选参数对比

自动部署分为两个阶段执行，预览 API 只执行了第一阶段：

| 参数 | 预览 API（阶段①） | 真实创建阶段① FindViableNodesService | 真实创建阶段② AllocationSelectionService |
|------|-----------------|-----------------------------------|--------------------------------------|
| `locations` | ✅ | ✅ | ❌（用阶段①的结果） |
| `memory` | ✅ | ✅ | ❌ |
| `disk` | ✅ | ✅ | ❌ |
| `public=1` | ✅（隐含在 FVNS） | ✅ | ❌ |
| `dedicated_ip` | ❌ **未传** | ❌ 不处理 | ✅ setDedicated() |
| `port_range` | ❌ **未传** | ❌ 不处理 | ✅ setPorts() |
| `maintenance_mode` | ❌ | ❌ 不处理 | ❌ 不处理 |

**结论**：预览 API 只做了「节点容量维度」的检查，完全没有做「IP 端口维度」的检查。

### 3.3 dedicated_ip / port_range 在哪一层才生效

答案：**只在 AllocationSelectionService（阶段②）才生效**，FindViableNodesService（阶段①）和预览 API 都不处理。

生效链路：
```
StoreServerRequest::getDeploymentObject()
    ├─ 从 deploy.dedicated_ip 解析出 $dedicated
    └─ 从 deploy.port_range 解析出 $ports
            ↓
ServerCreationService::configureDeployment()
    └─ AllocationSelectionService
        ├─ setDedicated($dedicated)    → 进入阶段②
        └─ setPorts($ports)            → 进入阶段②
                ↓
AllocationRepository::getRandomAllocation()
    ├─ 条件③：WHERE port IN / BETWEEN ...（port_range）
    └─ 条件④：WHERE CONCAT(node_id, ip) NOT IN (dedicated_ip)
```

### 3.4 为什么「预览通过后，最终仍可能 allocation 失败」

有四重原因，按发生概率从高到低排列：

**原因 ①：dedicated_ip / port_range 在预览阶段不检查（主要原因）**

预览 API 根本没有接收和处理这两个参数。举例：
- 用户请求：`memory=1024, disk=10240, dedicated_ip=true, port_range=["25565"]`
- 预览 API：只查 `memory/disk/location` → 显示有 3 个节点可用
- 真实创建：到阶段②时，用了 `dedicated_ip` 和 `port_range` 过滤 → 发现没有 IP 能满足独占 / 25565 端口已占满 → `NoViableAllocationException`

**原因 ②：竞争条件（Race Condition）**

预览与真实创建之间存在时间差（可能几秒到几分钟），期间其他部署请求可能：
- 占满了原本可用的容量（其他请求也在这个节点开了新服）→ 阶段①失败
- 占用了原本空闲的 Allocation → 阶段②失败

**原因 ③：节点结果集分页截断**

预览 API 支持分页（默认 50 条/页），如果满足容量的节点很多且调用方只取了前 50 个，刚好满足端口条件的节点可能落在后续页未被展示。

**原因 ④：整体只有部分节点满足两个阶段（交集为空）**

集合 A = {满足容量的节点}，集合 B = {满足端口/IP 的节点}。预览只显示 A ∪ (未做端口校验的全集)，真实创建要求属于 A ∩ B——交集可能为空，即使 A、B 各自都非空。

---

## 四、第二阶段：AllocationSelectionService — 端口与 IP 抽取

[AllocationSelectionService::handle()](file:///d:/fz/0508-3/solo-dogfeeding/code/210-panel/app/Services/Deployment/AllocationSelectionService.php#L85-L94) 将上一阶段筛选出的节点 ID 列表传入 AllocationRepository，抽取一个空闲 Allocation 作为服务器主分配。

### 4.1 抽取算法（AllocationRepository::getRandomAllocation）

[AllocationRepository::getRandomAllocation()](file:///d:/fz/0508-3/solo-dogfeeding/code/210-panel/app/Repositories/Eloquent/AllocationRepository.php#L59-L99) 逐步收窄范围：

```php
$query = Allocation::query()->whereNull('server_id');      // 条件①：未分配

if (!empty($nodes)) {
    $query->whereIn('node_id', $nodes);                    // 条件②：候选节点
}

if (!empty($ports)) {
    $query->where(function (Builder $inner) use ($ports) { // 条件③：端口范围
        foreach ($ports as $port) {
            if (is_array($port)) {
                $inner->orWhereBetween('port', $port);      // 端口范围 BETWEEN
            } else {
                $whereIn[] = $port;                         // 单端口 IN
            }
        }
        if (!empty($whereIn)) $inner->orWhereIn('port', $whereIn);
    });
}

if ($dedicated) {                                           // 条件④：独占 IP
    $discard = $this->getDiscardableDedicatedAllocations($nodes);
    if (!empty($discard)) {
        // 排除已有服务器占用的 (node_id, ip) 组合
        $query->whereNotIn(
            $this->getBuilder()->raw('CONCAT_WS("-", node_id, ip)'),
            $discard
        );
    }
}

return $query->inRandomOrder()->first();                    // ⚠️ 随机取第一条
```

### 4.2 关键机制说明

**独占 IP（dedicated_ip）的判定粒度**：
- 通过 `CONCAT_WS("-", node_id, ip)` 组合键判断
- 同一 IP 下，只要有任意一个端口已分配给某个服务器，整个 IP 段对 dedicated 请求失效
- 粒度是「每个 IP 地址」，不是「每个节点」也不是「每个端口段」

**随机选择的实际效果**：
- 使用 `inRandomOrder()` 完全随机
- 空闲 Allocation 条目数多的节点，被抽中的概率更高（统计意义上的间接均衡）
- **不按「资源使用率」排序**，即「资源最少的节点优先」在当前代码中并未实现

### 4.3 失败条件

- 返回 `null` → 抛出 [NoViableAllocationException](file:///d:/fz/0508-3/solo-dogfeeding/code/210-panel/app/Exceptions/Service/Deployment/NoViableAllocationException.php)
- 场景：节点容量够，但所有满足条件的 IP/端口都被占满了
- 与 NoViableNodeException 是**独立**的两层异常，不会自动降级

---

## 五、第三阶段：数据库持久化与 Wings 下发

### 5.1 ServerCreationService 主流程

[ServerCreationService::handle()](file:///d:/fz/0508-3/solo-dogfeeding/code/210-panel/app/Services/Servers/ServerCreationService.php#L52-L107) 的执行顺序是：**先写库，后调 Wings**。

```php
// 步骤 1：数据库事务（重试 5 次）
$server = $this->connection->transaction(function () use ($data, $eggVariableData) {
    $server = $this->createModel($data);              // INSERT servers
    $this->storeAssignedAllocations($server, $data);   // UPDATE allocations SET server_id
    $this->storeEggVariables($server, $eggVariableData); // INSERT server_variables
    return $server;
}, 5);

// 步骤 2：调用 Wings Daemon
try {
    $this->daemonServerRepository->setServer($server)->create(
        Arr::get($data, 'start_on_completion', false) ?? false
    );
} catch (DaemonConnectionException $exception) {
    // ⚠️ 失败回退：强制删除已持久化的服务器
    $this->serverDeletionService->withForce()->handle($server);
    throw $exception;
}
```

### 5.2 Wings 下发细节

[DaemonServerRepository::create()](file:///d:/fz/0508-3/solo-dogfeeding/code/210-panel/app/Repositories/Wings/DaemonServerRepository.php#L42-L56) 只发送基础信息：

```
POST {node.getConnectionAddress()}/api/servers
{
    "uuid": "<server-uuid>",
    "start_on_completion": true/false
}
```

Wings 收到后会**主动回拉**完整配置（资源限制、挂载、启动命令、Egg 变量等），所以此处不传细节。

---

## 六、失败回退：ServerDeletionService 分级处理

当 Wings 调用失败时，`ServerCreationService` 会调用 `ServerDeletionService::withForce()->handle($server)` 进行清理。

[ServerDeletionService::handle()](file:///d:/fz/0508-3/solo-dogfeeding/code/210-panel/app/Services/Servers/ServerDeletionService.php#L43-L86) 采用「**三级降级**」策略：

### 6.1 第一级：Wings 端删除（可降级）

```php
try {
    $this->daemonServerRepository->setServer($server)->delete();
} catch (DaemonConnectionException $exception) {
    if (!$this->force && $exception->getStatusCode() !== Response::HTTP_NOT_FOUND) {
        throw $exception;  // 非 force 且非 404 → 中止删除
    }
    Log::warning($exception); // force 模式或 404 → 记日志，继续
}
```

| 场景 | 非 force 模式 | force 模式 |
|------|-------------|-----------|
| Wings 正常删除（2XX） | ✅ 继续 | ✅ 继续 |
| Wings 返回 404 | ✅ 继续（视为已不存在） | ✅ 继续 |
| Wings 连接失败 / 5XX / 其他错误 | ❌ 抛出异常，中止删除 | ✅ 记 warning 日志，继续面板清理 |

### 6.2 第二级：数据库 host 上的数据库（可降级）

```php
foreach ($server->databases as $database) {
    try {
        $this->databaseManagementService->delete($database); // 真实删 host 上的 DB
    } catch (\Exception $exception) {
        if (!$this->force) {
            throw $exception;  // 非 force → 中止
        }
        $database->delete();    // force → 只删面板记录，host 上留 dangling DB
        Log::warning($exception);
    }
}
```

force 模式下会留下「悬挂数据库」（dangling database），这是已知行为（见代码注释中的 issue #2085）。

### 6.3 第三级：面板数据库清理（必执行）

```php
// 清空 Allocation 备注（防止信息泄露）
$server->allocations()->update(['notes' => null]);

// 删除 Server 记录（级联影响：
//   - allocations.server_id → 通过外键/模型事件置空
//   - server_variables / subusers / schedules 等关联数据级联删除
// )
$server->delete();
```

> **注意**：Allocation 的 `server_id` 被置空后，该端口重新回到可分配池中。notes 被清空是为了防止前一个服务器的备注信息泄露给后续分配到该端口的服务器。

---

## 七、与现有 Allocation 的相互影响

### 7.1 Allocation 生命周期

| 阶段 | `allocations.server_id` | 触发方式 |
|------|----------------------|---------|
| 初始创建 | `NULL` | [AssignmentService::handle()](file:///d:/fz/0508-3/solo-dogfeeding/code/210-panel/app/Services/Allocations/AssignmentService.php) 批量导入 |
| 分配给服务器 | `= server.id` | `storeAssignedAllocations()` → `UPDATE allocations SET server_id = ?` |
| 服务器删除 | → `NULL` | ServerDeletionService 事务中 `$server->delete()` 级联 |
| Build 修改（增/减端口） | 切换 | [BuildModificationService::processAllocations()](file:///d:/fz/0508-3/solo-dogfeeding/code/210-panel/app/Services/Servers/BuildModificationService.php#L82-L130) |

### 7.2 Default Allocation 的安全回退

在修改服务器 Build 配置（增减 Allocation）时，有一层**重要的安全回退**机制：

[BuildModificationService::processAllocations()](file:///d:/fz/0508-3/solo-dogfeeding/code/210-panel/app/Services/Servers/BuildModificationService.php#L103-L129)：

```php
// 如果用户试图删除默认分配（default allocation）
if ($allocation === ($data['allocation_id'] ?? $server->allocation_id)) {
    if (empty($freshlyAllocated)) {
        // ❌ 没有可替代的新分配 → 拒绝操作
        throw new DisplayException(
            'You are attempting to delete the default allocation 
             for this server but there is no fallback allocation to use.'
        );
    }
    // ✅ 自动回退到第一个新增的 Allocation 作为新 default
    $data['allocation_id'] = $freshlyAllocated;
}
```

即：删除默认端口时，如果同时有新增端口，自动用第一个新增端口作为新的默认端口；没有新增端口则禁止删除。

### 7.3 自动部署对 Allocation 池的影响

自动部署使用 `getRandomAllocation()` 从池中随机抽取，对 Allocation 池的影响与手动分配一致：
- 选中后通过 `storeAssignedAllocations()` 标记 `server_id`
- 服务器删除后自动回到池中
- dedicated_ip 模式下，分配后该 IP 对其他 dedicated 请求不可见

### 7.4 容量计算与 Allocation 的独立性

**两个维度相互独立**：
- **容量维度**（memory / disk）→ 通过 `servers` 表聚合计算，与 Allocation 数量无关
- **端口维度**（IP / port）→ 通过 `allocations` 表判断，与服务器资源无关

可能出现的边界场景：
- 「容量够但端口不够」→ `NoViableAllocationException`
- 「端口够但容量不够」→ `NoViableNodeException`
- 两种异常独立抛出，不会交叉降级

---

## 八、完整调用链总图

```
┌───────────────────────────────────────────────────────────────────┐
│                     Application API 自动部署路径                   │
├───────────────────────────────────────────────────────────────────┤
│  POST /api/application/servers                                    │
│    (带 deploy: { locations, dedicated_ip, port_range })          │
│         ↓                                                          │
│  StoreServerRequest::validate()                                   │
│    └─ getDeploymentObject() → DeploymentObject                   │
│         ↓                                                          │
│  ServerController::store()                                        │
│         ↓                                                          │
│  ServerCreationService::handle($data, $deployment)                │
│    ├─ configureDeployment()                                       │
│    │   ├─ FindViableNodesService                                  │
│    │   │   └─ SQL: public=1 + location IN + capacity HAVING       │
│    │   │      → 返回 Node 集合（含 sum_memory / sum_disk）         │
│    │   └─ AllocationSelectionService                              │
│    │       └─ AllocationRepository::getRandomAllocation()         │
│    │          → 返回单个随机 Allocation                            │
│    ├─ DB 事务（重试 5 次）                                         │
│    │   ├─ createModel() → INSERT servers                         │
│    │   ├─ storeAssignedAllocations() → UPDATE allocations        │
│    │   └─ storeEggVariables() → INSERT server_variables          │
│    └─ DaemonServerRepository::create() → POST Wings              │
│       └─ 失败 → ServerDeletionService::withForce() 回退清理       │
└───────────────────────────────────────────────────────────────────┘

┌───────────────────────────────────────────────────────────────────┐
│                      管理端 UI 手动创建路径                        │
├───────────────────────────────────────────────────────────────────┤
│  POST /admin/servers/new                                          │
│    (带 node_id + allocation_id + allocation_additional)           │
│         ↓                                                          │
│  ServerFormRequest::validate()                                    │
│    └─ auto_deploy 字段预留（UI 上不存在）                           │
│         ↓                                                          │
│  CreateServerController::store()                                  │
│         ↓                                                          │
│  ServerCreationService::handle($data)                             │
│    ├─ 直接使用传入的 node_id / allocation_id                      │
│    ├─ ❌ 不经过 FindViableNodesService（无容量校验）               │
│    ├─ DB 事务（重试 5 次）                                         │
│    └─ DaemonServerRepository::create() → POST Wings              │
│       └─ 失败 → ServerDeletionService::withForce() 回退清理       │
└───────────────────────────────────────────────────────────────────┘
```

---

## 九、异常诊断速查表

| 异常 | 触发阶段 | 根因方向 | 排查命令/字段 |
|------|---------|---------|-------------|
| `NoViableNodeException` | FindViableNodesService | 所有 public 节点容量不足 | `SELECT memory, disk, memory_overallocate, disk_overallocate, public FROM nodes` |
| `NoViableAllocationException` | AllocationSelectionService | 合格节点的空闲端口耗尽 | `SELECT COUNT(*) FROM allocations WHERE node_id IN (...) AND server_id IS NULL` |
| `DaemonConnectionException` | Wings 调用 | Wings 不可达 / 返回错误 | 检查 Wings 日志、`daemonListen`、`scheme`、防火墙 |
| `ValidationException` | 请求验证 | 参数缺失 / 不合法 | 检查 `deploy` 与 `allocation` 是否互斥 |
| `DataValidationException` | Model 保存 | 数据不满足字段规则 | 查看 Server 模型 `getRules()` |

---

## 十、关键文件索引

| 文件 | 职责 |
|------|------|
| [FindViableNodesService.php](file:///d:/fz/0508-3/solo-dogfeeding/code/210-panel/app/Services/Deployment/FindViableNodesService.php) | 节点容量筛选（SQL HAVING） |
| [AllocationSelectionService.php](file:///d:/fz/0508-3/solo-dogfeeding/code/210-panel/app/Services/Deployment/AllocationSelectionService.php) | Allocation 抽取封装（dedicated/ports 生效层） |
| [AllocationRepository.php](file:///d:/fz/0508-3/solo-dogfeeding/code/210-panel/app/Repositories/Eloquent/AllocationRepository.php) | getRandomAllocation() 实际算法 |
| [NodeDeploymentController.php](file:///d:/fz/0508-3/solo-dogfeeding/code/210-panel/app/Http/Controllers/Api/Application/Nodes/NodeDeploymentController.php) | 部署预览 API（仅做容量检查） |
| [GetDeployableNodesRequest.php](file:///d:/fz/0508-3/solo-dogfeeding/code/210-panel/app/Http/Requests/Api/Application/Nodes/GetDeployableNodesRequest.php) | 预览 API 请求验证（无 dedicated/ports 参数） |
| [ServerCreationService.php](file:///d:/fz/0508-3/solo-dogfeeding/code/210-panel/app/Services/Servers/ServerCreationService.php) | 创建编排 + Wings 调用 + 失败回退 |
| [ServerDeletionService.php](file:///d:/fz/0508-3/solo-dogfeeding/code/210-panel/app/Services/Servers/ServerDeletionService.php) | 删除与 force 三级降级逻辑 |
| [Node.php](file:///d:/fz/0508-3/solo-dogfeeding/code/210-panel/app/Models/Node.php) | Node 模型 + isViable() 方法 + maintenance_mode 方法 |
| [NodeRepository.php](file:///d:/fz/0508-3/solo-dogfeeding/code/210-panel/app/Repositories/Eloquent/NodeRepository.php) | 使用率统计（与容量判断公式不一致） |
| [DeploymentObject.php](file:///d:/fz/0508-3/solo-dogfeeding/code/210-panel/app/Models/Objects/DeploymentObject.php) | 部署参数对象 |
| [StoreServerRequest.php](file:///d:/fz/0508-3/solo-dogfeeding/code/210-panel/app/Http/Requests/Api/Application/Servers/StoreServerRequest.php) | API 层 deploy 参数解析（含 dedicated_ip/port_range） |
| [ServerFormRequest.php](file:///d:/fz/0508-3/solo-dogfeeding/code/210-panel/app/Http/Requests/Admin/ServerFormRequest.php) | 管理端 auto_deploy 验证（有验证无部署对象实现） |
| [CreateServerController.php](file:///d:/fz/0508-3/solo-dogfeeding/code/210-panel/app/Http/Controllers/Admin/Servers/CreateServerController.php) | 管理端创建入口（未接入 DeploymentObject） |
| [ServerController.php](file:///d:/fz/0508-3/solo-dogfeeding/code/210-panel/app/Http/Controllers/Api/Application/Servers/ServerController.php) | API 服务器创建入口 |
| [DaemonServerRepository.php](file:///d:/fz/0508-3/solo-dogfeeding/code/210-panel/app/Repositories/Wings/DaemonServerRepository.php) | Wings Daemon 通信 |
| [DaemonConnectionException.php](file:///d:/fz/0508-3/solo-dogfeeding/code/210-panel/app/Exceptions/Http/Connection/DaemonConnectionException.php) | Wings 连接异常（含 getStatusCode） |
| [NoViableNodeException.php](file:///d:/fz/0508-3/solo-dogfeeding/code/210-panel/app/Exceptions/Service/Deployment/NoViableNodeException.php) | 节点容量不足异常 |
| [NoViableAllocationException.php](file:///d:/fz/0508-3/solo-dogfeeding/code/210-panel/app/Exceptions/Service/Deployment/NoViableAllocationException.php) | 端口分配失败异常 |
