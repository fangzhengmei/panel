# Pterodactyl Panel 服务器挂起（Suspension）资源限制说明

> **标注规则**：本文档严格区分以下两种声明，请客服注意边界。
>
> - **✅ 代码事实**：在 Panel 代码中可直接验证的逻辑
> - **⚠️ 推断/行为**：依赖 Wings 守护进程或业务系统，Panel 代码无法直接验证

---

## 一、挂起触发入口 ✅ 代码事实

### 1.1 三种触发方式

| 触发方式 | 代码位置 | 说明 |
|---------|---------|------|
| 管理后台界面 | [manage.blade.php](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/resources/views/admin/servers/view/manage.blade.php#L57-L93) | 管理员在"Manage"页点击按钮，POST `action=suspend\|unsuspend` 到 [ServersController::manageSuspension()](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/app/Http/Controllers/Admin/ServersController.php#L125-L133) |
| 应用 API（挂起） | [ServerManagementController.php](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/app/Http/Controllers/Api/Application/Servers/ServerManagementController.php#L29-L34) | `POST /api/application/servers/{server}/suspend` |
| 应用 API（解除挂起） | [ServerManagementController.php](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/app/Http/Controllers/Api/Application/Servers/ServerManagementController.php#L41-L46) | `POST /api/application/servers/{server}/unsuspend` |

### 1.2 前置检查 ✅ 代码事实

三种方式最终都调用 [SuspensionService::toggle()](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/app/Services/Servers/SuspensionService.php#L28-L60)，执行两项检查：

1. **状态去重**：已挂起时再次挂起 / 未挂起时解除挂起 → 直接返回，不操作
2. **转移拦截**：服务器正在转移中（`$server->transfer` 非空）→ 抛 `ConflictHttpException`

```php
// SuspensionService.php 第 36-43 行
if ($isSuspending === $server->isSuspended()) {
    return;
}
if (!is_null($server->transfer)) {
    throw new ConflictHttpException('Cannot toggle suspension status on a server that is currently being transferred.');
}
```

> **注意** ✅ 代码事实：SuspensionService 中**没有**调用 Activity Log，挂起/解除挂起操作**不记录操作日志**。[ServersController::manageSuspension()](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/app/Http/Controllers/Admin/ServersController.php#L125-L133) 也仅显示 flash 提示，无持久化日志。

---

## 二、状态变更执行流程

### 2.1 状态字段 ✅ 代码事实

挂起状态通过 `servers.status` 字段标识，常量定义于 [Server.php 第 125 行](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/app/Models/Server.php#L122-L126)：

| 状态 | `status` 字段值 |
|-----|----------------|
| 正常 | `null` |
| 挂起 | `'suspended'`（`Server::STATUS_SUSPENDED`） |

**校验辅助方法** ✅ 代码事实：[Server::isSuspended()](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/app/Models/Server.php#L218-L221)
```php
public function isSuspended(): bool
{
    return $this->status === self::STATUS_SUSPENDED;
}
```

> **重要** ✅ 代码事实：`servers` 表中**没有** `suspended_at`、`unsuspended_at` 等时间字段。数据库迁移 [2016_09_01_193520_add_suspension_for_servers.php](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/database/migrations/2016_09_01_193520_add_suspension_for_servers.php) 只添加了 `suspended` 布尔列（后被替换为 `status` 通用列，见 [2021_01_17_152623_add_generic_server_status_column.php](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/database/migrations/2021_01_17_152623_add_generic_server_status_column.php)）。

### 2.2 完整执行步骤

```
① 更新数据库 status 字段                 ← ✅ Panel 代码事实
      ↓
② 向 Wings 发送空 POST /sync             ← ✅ Panel 代码事实
      ↓
③ ⚠️ Wings 收到后主动拉取服务器配置        ← ⚠️ Wings 推断
      ↓
④ Panel 返回配置，含 suspended:true       ← ✅ Panel 代码事实
      ↓
⑤ ⚠️ Wings 执行挂起操作（停容器等）        ← ⚠️ Wings 推断
      ↓
⑥ Wings HTTP 成功 → 流程完成
   Wings HTTP 失败 → 回滚数据库并抛异常    ← ✅ Panel 代码事实
```

**步骤 ① 更新数据库** ✅ 代码事实：[SuspensionService.php 第 46-48 行](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/app/Services/Servers/SuspensionService.php#L46-L48)
```php
$server->update([
    'status' => $isSuspending ? Server::STATUS_SUSPENDED : null,
]);
```

**步骤 ② 发送 sync 请求** ✅ 代码事实：[DaemonServerRepository::sync()](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/app/Repositories/Wings/DaemonServerRepository.php#L63-L72)
```php
// 注意：POST 没有请求体（无 $data 参数），只触发 Wings
public function sync(): void
{
    $this->getHttpClient()->post("/api/servers/{$this->server->uuid}/sync");
}
```

**步骤 ④ Panel 返回配置** ✅ 代码事实：Wings 通过 `GET /api/remote/servers/{uuid}` 拉取，由 [ServerDetailsController](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/app/Http/Controllers/Api/Remote/Servers/ServerDetailsController.php#L38-L60) 返回：

```json
{
  "settings": {
    "uuid": "...",
    "suspended": true,
    "environment": { ... },
    "build": { ... },
    ...
  },
  "process_configuration": { ... }
}
```

`settings` 结构由 [ServerConfigurationStructureService](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/app/Services/Servers/ServerConfigurationStructureService.php) 生成，`suspended` 字段在第 51 行。

**步骤 ⑥ 失败回滚** ✅ 代码事实：[SuspensionService.php 第 53-59 行](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/app/Services/Servers/SuspensionService.php#L53-L59)
```php
try {
    $this->daemonServerRepository->setServer($server)->sync();
} catch (\Exception $exception) {
    // 回滚到操作前的状态
    $server->update([
        'status' => $isSuspending ? null : Server::STATUS_SUSPENDED,
    ]);
    throw $exception;
}
```

> **参考**：管理后台界面 [manage.blade.php 第 64 行](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/resources/views/admin/servers/view/manage.blade.php#L64) 对挂起操作的官方描述：
>
> *"This will suspend the server, stop any running processes, and immediately block the user from being able to access their files or otherwise manage the server through the panel or API."*

---

## 三、挂起期间的访问限制

### 3.1 访问控制总入口 ✅ 代码事实

客户端 API 所有服务器路由经过中间件 [AuthenticateServerAccess](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/app/Http/Middleware/Api/Client/Server/AuthenticateServerAccess.php)，核心调用 [Server::validateCurrentState()](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/app/Models/Server.php#L390-L401)：

```php
// validateCurrentState() 同时检查以下所有异常状态
if (
    $this->isSuspended()          // ← 已挂起
    || $this->node->isUnderMaintenance()
    || !$this->isInstalled()
    || $this->status === self::STATUS_RESTORING_BACKUP
    || !is_null($this->transfer)
) {
    throw new ServerStateConflictException($this);
}
```

`validateCurrentState()` 被调用的位置 ✅ 代码事实：
1. 客户端 API 中间件（所有服务器路由）
2. SFTP 认证（[SftpAuthenticationController.php 第 153 行](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/app/Http/Controllers/Api/Remote/SftpAuthenticationController.php#L153)）
3. 备份恢复控制器（[BackupController.php 第 211 行](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/app/Http/Controllers/Api/Client/Servers/BackupController.php#L198-L229)，双重保险）

### 3.2 客户端 API 白名单 ✅ 代码事实

挂起状态下，只有两个接口可访问，其余全部拒绝并返回 `409 Conflict`：

| 路由名称 | 端点 | 说明 |
|---------|------|------|
| `api:client:server.view` | `GET /api/client/servers/{server}` | 查看服务器基本信息 |
| `api:client:server.resources` | `GET /api/client/servers/{server}/resources` | 查看资源使用情况 |

**中间件放行逻辑** ✅ 代码事实：[AuthenticateServerAccess.php 第 49-62 行](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/app/Http/Middleware/Api/Client/Server/AuthenticateServerAccess.php#L49-L62)

```php
try {
    $server->validateCurrentState();
} catch (ServerStateConflictException $exception) {
    // 例外 1：所有异常状态下都可以 view
    if (!$request->routeIs('api:client:server.view')) {
        // 例外 2：只有挂起/节点维护时可以看 resources
        if (($server->isSuspended() || $server->node->isUnderMaintenance()) 
            && !$request->routeIs('api:client:server.resources')) {
            throw $exception;
        }
        // 例外 3：管理员 (root_admin) 可以访问 websocket
        if (!$user->root_admin || !$request->routeIs($this->except)) {
            throw $exception;
        }
    }
}
```

**被拒绝的操作**（全部由中间件拦截，控制器本身无额外挂起检查）✅ 代码事实：
- 电源操作（启动/停止/重启）[PowerController](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/app/Http/Controllers/Api/Client/Servers/PowerController.php)
- 发送控制台命令 [CommandController](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/app/Http/Controllers/Api/Client/Servers/CommandController.php)
- 文件管理（浏览/上传/下载/压缩等）[FileController](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/app/Http/Controllers/Api/Client/Servers/FileController.php)
- 备份管理（列表/创建/下载/恢复/删除）
- 数据库、调度任务、子用户、网络分配、启动参数、设置等

> **例外说明** ✅ 代码事实：管理员（`user.root_admin = true`）在挂起时仍可访问 **Websocket 接口** `api:client:server.ws`。

### 3.3 错误响应 ✅ 代码事实

拒绝访问时抛出 [ServerStateConflictException](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/app/Exceptions/Http/Server/ServerStateConflictException.php)，挂起时的错误消息：

> `"This server is currently suspended and the functionality requested is unavailable."`

HTTP 状态码：`409 Conflict`

---

## 四、各资源的具体限制

### 4.1 网络与运行状态

| 项目 | 限制行为 | 验证类型 | 代码依据 |
|-----|---------|---------|---------|
| Websocket 连接 | 收到关闭码 4409/4400 时前端主动关闭连接 | ✅ 事实 | [Websocket.ts 第 40-47 行](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/resources/scripts/plugins/Websocket.ts#L40-L47) |
| 停止运行容器 | 预期停止 | ⚠️ 推断 | 管理后台描述 + Wings 通用逻辑 |
| 禁用网络连接 | 预期禁用 | ⚠️ 推断 | 管理后台描述 + Wings 通用逻辑 |

**前端 Websocket 处理** ✅ 代码事实：
```typescript
// 代码 4409 为 Wings 主动关闭时的挂起标识
// 代码 4400 为预留扩展码
if (evt.code === 4409 || evt.code === 4400) {
    this.close(1000);
}
```

### 4.2 存储（文件）限制

| 操作 | 挂起后行为 | 验证类型 | 代码依据 |
|-----|-----------|---------|---------|
| 文件管理 API | 中间件拦截 → 409 拒绝 | ✅ 事实 | AuthenticateServerAccess 中间件 |
| SFTP 登录 | Panel 认证阶段拒绝 | ✅ 事实 | [SftpAuthenticationController.php 第 153 行](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/app/Http/Controllers/Api/Remote/SftpAuthenticationController.php#L153) 调用 `validateCurrentState()` |
| 已有服务器文件 | Panel 代码中无删除逻辑 | ✅ 事实 | SuspensionService 中无文件/存储删除操作 |
| Wings 端文件保留 | 预期保留 | ⚠️ 推断 | 管理后台描述中无"删除"措辞 |

### 4.3 备份限制

| 操作 | 挂起后行为 | 验证类型 | 代码依据 |
|-----|-----------|---------|---------|
| 手动创建备份 | 中间件拦截 → 409 拒绝 | ✅ 事实 | AuthenticateServerAccess 中间件 |
| 查看 / 下载 / 删除备份 | 中间件拦截 → 409 拒绝 | ✅ 事实 | 同上 |
| 恢复备份 | 中间件 + 控制器双重拒绝 | ✅ 事实 | [BackupController.php 第 211 行](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/app/Http/Controllers/Api/Client/Servers/BackupController.php#L198-L229) |
| 定时备份任务 | 直接跳过，不执行备份动作 | ✅ 事实 | [RunTaskJob.php 第 53-57 行](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/app/Jobs/Schedule/RunTaskJob.php#L53-L57) |
| 已有备份记录 | Panel 数据库无删除逻辑 | ✅ 事实 | SuspensionService 无备份删除操作 |
| Wings 端备份文件 | 预期保留 | ⚠️ 推断 | Panel 无删除调用 |

**定时任务跳过逻辑** ✅ 代码事实：
```php
// RunTaskJob.php 第 53-57 行
if (!is_null($server->status)) {  // 包含 suspended 在内的任何异常状态
    $this->failed();
    return;
}
```

**定时任务跳过的具体效果** ✅ 代码事实（测试 [RunTaskJobTest.php 第 149-172 行](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/tests/Integration/Jobs/Schedule/RunTaskJobTest.php#L149-L172) 验证）：
- `task.is_queued` → `false`（任务不再标记为排队中）
- `schedule.is_processing` → `false`（调度不再标记为处理中）
- `schedule.last_run_at` → **更新为当前时间**（即使挂起也会更新，见下文详述）

---

## 五、客服判断挂起状态的可用数据源

### 5.1 判断挂起状态的权威来源 ✅ 代码事实

| 数据源 | 字段 | 精度 | 是否推荐 | 说明 |
|-------|------|------|---------|------|
| Panel 数据库 / 服务器基本信息 API | `status` | 精确 | ✅ **推荐** | 状态权威来源，Panel 所有逻辑基于此字段 |
| 资源使用 API | `is_suspended` | 不可靠 | ❌ **不推荐** | 来自 Wings，有缓存和默认值问题 |

**服务器基本信息 API**（`api:client:server.view`）✅ 代码事实：
- 返回 `status` 字段，挂起时为 `'suspended'`
- 数据来自 Panel 数据库，实时准确
- 挂起时可访问（白名单内）

**资源使用 API**（`api:client:server.resources`）⚠️ 注意事项：
- 返回 `is_suspended` 字段，数据**来自 Wings**，不是 Panel 数据库
- 有 **20 秒缓存**（[ResourceUtilizationController.php 第 33 行](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/app/Http/Controllers/Api/Client/Servers/ResourceUtilizationController.php#L33)）
- **默认值为 `false`**（[StatsTransformer.php 第 22 行](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/app/Transformers/Api/Client/StatsTransformer.php#L22)）
- 如果 Wings 连接异常或数据格式不同，可能返回默认值 `false`，造成"未挂起"的假象

> **客服操作建议**：判断服务器是否挂起，**务必使用服务器基本信息接口的 `status` 字段**，不要使用资源接口的 `is_suspended` 字段。

### 5.2 资源接口返回内容细节 ✅ 代码事实

资源接口数据结构（[StatsTransformer.php](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/app/Transformers/Api/Client/StatsTransformer.php)）：

```json
{
  "current_state": "stopped",     // 电源状态，默认 "stopped"
  "is_suspended": false,          // 是否挂起，默认 false（来自 Wings）
  "resources": {
    "memory_bytes": 0,            // 默认 0
    "cpu_absolute": 0,            // 默认 0
    "disk_bytes": 0,              // 默认 0
    "network_rx_bytes": 0,        // 默认 0
    "network_tx_bytes": 0,        // 默认 0
    "uptime": 0                   // 默认 0
  }
}
```

所有字段都有默认值（0 或 false），当 Wings 未返回对应字段时使用默认值。

> **重要** ⚠️ 推断：挂起状态下 Wings 是否还会返回资源使用数据、返回什么数据，Panel 代码无法验证。客服不能假设挂起时资源数据一定是 0。

---

## 六、调度任务与挂起的详细关系 ✅ 代码事实

### 6.1 挂起时调度任务的行为

当服务器处于挂起状态时，定时调度到了会触发，但任务**不会实际执行**（不会发电源指令、不会执行备份等）。

### 6.2 last_run_at 的变化

**会更新**。即使服务器挂起，`schedule.last_run_at` 仍然会被更新为当前时间。

**代码调用链**：
```
handle() 检查 status
  → status 非 null（挂起）
  → 调用 failed()
  → 调用 markScheduleComplete()
  → 更新 last_run_at = 当前时间
```

**关键代码** [RunTaskJob.php 第 120-126 行](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/app/Jobs/Schedule/RunTaskJob.php#L120-L126)：
```php
private function markScheduleComplete()
{
    $this->task->schedule()->update([
        'is_processing' => false,
        'last_run_at' => CarbonImmutable::now()->toDateTimeString(),
    ]);
}
```

测试验证 [RunTaskJobTest.php 第 149-172 行](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/tests/Integration/Jobs/Schedule/RunTaskJobTest.php#L149-L172)：
- 初始 `last_run_at` = 1 小时前
- 挂起状态下执行任务后
- `last_run_at` = 当前时间（已更新）

### 6.3 客服注意事项

- ❌ 不能用 `last_run_at` 是否等于当前时间来判断"任务是否成功执行了"
- ❌ 不能用 `last_run_at` 倒推"服务器什么时候被挂起的"
- ✅ `last_run_at` 只能说明"调度器最近一次尝试运行的时间"，不代表任务实际执行成功
- ✅ 想知道任务是否真正执行了，需要看具体任务的产物（如备份文件、服务器状态变化等）

---

## 七、计费口径说明（客服判断边界）

### 7.1 Panel 代码中可直接判断的信息 ✅ 代码事实

客服可以 100% 确认的：

| 信息项 | 判断方式 | 精度 |
|-------|---------|------|
| **服务器当前是否挂起** | 查询 `servers.status` 字段或调用服务器基本信息 API 读取 `status` 字段 | 精确 |
| **挂起状态变更的最近时间** | 只能读取 `servers.updated_at`（通用更新时间） | 粗粒度 |
| **当前是否还占用 API 白名单资源** | 是（view 和 resources 接口始终可访问） | 精确 |
| **定时任务是否还在执行实际操作** | 否（挂起时任务动作被跳过） | 精确 |
| **调度器是否还在运行** | 是（调度器仍会触发，last_run_at 仍会更新） | 精确 |

> **重要边界** ✅ 代码事实：
> - `servers.updated_at` 会被任何更新操作改变（修改配置、创建子用户等都算），**不能**精确等同于挂起开始时间
> - Panel 代码**没有** `suspended_at` / `unsuspended_at` 字段
> - Panel 代码**没有**记录挂起/解除操作的 Activity Log

### 7.2 不能仅凭 Panel 代码判断的信息 ❌ Panel 不可判断

以下信息 Panel 代码无法直接支持，**客服不能下结论**：

| 信息项 | 原因 | 需依赖 |
|-------|------|--------|
| **挂起的精确开始时间** | 无 `suspended_at` 字段，无操作日志 | 外部系统自行记录 |
| **挂起累计时长** | 无时间字段，无法计算 | 外部系统自行记录 |
| **CPU / 内存是否已释放** | Panel 不追踪容器实时状态 | Wings 资源接口 / 节点监控 |
| **是否停止节点计费** | Panel 无任何计费逻辑 | 外部业务系统决策 |
| **磁盘占用是否应计费** | Panel 只存配额，不判断是否计费 | 外部业务系统决策 |
| **备份存储是否应计费** | Panel 只存备份列表，不判断是否计费 | 外部业务系统决策 |
| **资源使用数据是否为 0** | 挂起时 Wings 返回什么数据，Panel 不确定 | Wings 实际行为 |

### 7.3 计费结论建议

> Panel 仅提供**当前状态标识**（`status = 'suspended'`）和**最近更新时间**（`updated_at`），任何计费相关的判断必须由外部业务系统基于自身的记录和规则作出，不能仅凭 Panel 数据推断。
>
> 如果需要精确计费，建议在外部系统中自行记录每次挂起和解除挂起的时间戳，而不是依赖 Panel 的 `updated_at` 或 `last_run_at`。

---

## 八、解除挂起的恢复路径 ✅ 代码事实

### 8.1 状态变更

解除挂起调用同一个 `SuspensionService::toggle()`，参数为 `ACTION_UNSUSPEND`，数据库状态变化：

```
status = 'suspended'  ──►  status = null
```

### 8.2 恢复后的各项行为

| 项目 | 解除挂起后的行为 | 验证类型 | 说明 |
|-----|----------------|---------|------|
| API 接口恢复 | 全部可访问 | ✅ 事实 | `status = null`，中间件不再拦截 |
| SFTP 恢复 | 认证通过 | ✅ 事实 | `validateCurrentState()` 不再抛异常 |
| 定时任务恢复 | 下次调度正常执行实际动作 | ✅ 事实 | `status = null`，不再跳过 |
| Websocket 恢复 | 正常连接 | ✅ 事实 | 中间件放行 |
| 服务器自动启动 | **不会**自动启动 | ✅ 事实 | Panel 代码无自动启动调用 |
| Wings 端恢复可启动 | 预期可启动 | ⚠️ 推断 | Wings 基于 `suspended:false` 恢复 |

> **客服要点** ✅ 代码事实：解除挂起后，**必须由用户手动启动服务器**。Panel 在解除挂起流程中没有调用任何 `start` 电源操作。

### 8.3 安装完成时的挂起保持 ✅ 代码事实

服务器在**安装期间被挂起**，安装完成后会**保持挂起状态**，不自动解除。见 [ServerInstallController.php 第 71-74 行](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/app/Http/Controllers/Api/Remote/Servers/ServerInstallController.php#L71-L74)：

```php
// Keep the server suspended if it's already suspended
if ($server->status === Server::STATUS_SUSPENDED) {
    $status = Server::STATUS_SUSPENDED;
}
```

### 8.4 回滚路径 ✅ 代码事实

当 Wings sync 请求失败时（HTTP 非 2xx 或超时），Panel 会**回滚数据库状态**到操作前，避免 Panel 与 Wings 状态不一致。见 [SuspensionService.php 第 53-59 行](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/app/Services/Servers/SuspensionService.php#L53-L59)。

---

## 九、前端界面表现 ✅ 代码事实

### 9.1 服务器列表页 [ServerRow.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/resources/scripts/components/dashboard/ServerRow.tsx)
- 显示红色 `Suspended` 标签
- 挂起时**不发起**资源使用数据请求（减少无意义请求）
- 服务器卡片仍可点击进入详情页

### 9.2 服务器详情页 [ConflictStateRenderer.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/resources/scripts/components/server/ConflictStateRenderer.tsx)
- 显示**全屏遮罩**，阻止所有操作
- 标题：`Server Suspended`
- 提示：`This server is suspended and cannot be accessed.`
- 不渲染控制台、文件、备份等任何功能页面

---

## 十、客服问答速查（仅包含代码可验证内容）

**Q: 服务器挂起后数据会丢失吗？**  
A: Panel 代码中没有任何删除服务器文件、备份或数据库的逻辑。根据管理后台官方说明，挂起只是停止进程并阻止访问，不会删除数据。Wings 端的具体数据保留行为由守护进程实现。

**Q: 挂起期间还计费吗？**  
A: Panel 本身不包含计费逻辑，只提供一个 `status = 'suspended'` 状态标识。是否暂停计费由你们的外部业务系统决定。Panel 没有记录挂起开始时间的字段，如果需要精确计费，建议在外部系统中自行记录每次挂起/解除操作的时间。

**Q: 能知道服务器挂了多久吗？**  
A: Panel 数据库中没有专门的挂起时间字段，只有通用的 `updated_at` 最近更新时间。由于任何配置修改都会改变 `updated_at`，它不能精确代表挂起开始时间。如需精确统计，建议在外部系统中自行记录挂起/解除操作的时间。

**Q: 怎么判断服务器是不是挂起状态？**  
A: 看服务器基本信息接口的 `status` 字段，如果是 `'suspended'` 就是挂起状态。**不要用资源接口的 `is_suspended` 字段判断**，因为那个数据来自 Wings，有 20 秒缓存，而且默认值是 `false`，可能不准确。

**Q: 挂起后用户能做什么？**  
A: 用户只能查看服务器基本信息和资源使用情况。包括启动服务器、文件管理、备份管理、控制台、SFTP 在内的所有其他操作都会被拒绝。

**Q: 解除挂起后服务器会自动启动吗？**  
A: 不会。解除挂起只恢复状态，不会自动启动服务器。用户需要手动点击启动。

**Q: 挂起期间定时备份还会执行吗？**  
A: 不会实际执行备份动作。调度器仍然会触发（`last_run_at` 会更新），但发现服务器挂起后会直接跳过，不会创建备份文件，也不会执行其他任务动作。

**Q: 挂起操作有操作日志吗？**  
A: Panel 代码中没有对挂起/解除挂起操作记录 Activity Log，只有管理后台界面会显示一次性的 flash 提示。如需审计，建议在外部业务系统中自行记录。

**Q: 管理员在挂起时能做什么？**  
A: 管理员比普通用户多一项权限——可以连接 Websocket。其他操作（启动、文件管理等）管理员也和普通用户一样被限制。

**Q: 安装中被挂起的服务器，装好后会自动解除挂起吗？**  
A: 不会。代码明确处理了这个场景——安装完成时如果发现服务器本来是挂起状态，就继续保持挂起，不会自动解除。

**Q: 资源接口返回的 is_suspended 能用来判断挂起吗？**  
A: 不推荐。`is_suspended` 来自 Wings，不是 Panel 数据库的权威状态，而且有 20 秒缓存，默认值还是 `false`，可能因为连接异常等原因返回不准确的值。判断挂起请用服务器基本信息接口的 `status` 字段。

**Q: last_run_at 更新了说明备份成功了吗？**  
A: 不能说明。挂起时 `last_run_at` 也会更新，但任务实际被跳过了。要确认备份是否成功，需要看备份列表里有没有新增的备份文件，不能只看 `last_run_at`。
