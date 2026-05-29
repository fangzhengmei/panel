# Pterodactyl Panel 服务器重装与构建配置重放代码脉络

## 一、重装入口（三层入口）

### 1. 客户端 API 入口
**路由**：`routes/api-client.php:150`
```
POST /api/client/servers/{server}/settings/reinstall
```

**控制器**：`app/Http/Controllers/Api/Client/Servers/SettingsController.php:64-71`
```php
public function reinstall(ReinstallServerRequest $request, Server $server): JsonResponse
{
    $this->reinstallServerService->handle($server);
    Activity::event('server:reinstall')->log();
    return new JsonResponse([], Response::HTTP_ACCEPTED);
}
```

**前端调用**：`resources/scripts/api/server/reinstallServer.ts:3-8`
```typescript
http.post(`/api/client/servers/${uuid}/settings/reinstall`)
```

### 2. 应用 API 入口
**路由**：`routes/api-application.php:90`
```
POST /api/application/servers/{server:id}/reinstall
```

**控制器**：`app/Http/Controllers/Api/Application/Servers/ServerManagementController.php:55-60`
```php
public function reinstall(ServerWriteRequest $request, Server $server): Response
{
    $this->reinstallServerService->handle($server);
    return $this->returnNoContent();
}
```

### 3. 管理后台入口
**控制器**：`app/Http/Controllers/Admin/ServersController.php:110-116`
```php
public function reinstallServer(Server $server): RedirectResponse
{
    $this->reinstallService->handle($server);
    $this->alert->success(trans('admin/server.alerts.server_reinstalled'))->flash();
    return redirect()->route('admin.servers.view.manage', $server->id);
}
```

---

## 二、核心重装逻辑：ReinstallServerService

**文件**：`app/Services/Servers/ReinstallServerService.php:25-34`

```php
public function handle(Server $server): Server
{
    return $this->connection->transaction(function () use ($server) {
        // 1. 更新服务器状态为安装中
        $server->fill(['status' => Server::STATUS_INSTALLING])->save();
        
        // 2. 调用 Wings 接口触发重装
        $this->daemonServerRepository->setServer($server)->reinstall();
        
        return $server->refresh();
    });
}
```

**关键点**：
- 数据库事务包裹，确保状态更新和 Wings 调用的一致性
- 状态标记为 `STATUS_INSTALLING`（与首次安装共用状态）
- 同步调用 Wings，无异步队列

---

## 三、Wings 通信层：DaemonServerRepository

**文件**：`app/Repositories/Wings/DaemonServerRepository.php:95-107`

### 重装请求
```php
public function reinstall(): void
{
    Assert::isInstanceOf($this->server, Server::class);
    
    try {
        $this->getHttpClient()->post(sprintf(
            '/api/servers/%s/reinstall',
            $this->server->uuid
        ));
    } catch (TransferException $exception) {
        throw new DaemonConnectionException($exception);
    }
}
```

### HTTP 客户端配置（基类）
**文件**：`app/Repositories/Wings/DaemonRepository.php:49-64`
- 使用 Node 的 `getDecryptedKey()` 作为 Bearer Token
- 连接地址从 Node 模型的 `getConnectionAddress()` 获取

---

## 四、Wings 回调拉取配置流程

Wings 收到重装请求后，会主动回调 Panel 获取最新配置和安装脚本。

### 1. 服务器配置拉取
**路由**：`routes/api-remote.php:14`
```
GET /api/remote/servers/{uuid}
```

**控制器**：`app/Http/Controllers/Api/Remote/Servers/ServerDetailsController.php:38-60`
```php
public function __invoke(Request $request, string $uuid): JsonResponse
{
    // ... 权限验证 ...
    
    return new JsonResponse([
        'settings' => $this->configurationStructureService->handle($server),
        'process_configuration' => $this->eggConfigurationService->handle($server),
    ]);
}
```

### 2. 安装脚本拉取
**路由**：`routes/api-remote.php:15`
```
GET /api/remote/servers/{uuid}/install
```

**控制器**：`app/Http/Controllers/Api/Remote/Servers/ServerInstallController.php:31-45`
```php
public function index(Request $request, string $uuid): JsonResponse
{
    $server = $this->repository->getByUuid($uuid);
    $egg = $server->egg;
    
    // ... 权限验证 ...
    
    return new JsonResponse([
        'container_image' => $egg->copy_script_container,
        'entrypoint' => $egg->copy_script_entry,
        'script' => $egg->copy_script_install,
    ]);
}
```

---

## 五、配置结构生成

### ServerConfigurationStructureService
**文件**：`app/Services/Servers/ServerConfigurationStructureService.php:43-92`

