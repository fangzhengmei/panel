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

## 四、Remote 接口鉴权机制

### 4.1 中间件绑定

**文件**：`app/Providers/RouteServiceProvider.php:62-65`
```php
Route::middleware('daemon')
    ->prefix('/api/remote')
    ->scopeBindings()
    ->group(base_path('routes/api-remote.php'));
```

所有 `/api/remote/*` 路由统一挂载 `daemon` 中间件组。

**文件**：`app/Http/Kernel.php:86-89`
```php
'daemon' => [
    SubstituteBindings::class,
    DaemonAuthenticate::class,
],
```

`daemon` 中间件组包含两个中间件：路由模型绑定 + Daemon 身份认证。

### 4.2 DaemonAuthenticate 鉴权核心

**文件**：`app/Http/Middleware/Api/Daemon/DaemonAuthenticate.php:34-66`

```php
public function handle(Request $request, \Closure $next): mixed
{
    // 白名单：daemon.configuration 路由跳过鉴权
    if (in_array($request->route()->getName(), $this->except)) {
        return $next($request);
    }

    // 步骤 1：提取 Bearer Token
    if (is_null($bearer = $request->bearerToken())) {
        throw new HttpException(401, ...);
    }

    // 步骤 2：Token 格式校验 —— 必须为 {token_id}.{token} 两段式
    $parts = explode('.', $bearer);
    if (count($parts) !== 2 || empty($parts[0]) || empty($parts[1])) {
        throw new BadRequestHttpException('The Authorization header provided was not in a valid format.');
    }

    // 步骤 3：查找 Node —— 用 token_id 前缀定位唯一 Node
    $node = $this->repository->findFirstWhere([
        'daemon_token_id' => $parts[0],
    ]);

    // 步骤 4：比对 Token —— 解密数据库中存储的 daemon_token，与请求中的后半段做时间安全比较
    if (hash_equals((string) $this->encrypter->decrypt($node->daemon_token), $parts[1])) {
        $request->attributes->set('node', $node);  // 将 Node 模型注入请求
        return $next($request);
    }

    throw new AccessDeniedHttpException('You are not authorized to access this resource.');
}
```

**鉴权要点**：
| 步骤 | 校验内容 | 失败响应 |
|------|---------|---------|
| 1 | Bearer Token 存在 | 401 + WWW-Authenticate 头 |
| 2 | Token 格式为 `{id}.{secret}` | 400 BadRequest |
| 3 | `daemon_token_id` 匹配到 Node | 403 AccessDenied（不暴露 Node 是否存在） |
| 4 | 解密后 `daemon_token` 与 secret 做 `hash_equals` | 403 AccessDenied |

**安全设计**：
- `daemon_token` 在数据库中加密存储（Laravel Encrypter），泄露数据库无法直接伪造请求
- `hash_equals` 防止时序攻击
- `daemon_token_id` 和 `daemon_token` 均在 Node 模型 `$hidden` 中，不暴露于 API 输出

### 4.3 控制器层二次校验：Node 归属验证

中间件只验证"请求来自某个合法 Node"，控制器内部还需要验证"该 Node 有权操作此服务器"。

**ServerInstallController::index**（`ServerInstallController.php:36-38`）：
```php
if (! $server->node->is($request->attributes->get('node'))) {
    throw new HttpForbiddenException('Requesting node does not have permission to access this server.');
}
```

**ServerInstallController::store**（`ServerInstallController.php:58-60`）：
```php
if (! $server->node->is($request->attributes->get('node'))) {
    throw new HttpForbiddenException('Requesting node does not have permission to access this server.');
}
```

**ServerDetailsController::__invoke**（`ServerDetailsController.php:40-54`）：
```php
// 非转移：仅目标 Node 可访问
$valid = $node->id === $server->node_id;

// 转移中：源/目标 Node 均可访问
$valid = $transfer
    ? $node->id === $transfer->old_node || $node->id === $transfer->new_node
    : $node->id === $server->node_id;
```

**SftpAuthenticationController::getServer**（`SftpAuthenticationController.php:89-97`）：
```php
return Server::query()
    ->where(fn ($builder) => $builder->where('uuid', $uuid)->orWhere('uuidShort', $uuid))
    ->where('node_id', $request->attributes->get('node')->id)  // Node 归属过滤
    ->firstOr(function () use ($request) { $this->reject($request); });
```

**BackupStatusController::index**（`BackupStatusController.php:46-49`）：
```php
$server = $model->server;
if ($server->node_id !== $node->id) {
    throw new HttpForbiddenException('Requesting node does not have permission to access this server.');
}
```

**鉴权链路总结**：
```
请求到达
  │
  ▼
DaemonAuthenticate 中间件
  ├─ 校验 Bearer Token 格式
  ├─ 查找 Node（daemon_token_id）
  ├─ hash_equals 比对解密后的 daemon_token
  └─ 将 Node 注入 $request->attributes['node']
  │
  ▼
控制器方法
  ├─ 通过 UUID 查找 Server
  └─ 验证 Server.node_id === Node.id（或转移中的双 Node 校验）
```

### 4.4 InstallationDataRequest 请求体验证

**文件**：`app/Http/Requests/Api/Remote/InstallationDataRequest.php:1-20`

```php
class InstallationDataRequest extends FormRequest
{
    public function authorize(): bool
    {
        return true;  // 授权已在中间件层完成，此处直接放行
    }

    public function rules(): array
    {
        return [
            'successful' => 'present|boolean',   // 必须提供，布尔值
            'reinstall'  => 'sometimes|boolean',  // 可选，布尔值
        ];
    }
}
```

**关键点**：`reinstall` 字段是 Wings 在安装完成回调中传回的标志，Panel 用它区分首次安装失败 (`install_failed`) 与重装失败 (`reinstall_failed`)。此字段仅 Wings 自行填充，Panel 不校验其真实性——这是一个**信任边界**。

---