生成发送给 Wings 的完整配置：
```php
return [
    'uuid' => $server->uuid,
    'meta' => [ 'name' => $server->name, 'description' => $server->description ],
    'suspended' => $server->isSuspended(),
    'environment' => $this->environment->handle($server),    // 环境变量
    'invocation' => $server->startup,                         // 启动命令
    'skip_egg_scripts' => $server->skip_scripts,              // 跳过 egg 脚本标志
    'build' => [
        'memory_limit' => $server->memory,
        'swap' => $server->swap,
        'io_weight' => $server->io,
        'cpu_limit' => $server->cpu,
        'threads' => $server->threads,
        'disk_space' => $server->disk,
        'oom_disabled' => $server->oom_disabled,
    ],
    'container' => [
        'image' => $server->image,
        'requires_rebuild' => false,
    ],
    'allocations' => [ /* IP/端口映射 */ ],
    'mounts' => [ /* 挂载目录 */ ],
    'egg' => [
        'id' => $server->egg->uuid,
        'file_denylist' => $server->egg->inherit_file_denylist,  // 文件禁止列表
    ],
];
```

### EnvironmentService
**文件**：`app/Services/Servers/EnvironmentService.php:33-60`

生成环境变量：
1. Egg 变量（`server_variables.server_value` 优先于 `egg_variables.default_value`）
2. 内置变量：`STARTUP`, `P_SERVER_LOCATION`, `P_SERVER_UUID`
3. 配置文件定义的变量
4. 动态注册的变量

### EggConfigurationService
**文件**：`app/Services/Eggs/EggConfigurationService.php:22-34`

生成 Egg 进程配置：
```php
return [
    'startup' => [ 'done' => [...], 'user_interaction' => [], 'strip_ansi' => false ],
    'stop' => [ 'type' => 'command'|'signal', 'value' => '...' ],
    'configs' => [ /* 配置文件替换规则 */ ],
];
```

---

## 六、Egg 脚本继承机制

**文件**：`app/Models/Egg.php:159-266`

通过访问器自动处理脚本继承链：

| 访问器 | 继承逻辑 |
|--------|----------|
| `copy_script_install` | 自身 `script_install` → 父 `scriptFrom.script_install` |
| `copy_script_entry` | 自身 `script_entry` → 父 `scriptFrom.script_entry` |
| `copy_script_container` | 自身 `script_container` → 父 `scriptFrom.script_container` |
| `inherit_file_denylist` | 父 `configFrom.file_denylist` → 自身 `file_denylist` |
| `inherit_config_files` | 自身 `config_files` → 父 `configFrom.config_files` |
| `inherit_config_startup` | 自身 `config_startup` → 父 `configFrom.config_startup` |

**继承关系字段**：
- `copy_script_from`: 脚本继承源 Egg ID
- `config_from`: 配置继承源 Egg ID

---

## 七、数据目录保留与覆盖边界

### 1. 保留边界（Panel 侧）
Panel **不直接**操作服务器数据目录，所有文件操作由 Wings 执行。Panel 仅通过以下字段影响边界：

| 字段 | 位置 | 作用 |
|------|------|------|
| `skip_scripts` | `Server` 模型 | 布尔值，是否跳过 egg 安装脚本执行。`true` 时重装只重建容器，不执行脚本 |
| `file_denylist` | `Egg` 模型 | 数组，定义重装/安装时禁止覆盖的文件路径列表。通过 `inherit_file_denylist` 继承 |

**代码位置**：`ServerConfigurationStructureService.php:87-90`
```php
'egg' => [
    'id' => $server->egg->uuid,
    'file_denylist' => $server->egg->inherit_file_denylist,
],
```

### 2. 覆盖边界（Wings 侧）
Wings 收到 `file_denylist` 后，在执行安装脚本时会保护这些文件不被覆盖。
具体覆盖逻辑在 Wings 代码中实现，Panel 仅负责下发配置。

### 3. 重装时的数据处理
根据前端提示（`ReinstallServerBox.tsx:48-57`）：
- 服务器会被停止
- 重新运行安装脚本
- "Some files may be deleted or modified"
- 具体哪些文件被删除/修改由 egg 安装脚本决定

---

## 八、构建配置重放逻辑

### BuildModificationService
**文件**：`app/Services/Servers/BuildModificationService.php:33-75`