## 五、Wings 回调拉取配置流程

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
    Assert::isInstanceOf($node = $request->attributes->get('node'), Node::class);
    $server = $this->repository->getByUuid($uuid);
    $transfer = $server->transfer;

    $valid = $transfer
        ? $node->id === $transfer->old_node || $node->id === $transfer->new_node
        : $node->id === $server->node_id;

    if (! $valid) {
        throw new HttpForbiddenException('Requesting node does not have permission to access this server.');
    }

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

    if (! $server->node->is($request->attributes->get('node'))) {
        throw new HttpForbiddenException('Requesting node does not have permission to access this server.');
    }

    return new JsonResponse([
        'container_image' => $egg->copy_script_container,
        'entrypoint' => $egg->copy_script_entry,
        'script' => $egg->copy_script_install,
    ]);
}
```

### 3. 批量配置拉取（Wings 启动时）
**路由**：`routes/api-remote.php:9`
```
GET /api/remote/servers
```

**控制器**：`app/Http/Controllers/Api/Remote/Servers/ServerDetailsController.php:65-79`
```php
public function list(Request $request): ServerConfigurationCollection
{
    $node = $request->attributes->get('node');

    $servers = Server::query()->with('allocations', 'egg', 'mounts', 'variables', 'location')
        ->where('node_id', $node->id)
        ->paginate((int) $request->input('per_page', 50));

    return new ServerConfigurationCollection($servers);
}
```

**资源类**：`app/Http/Resources/Wings/ServerConfigurationCollection.php:19-31`
```php
public function toArray($request): array
{
    return $this->collection->map(function (Server $server) use ($configuration, $egg) {
        return [
            'uuid' => $server->uuid,
            'settings' => $configuration->handle($server),
            'process_configuration' => $egg->handle($server),
        ];
    })->toArray();
}
```

**注意**：批量接口使用 `where('node_id', $node->id)` 过滤，确保每个 Node 只能获取自己的服务器配置，无需逐个校验归属。

---

## 六、配置结构生成

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

## 七、Egg 脚本继承机制

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

## 八、数据目录保留与覆盖边界

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

## 九、构建配置重放逻辑

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

## 十、Wings 安装完成回调

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

## 十一、完整时序图

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

## 十二、关键状态枚举

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

## 十三、Wings 重启状态重置（resetState 收敛机制）

**文件**：`app/Http/Controllers/Api/Remote/Servers/ServerDetailsController.php:89-134`

**路由**：`routes/api-remote.php:10`
```
POST /api/remote/servers/reset
```

### 13.1 问题背景

Wings 进程重启时，正在执行的操作（安装脚本、备份恢复）会被强制中断，但不会回调 Panel 的完成接口。如果不处理，服务器将永久卡在中间状态。

### 13.2 收敛策略：分两阶段处理

```php
public function resetState(Request $request): JsonResponse
{
    $node = $request->attributes->get('node');

    // ═══ 阶段 1：处理卡在 RESTORING_BACKUP 的服务器 ═══
    $servers = Server::query()
        ->with([
            'activity' => fn ($builder) => $builder
                ->where('activity_logs.event', 'server:backup.restore-started')
                ->latest('timestamp'),
        ])
        ->where('node_id', $node->id)
        ->where('status', Server::STATUS_RESTORING_BACKUP)
        ->get();

    $this->connection->transaction(function () use ($node, $servers) {
        foreach ($servers as $server) {
            $activity = $server->activity->first();
            if (!is_null($activity)) {
                if ($subject = $activity->subjects->where('subject_type', 'backup')->first()) {
                    // 为中断的恢复操作写入失败审计记录
                    Activity::event('server:backup.restore-failed')
                        ->subject($server, $subject->subject)
                        ->property('name', $subject->subject->name)
                        ->log();
                }
            }
        }

        // ═══ 阶段 2：批量重置中间状态 ═══
        Server::query()->where('node_id', $node->id)
            ->whereIn('status', [Server::STATUS_INSTALLING, Server::STATUS_RESTORING_BACKUP])
            ->update(['status' => null]);
    });

    return new JsonResponse([], JsonResponse::HTTP_NO_CONTENT);
}
```

### 13.3 收敛状态转换图

```
Wings 重启前可能的状态          resetState 后
─────────────────────         ──────────────
STATUS_INSTALLING        →    null（正常）
STATUS_RESTORING_BACKUP  →    null（正常）+ 审计日志记录
STATUS_INSTALL_FAILED    →    不变（已是终态，可人工干预）
STATUS_REINSTALL_FAILED  →    不变（已是终态，可人工干预）
STATUS_SUSPENDED         →    不变（挂起状态与 Wings 无关）
null（正常）             →    不变
```

### 13.4 收敛证据链

| 步骤 | 代码位置 | 可验证性 |
|------|---------|---------|
| 1. Wings 启动时主动调用 `/api/remote/servers/reset` | Wings 源码（Panel 侧不可见） | ❌ 不可验证 |
| 2. Panel 查询该 Node 下所有 `INSTALLING` / `RESTORING_BACKUP` 的服务器 | `ServerDetailsController.php:128-130` | ✅ SQL 可追踪 |
| 3. 为 `RESTORING_BACKUP` 服务器写入 `server:backup.restore-failed` 审计日志 | `ServerDetailsController.php:118-122` | ✅ activity_logs 表可查 |
| 4. 批量将中间状态更新为 `null` | `ServerDetailsController.php:128-130` | ✅ 数据库变更可追踪 |
| 5. 仅影响本 Node 的服务器（`where('node_id', $node->id)`） | `ServerDetailsController.php:100,128` | ✅ 隔离性可验证 |

### 13.5 关键设计决策（含代码不一致性说明）

1. **为什么 `INSTALL_FAILED` / `REINSTALL_FAILED` 不被重置？**
   这些状态表示安装确实失败了。但代码中对两种失败状态的处理**完全不对称**（详见第十六章）。

2. **为什么 `RESTORING_BACKUP` 需要写审计日志但 `INSTALLING` 不需要？**
   恢复备份中断后用户需要知道哪个备份失败了（影响数据完整性判断）。安装中断只需重置状态——用户可以重新触发重装。

3. **为什么不在 `resetState` 中自动重试？**
   Wings 重启后状态未知，自动重试可能导致重复操作。正确做法是重置为干净状态，由用户/调度器决定是否重新执行。

---

## 十四、失败状态的真实流转差异：INSTALL_FAILED vs REINSTALL_FAILED

### 14.1 isInstalled() 的不对称判断

**文件**：`app/Models/Server.php:213-216`

```php
public function isInstalled(): bool
{
    return $this->status !== self::STATUS_INSTALLING && $this->status !== self::STATUS_INSTALL_FAILED;
}
```

**⚠️ 关键发现**：此函数**只排除了 `STATUS_INSTALL_FAILED`，但完全遗漏了 `STATUS_REINSTALL_FAILED`！**

| 状态 | isInstalled() 返回值 |
|------|---------------------|
| `null` (正常) | `true` |
| `STATUS_INSTALLING` | `false` |
| `STATUS_INSTALL_FAILED` | `false` |
| `STATUS_REINSTALL_FAILED` | **`true`** ❗ |
| `STATUS_SUSPENDED` | `true` |

### 14.2 validateCurrentState() 的实际效果

**文件**：`app/Models/Server.php:390-401`

```php
public function validateCurrentState()
{
    if (
        $this->isSuspended()
        || $this->node->isUnderMaintenance()
        || !$this->isInstalled()    // ← 间接使用 isInstalled()
        || $this->status === self::STATUS_RESTORING_BACKUP
        || !is_null($this->transfer)
    ) {
        throw new ServerStateConflictException($this);
    }
}
```

**对用户可访问性的影响**：

| 状态 | validateCurrentState() | 用户是否可访问 |
|------|-----------------------|---------------|
| `STATUS_INSTALL_FAILED` | ❌ 抛异常 | ❌ 完全被锁死 |
| `STATUS_REINSTALL_FAILED` | ✅ 通过 | ✅ **正常访问** |

**调用此校验的关键路径**：
- 客户端 API 中间件：`AuthenticateServerAccess.php:50`
- SFTP 认证控制器：`SftpAuthenticationController.php:153`

**结论**：`STATUS_REINSTALL_FAILED` 的服务器用户可以：
- ✅ 正常登录控制台
- ✅ 启动/停止/重启服务器
- ✅ 通过 SFTP 上传下载文件
- ✅ 查看日志和控制台输出

而 `STATUS_INSTALL_FAILED` 的服务器以上操作**全部被禁止**。

### 14.3 toggleInstall() 的不对称处理

**文件**：`app/Http/Controllers/Admin/ServersController.php:88-100`

```php
public function toggleInstall(Server $server): RedirectResponse
{
    if ($server->status === Server::STATUS_INSTALL_FAILED) {
        throw new DisplayException(trans('admin/server.exceptions.marked_as_failed'));
    }

    $this->repository->update($server->id, [
        'status' => $server->isInstalled() ? Server::STATUS_INSTALLING : null,
    ], true, true);

    // ...
}
```

**⚠️ 关键发现**：此函数**只拦截 `STATUS_INSTALL_FAILED`，完全不检查 `STATUS_REINSTALL_FAILED`！**

| 状态 | toggleInstall 行为 |
|------|-------------------|
| `STATUS_INSTALL_FAILED` | ❌ 抛异常："marked_as_failed" |
| `STATUS_REINSTALL_FAILED` | ✅ 执行！`isInstalled()=true` → 设为 `INSTALLING` |
| `null` (正常) | ✅ 设为 `INSTALLING` |
| `STATUS_INSTALLING` | ✅ `isInstalled()=false` → 设为 `null` |

### 14.4 ServerViewController::manage() 的不对称拦截

**文件**：`app/Http/Controllers/Admin/Servers/ServerViewController.php:119-123`

```php
public function manage(Request $request, Server $server): View
{
    if ($server->status === Server::STATUS_INSTALL_FAILED) {
        throw new DisplayException('This server is in a failed install state and cannot be recovered. Please delete and re-create the server.');
    }
    // ...
}
```

**对管理员的影响**：

| 状态 | 管理页面访问 | 信息提示 |
|------|-------------|---------|
| `STATUS_INSTALL_FAILED` | ❌ 完全无法访问 | "cannot be recovered"，建议删除重建 |
| `STATUS_REINSTALL_FAILED` | ✅ 完全可访问 | **无任何提示** ❗ |

### 14.5 两种失败状态的完整行为对比

| 行为维度 | STATUS_INSTALL_FAILED | STATUS_REINSTALL_FAILED |
|---------|----------------------|------------------------|
| **isInstalled()** | `false` | **`true`** ❗ |
| **用户可访问性** | ❌ 完全锁死 | ✅ 正常访问 |
| **管理后台可访问** | ❌ 完全锁死 | ✅ 完全可访问 |
| **toggleInstall 结果** | ❌ 抛异常 | ✅ 转为 INSTALLING |
| **再次触发重装** | ❌ 用户被锁死 | ✅ 用户可正常发起 |
| **resetState 处理** | ➖ 不变 | ➖ 不变 |
| **设计意图** | 首次安装失败=不可恢复 | 重装失败=可重试 |
| **信息提示** | "cannot be recovered" | **无任何提示** ❗ |

### 14.6 设计意图 vs 代码实现的差异

**设计意图推测**：
- `INSTALL_FAILED` → 首次安装失败，可能涉及严重问题（如节点资源不足、镜像拉取失败），标记为不可恢复，建议删除重建
- `REINSTALL_FAILED` → 重装失败，服务器本身是可运行的，只是安装脚本出错，用户应该可以继续使用并重试

**代码实现的问题**：
1. `isInstalled()` 遗漏了 `STATUS_REINSTALL_FAILED` 是有意设计还是 bug？
2. 管理后台对 `STATUS_REINSTALL_FAILED` 没有任何视觉提示，管理员可能意识不到重装失败了
3. 文档中对两种失败状态的区别完全没有说明

**恢复路径对比**：

```
STATUS_INSTALL_FAILED 路径：
  用户被锁死 → 管理员也被锁死 → 只能删除重建
    (无其他出路)

STATUS_REINSTALL_FAILED 路径：
  用户正常使用 → 可随时再次触发重装
       │
       └─ 管理员正常管理 → 可 toggleInstall 或重新安装
```

### 14.7 修正后的完整状态机

```
                    ┌──────────────┐
                    │   null (正常)  │◄──────────────────────────────────┐
                    └──────┬───────┘                                   │
                           │                                           │
              reinstall()  │                                成功回调     │
                           ▼                                 (store)    │
                    ┌──────────────┐         ┌──────────────┐          │
                    │  INSTALLING  │────────►│    null      │──────────┘
                    └──────┬───────┘         └──────────────┘
                           │                        │
              失败回调     │           失败回调      │
              (reinstall=false)      (reinstall=true)
                           ▼                        ▼
                ┌─────────────────┐    ┌────────────────────┐
                │ INSTALL_FAILED  │    │ REINSTALL_FAILED   │
                └────────┬────────┘    └─────────┬──────────┘
                         │                       │
          🔒 用户锁死     │            ✅ 用户正常访问
          🔒 管理锁死     │            ✅ 管理正常访问
         ❌ toggleInstall │           ✅ toggleInstall → INSTALLING
                         │                       │
                         ▼                       ▼
                   [删除重建]          [可再次重装]


                    ┌──────────────┐
                    │  SUSPENDED   │◄──── 挂起操作
                    └──────┬───────┘
                           │ reinstall 成功后保持 SUSPENDED
                           │ (ServerInstallController:72-74)
                           ▼
                    ┌──────────────┐
                    │  SUSPENDED   │  ← 安装完成时检测到已挂起，保持挂起
                    └──────────────┘


         resetState 收敛（仅处理进行中状态）：
         ┌──────────────┐                ┌──────────────┐
         │  INSTALLING  │ ─────────────► │    null      │
         └──────────────┘  Wings 重启时   └──────────────┘
         ┌──────────────────┐            ┌──────────────┐
         │RESTORING_BACKUP  │ ─────────► │  null + 审计  │
         └──────────────────┘             └──────────────┘