```php
public function handle(Server $server, array $data): Server
{
    $server = $this->connection->transaction(function () use ($server, $data) {
        // 1. 处理分配变更（添加/删除端口）
        $this->processAllocations($server, $data);
        
        // 2. 更新构建参数
        $merge = Arr::only($data, ['oom_disabled', 'memory', 'swap', 'io', 'cpu', 'threads', 'disk', 'allocation_id']);
        $server->forceFill(array_merge($merge, [
            'database_limit' => Arr::get($data, 'database_limit', 0) ?? null,
            'allocation_limit' => Arr::get($data, 'allocation_limit', 0) ?? null,
            'backup_limit' => Arr::get($data, 'backup_limit', 0) ?? 0,
        ]))->saveOrFail();
        
        return $server->refresh();
    });
    
    // 3. 同步到 Wings
    $updateData = $this->structureService->handle($server);
    if (!empty($updateData['build'])) {
        try {
            $this->daemonServerRepository->setServer($server)->sync();
        } catch (DaemonConnectionException $exception) {
            Log::warning($exception, ['server_id' => $server->id]);
        }
    }
    
    return $server;
}
```

### 配置重放机制
1. **实时同步**：修改构建参数后立即调用 `sync()` 方法
2. **失败容忍**：Wings 连接失败仅记录日志，不阻塞数据库更新
3. **最终一致**：Wings 每次启动/重启服务器时，都会主动回调 Panel 拉取最新配置

**同步接口**：`DaemonServerRepository.php:63-72`
```php
public function sync(): void
{
    $this->getHttpClient()->post("/api/servers/{$this->server->uuid}/sync");
}
```

---

## 九、Wings 安装完成回调

**路由**：`routes/api-remote.php:16`
```
POST /api/remote/servers/{uuid}/install
```

**控制器**：`app/Http/Controllers/Api/Remote/Servers/ServerInstallController.php:53-88`

```php
public function store(InstallationDataRequest $request, string $uuid): JsonResponse
{
    // ... 权限验证 ...
    
    // 确定最终状态
    if (!$request->boolean('successful')) {
        $status = $request->boolean('reinstall') 
            ? Server::STATUS_REINSTALL_FAILED 
            : Server::STATUS_INSTALL_FAILED;
    }
    
    // 保持挂起状态
    if ($server->status === Server::STATUS_SUSPENDED) {
        $status = Server::STATUS_SUSPENDED;
    }
    
    // 更新状态和安装时间
    $this->repository->update($server->id, [
        'status' => $status, 
        'installed_at' => CarbonImmutable::now()
    ], true, true);
    
    // 发送邮件通知（可配置）
    $isInitialInstall = is_null($server->installed_at);
    if ($isInitialInstall && config('pterodactyl.email.send_install_notification', true)) {
        $this->eventDispatcher->dispatch(new ServerInstalled($server));
    } elseif (!$isInitialInstall && config('pterodactyl.email.send_reinstall_notification', true)) {
        $this->eventDispatcher->dispatch(new ServerInstalled($server));
    }
    
    return new JsonResponse([], Response::HTTP_NO_CONTENT);
}
```

---

## 十、完整时序图

```
前端/API 调用
    │
    ▼
SettingsController::reinstall()
    │
    ▼
ReinstallServerService::handle()
    ├─ 事务开始
    ├─ Server.status = STATUS_INSTALLING
    └─ DaemonServerRepository::reinstall()
        │
        ▼
Wings: POST /api/servers/{uuid}/reinstall
    │
    ├─ Wings 停止服务器
    │
    ▼
Wings: GET /api/remote/servers/{uuid} (拉取配置)
    │
    ├─ ServerConfigurationStructureService → settings
    └─ EggConfigurationService → process_configuration
    │
    ▼
Wings: GET /api/remote/servers/{uuid}/install (拉取脚本)
    │
    └─ Egg 访问器自动解析继承链
    │
    ├─ Wings 执行安装脚本（受 file_denylist 保护）
    ├─ Wings 启动服务器
    │
    ▼
Wings: POST /api/remote/servers/{uuid}/install (完成回调)
    │
    ├─ 更新 Server.status = null（成功）
    ├─ 更新 Server.installed_at
    └─ 触发 ServerInstalled 事件（发送邮件）
```

---

## 十一、关键状态枚举

**文件**：`app/Models/Server.php:122-126`
```php
public const STATUS_INSTALLING = 'installing';
public const STATUS_INSTALL_FAILED = 'install_failed';
public const STATUS_REINSTALL_FAILED = 'reinstall_failed';
public const STATUS_SUSPENDED = 'suspended';
public const STATUS_RESTORING_BACKUP = 'restoring_backup';
```

**注意**：重装成功后状态为 `null`（表示正常），而非特定的成功状态。

---

## 十二、Wings 重启状态重置

**文件**：`app/Http/Controllers/Api/Remote/Servers/ServerDetailsController.php:89-134`

Wings 重启时会调用 `/api/remote/servers/reset`，Panel 会：
1. 将 `STATUS_RESTORING_BACKUP` 状态的服务器标记为失败
2. 将 `STATUS_INSTALLING` 和 `STATUS_RESTORING_BACKUP` 的服务器重置为正常状态（`null`）

防止 Wings 重启后服务器状态永久卡在中间状态。