```

**状态机修正要点**：
1. `INSTALL_FAILED` 和 `REINSTALL_FAILED` 行为**完全不对称**，不是等价的终态
2. `REINSTALL_FAILED` 不是死胡同——用户可以正常访问并再次触发重装
3. `toggleInstall` 只能从 `REINSTALL_FAILED` 进入 `INSTALLING`，对 `INSTALL_FAILED` 无效
4. `resetState` 对两种失败状态都不处理（已是终态）
5. 从 `REINSTALL_FAILED` 恢复的主要路径是**用户再次触发重装**（而不是 toggleInstall）

---

## 十五、可验证链路与不可验证的 Wings 行为边界

### 15.1 边界总览

Panel 的代码执行在 HTTP 请求/响应的边界内，所有可验证行为都有数据库写入或 HTTP 日志作为证据。而 Wings 的内部行为对 Panel 是黑盒。

```
┌──────────────────────────────────────────────────────────┐
│                     Panel 可验证区域                       │
│                                                          │
│  ┌──────────┐    ┌───────────────┐    ┌───────────────┐  │
│  │ 入口校验  │    │ 数据库写入     │    │ 配置下发      │  │
│  │ · 权限    │    │ · status 变更 │    │ · settings    │  │
│  │ · 请求体  │    │ · installed_at│    │ · environment │  │
│  │ · Node归属│    │ · audit log   │    │ · egg config  │  │
│  └──────────┘    └───────────────┘    └───────────────┘  │
│                                                          │
│  ══════════════ Panel ↔ Wings HTTP 边界 ═══════════════  │
│                                                          │
│  ┌──────────────────────────────────────────────────────┐│
│  │               Wings 不可验证区域                      ││
│  │                                                      ││
│  │  · 何时停止/启动容器                                   ││
│  │  · 安装脚本的具体执行过程                               ││
│  │  · file_denylist 的实际保护效果                        ││
│  │  · 数据目录的实际删除/保留范围                          ││
│  │  · 是否拉取了最新配置（Wings 可能使用缓存）             ││
│  │  · reinstall 回调中 `reinstall` 字段的真实性           ││
│  │  · Wings 重启后是否正确调用 resetState                  ││
│  │                                                      ││
│  └──────────────────────────────────────────────────────┘│
└──────────────────────────────────────────────────────────┘
```

### 15.2 可验证链路清单

每个重装操作的可验证步骤及证据来源：

| # | 操作 | 代码位置 | 证据来源 |
|---|------|---------|---------|
| 1 | 用户/管理员发起重装请求 | `SettingsController:64`, `ServerManagementController:55`, `ServersController:110` | HTTP 访问日志 |
| 2 | 权限校验通过 | `ReinstallServerRequest:10-13` (`settings.reinstall`) | 中间件执行记录 |
| 3 | 服务器状态设为 `INSTALLING` | `ReinstallServerService:28` | `servers.status` 字段 |
| 4 | 向 Wings 发送重装请求 | `DaemonServerRepository:100-103` | Guzzle HTTP 日志 |
| 5 | Wings 回调拉取配置 | `ServerDetailsController:38-60` | HTTP 访问日志 |
| 6 | Wings 回调拉取安装脚本 | `ServerInstallController:31-45` | HTTP 访问日志 |
| 7 | Wings 回报安装结果 | `ServerInstallController:53-88` | HTTP 访问日志 + `InstallationDataRequest` 验证 |
| 8 | 更新服务器状态和安装时间 | `ServerInstallController:76` | `servers.status` + `servers.installed_at` |
| 9 | 触发安装完成事件 | `ServerInstallController:81-85` | `ServerInstalled` 事件 + 邮件发送记录 |
| 10 | 活动日志记录 | `Activity::event('server:reinstall')->log()` | `activity_logs` 表 |

### 15.3 不可验证的 Wings 行为清单

| # | Wings 行为 | Panel 的预期 | 无法验证的原因 |
|---|-----------|-------------|--------------|
| 1 | 停止服务器容器 | 收到 reinstall 后停止 | Panel 不感知容器生命周期 |
| 2 | 回调 Panel 拉取配置 | Wings 主动 GET 配置 | Panel 只能看到请求到来，无法确认 Wings 是否用了缓存 |
| 3 | 执行安装脚本 | 按返回的 script 执行 | Panel 无法观测脚本执行过程 |
| 4 | `file_denylist` 保护文件 | 安装过程中不覆盖列表中的文件 | 保护逻辑在 Wings 侧实现，Panel 无法验证效果 |
| 5 | `skip_scripts` 跳过脚本 | 不执行脚本直接启动 | Panel 无法区分"跳过脚本后启动"和"执行脚本后启动" |
| 6 | 安装完成回调的 `successful` 字段 | 真实反映安装结果 | 由 Wings 自行判断，Panel 只记录不验证 |
| 7 | 安装完成回调的 `reinstall` 字段 | 真实反映是重装还是首次安装 | 由 Wings 自行填充，Panel 直接信任 |
| 8 | Wings 重启后调用 resetState | 自动收敛中间状态 | Panel 只能被动等待，无法主动探测 Wings 重启 |
| 9 | 数据目录清理范围 | 安装脚本决定 | Panel 无法审计文件系统变更 |
| 10 | 安装脚本的 Docker 容器选择 | 使用 `copy_script_container` 返回的镜像 | Panel 只下发配置，无法验证实际使用的容器 |

### 15.4 信任边界分析

Panel 对 Wings 的信任是**全量信任**模型：

```
Panel 信任 Wings 的行为：
  ✓ Wings 收到 reinstall 后会正确执行安装流程
  ✓ Wings 回调时使用的 `successful` 字段是真实的
  ✓ Wings 回调时使用的 `reinstall` 字段是真实的
  ✓ Wings 会遵守 `file_denylist` 不覆盖受保护文件
  ✓ Wings 重启后会调用 resetState
  ✓ Wings 拉取配置后不会使用过期缓存

Panel 不信任的输入：
  ✗ 用户 API 调用（需要权限校验）
  ✗ Remote 接口的调用者身份（需要 DaemonAuthenticate）
  ✗ Remote 接口的 Node 归属（需要二次校验）
```

**安全影响**：
- 如果 Wings 被攻陷或行为异常，Panel 无法检测
- `reinstall` 字段被 Wings 篡改可导致状态分类错误（`install_failed` vs `reinstall_failed`）
- 唯一的状态收敛保障是 `resetState`——但前提是 Wings 重启后真的会调用它

### 15.5 Panel → Wings 方向的通信可靠性

| 调用方式 | 代码位置 | 失败处理 | 可靠性级别 |
|---------|---------|---------|-----------|
| `reinstall()` | `DaemonServerRepository:95-107` | 抛出 `DaemonConnectionException`，事务回滚 | 强一致 |
| `sync()` (构建变更) | `DaemonServerRepository:63-72` | `BuildModificationService` 中 catch 后仅记日志 | 最终一致 |
| `create()` (首次创建) | `DaemonServerRepository:42-56` | 抛出 `DaemonConnectionException` | 强一致 |
| `delete()` | `DaemonServerRepository:79-88` | 抛出 `DaemonConnectionException` | 强一致 |

**关键差异**：重装和创建操作要求 Wings 必须可达（同步失败则回滚），而构建配置同步允许失败（数据库已更新，Wings 下次启动时拉取最新配置）。
