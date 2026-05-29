# Wings 节点接入与 Panel 端持久凭据签发协作分析

## 1. 架构概览

Wings（原 Daemon）与 Panel 的认证体系采用**双轨制认证**：
- **持久凭据**：用于节点与 Panel 之间的 API 通信（Wings → Panel）
- **临时 JWT**：用于用户操作鉴权（Panel → Wings）

---

## 2. Deployment Token 生成机制

### 2.1 节点创建时的持久凭据生成

**核心代码**：`app/Services/Nodes/NodeCreationService.php:25-32`

```php
public function handle(array $data): Node
{
    $data['uuid'] = Uuid::uuid4()->toString();
    $data['daemon_token'] = app(Encrypter::class)->encrypt(Str::random(Node::DAEMON_TOKEN_LENGTH));
    $data['daemon_token_id'] = Str::random(Node::DAEMON_TOKEN_ID_LENGTH);
    return $this->repository->create($data, true, true);
}
```

**凭据构成**：
- `daemon_token_id`：16位随机字符串，明文存储，用于快速查找节点
- `daemon_token`：64位随机字符串，加密存储，实际的认证凭据

**常量定义**：`app/Models/Node.php:59-60`
```php
public const DAEMON_TOKEN_ID_LENGTH = 16;
public const DAEMON_TOKEN_LENGTH = 64;
```

### 2.2 自动部署 Token 生成

**核心代码**：`app/Http/Controllers/Admin/NodeAutoDeployController.php:30-52`

```php
public function __invoke(Request $request, Node $node): JsonResponse
{
    $key = ApiKey::query()
        ->where('user_id', $request->user()->id)
        ->where('key_type', ApiKey::TYPE_APPLICATION)
        ->where('r_nodes', 1)
        ->first();

    if (!$key) {
        $key = $this->keyCreationService->setKeyType(ApiKey::TYPE_APPLICATION)->handle([
            'user_id' => $request->user()->id,
            'memo' => 'Automatically generated node deployment key.',
            'allowed_ips' => [],
        ], ['r_nodes' => 1]);
    }

    return new JsonResponse([
        'node' => $node->id,
        'token' => $key->identifier . $this->encrypter->decrypt($key->token),
    ]);
}
```

**Deployment Token 权限范围**：
- 类型：`ApiKey::TYPE_APPLICATION`（Application API Key）
- 权限：`['r_nodes' => 1]` — 仅节点读取权限
- 权限级别：`AdminAcl::READ = 1`（二进制位运算检查）
- 其他资源权限：默认 0（无权限）

**权限系统说明**（`app/Services/Acl/Api/AdminAcl.php:19-22`）：
```php
public const NONE = 0;   // 无权限
public const READ = 1;   // 读取权限
public const WRITE = 2;  // 写入权限
```

权限检查采用位运算：`$permission & $action`，因此：
- `r_nodes = 1` → `1 & 1 = 1` → 允许读取
- `r_nodes = 1` → `1 & 2 = 0` → 不允许写入
- `r_nodes = 3` → 读取 + 写入权限

**其他可用资源权限**（`app/Models/ApiKey.php:26-34`）：
| 字段 | 资源 | 说明 |
|------|------|------|
| `r_servers` | 服务器 | 服务器管理权限 |
| `r_nodes` | 节点 | 节点管理权限 |
| `r_allocations` | 分配 | IP/端口分配权限 |
| `r_users` | 用户 | 用户管理权限 |
| `r_locations` | 位置 | 数据中心位置权限 |
| `r_nests` | 嵌套 | Egg 分组权限 |
| `r_eggs` | Egg | 游戏服务端配置权限 |
| `r_database_hosts` | 数据库主机 | 数据库服务器权限 |
| `r_server_databases` | 服务器数据库 | 游戏数据库权限 |

### 2.3 自动配置命令与配置写入关联

**自动配置命令**（`resources/views/admin/nodes/view/configuration.blade.php:76`）：
```bash
cd /etc/pterodactyl && sudo wings configure --panel-url {{ config('app.url') }} --token {{ deployment_token }} --node {{ node_id }}
```

**命令参数说明**：
| 参数 | 说明 |
|------|------|
| `--panel-url` | Panel 访问地址 |
| `--token` | Deployment Token（Application API Key） |
| `--node` | 节点 ID |
| `--allow-insecure` | 调试模式下忽略 HTTPS 证书验证 |

**自动配置流程（Wings 侧）**：
1. Wings 使用 `--token` 作为 Application API Key，请求 Panel 的节点配置接口
2. Panel 通过 Application API 认证中间件验证：
   - `AuthenticateApplicationUser.php:18`：验证用户是否为 root_admin
   - `ApplicationApiRequest.php:49`：检查 `r_nodes` 权限 >= READ
3. Panel 返回包含解密后持久凭据的配置
4. Wings 将配置写入 `/etc/pterodactyl/config.yml`

### 2.4 节点配置获取接口

> **⚠️ 重要修正**：节点配置获取**不是**通过 Remote API 进行的，也不存在所谓的"daemon.configuration"例外路由。

节点配置有两个获取入口：

**1. 管理后台（需要管理员登录）**：
- 路由：`admin.nodes.view.configuration` (`routes/admin.php:155`)
- 控制器：`app/Http/Controllers/Admin/Nodes/NodeViewController.php:60-63`

**2. Application API（需要 Application API Key）**：
- 路由：`/api/application/nodes/{node}/configuration` (`routes/api-application.php:38`)
- 控制器：`app/Http/Controllers/Api/Application/Nodes/NodeConfigurationController.php:17-20`

```php
public function __invoke(GetNodeRequest $request, Node $node): JsonResponse
{
    return new JsonResponse($node->getConfiguration());
}
```

**接口权限要求**：
- Request 类：`GetNodeRequest` extends `GetNodesRequest`
- 资源：`AdminAcl::RESOURCE_NODES`
- 权限：`AdminAcl::READ`（即 `r_nodes >= 1`）

**配置输出（含解密后的完整凭据）**：`app/Models/Node.php:142-168`

```php
public function getConfiguration(): array
{
    return [
        'debug' => false,
        'uuid' => $this->uuid,
        'token_id' => $this->daemon_token_id,
        'token' => Container::getInstance()->make(Encrypter::class)->decrypt($this->daemon_token),
        'api' => [/* ... */],
        'system' => [/* ... */],
        'allowed_mounts' => $this->mounts->pluck('source')->toArray(),
        'remote' => route('index'),
    ];
}
```

**首次接入完整流程**：
```
┌─────────┐          ┌─────────┐          ┌─────────┐
│  Admin  │          │  Panel  │          │  Wings  │
└────┬────┘          └────┬────┘          └────┬────┘
     │  1. 创建节点        │                       │
     │────────────────────>│                       │
     │                     │ 生成 daemon_token_id  │
     │                     │ 生成 daemon_token     │
     │  2. 点击"自动部署"  │                       │
     │────────────────────>│                       │
     │                     │ 生成/复用 API Key     │
     │                     │ r_nodes = 1           │
     │  返回 deployment_token │                   │
     │<────────────────────│                       │
     │                                             │
     │  3. 运行自动配置命令                         │
     │  =========================================>│
     │                                             │
     │                                             │ 4. 请求节点配置
     │                                             │ GET /api/application/nodes/{id}/configuration
     │                                             │ Authorization: Bearer <deployment_token>
     │                                             │───────────────────────────────────────────────>│
     │                                             │                                               │
     │                                             │ 验证 Application API Key                      │
     │                                             │ - 检查 root_admin                             │
     │                                             │ - 检查 r_nodes >= 1                           │
     │                                             │ 返回配置（含 daemon_token）                   │
     │                                             │<───────────────────────────────────────────────│
     │                                             │
     │                                             │ 5. 写入 config.yml
     │                                             │ token_id: <daemon_token_id>
     │                                             │ token: <daemon_token>
     │                                             │
     │                                             │ 6. 启动 Wings
     │                                             │ 使用持久凭据通信
```

---

## 3. Wings 换取持久凭据认证流程

### 3.1 DaemonAuthenticate 中间件

**核心代码**：`app/Http/Middleware/Api/Daemon/DaemonAuthenticate.php:34-66`

```php
public function handle(Request $request, \Closure $next): mixed
{
    if (in_array($request->route()->getName(), $this->except)) {
        return $next($request);
    }

    if (is_null($bearer = $request->bearerToken())) {
        throw new HttpException(401, 'Access to this endpoint must include an Authorization header.', null, ['WWW-Authenticate' => 'Bearer']);
    }

    $parts = explode('.', $bearer);
    if (count($parts) !== 2 || empty($parts[0]) || empty($parts[1])) {
        throw new BadRequestHttpException('The Authorization header provided was not in a valid format.');
    }

    try {
        $node = $this->repository->findFirstWhere([
            'daemon_token_id' => $parts[0],
        ]);

        if (hash_equals((string) $this->encrypter->decrypt($node->daemon_token), $parts[1])) {
            $request->attributes->set('node', $node);
            return $next($request);
        }
    } catch (RecordNotFoundException $exception) {
        // Do nothing, avoid exposing node existence
    }

    throw new AccessDeniedHttpException('You are not authorized to access this resource.');
}
```

**认证格式**：
```
Authorization: Bearer <daemon_token_id>.<decrypted_daemon_token>
```

> **⚠️ 关键发现**：中间件中的例外路由 `daemon.configuration` **实际上永远不会匹配**！
>
> 原因：`routes/api-remote.php` 中的所有路由都**没有设置路由名称**（没有 `->name()` 调用），所以 `$request->route()->getName()` 返回 `null`，`$this->except` 中的例外路由永远不会被命中。
>
> 结论：**所有 `/api/remote/*` 路由都需要认证，没有例外！**

**例外路由列表**（`DaemonAuthenticate.php:25-27`）：
```php
protected array $except = [
    'daemon.configuration',
];
```

**证据**：检查 `routes/api-remote.php` 中所有路由，确认没有 `->name()` 调用。

### 3.2 路由组配置

**核心代码**：`app/Providers/RouteServiceProvider.php:62-65`

```php
Route::middleware('daemon')
    ->prefix('/api/remote')
    ->scopeBindings()
    ->group(base_path('routes/api-remote.php'));
```

**中间件组**：`app/Http/Kernel.php:86-89`

```php
'daemon' => [
    SubstituteBindings::class,
    DaemonAuthenticate::class,
],
```

### 3.3 Remote API 真实端点集合

> **⚠️ 修正**：之前列出的路由名称都是错误的，实际上 `api-remote.php` 中的路由都没有设置名称。以下是真实的端点：

**路由文件**：`routes/api-remote.php`

| 端点 | 方法 | 控制器 | 用途 |
|------|------|--------|------|
| `/api/remote/sftp/auth` | POST | `SftpAuthenticationController::__invoke` | SFTP 认证 |
| `/api/remote/servers` | GET | `ServerDetailsController::list` | 获取节点服务器列表 |
| `/api/remote/servers/reset` | POST | `ServerDetailsController::resetState` | 重置服务器状态 |
| `/api/remote/activity` | POST | `ActivityProcessingController::__invoke` | 活动日志上报 |
| `/api/remote/servers/{uuid}` | GET | `ServerDetailsController::__invoke` | 获取单个服务器配置 |
| `/api/remote/servers/{uuid}/install` | GET | `ServerInstallController::index` | 获取服务器安装配置 |
| `/api/remote/servers/{uuid}/install` | POST | `ServerInstallController::store` | 上报服务器安装状态 |
| `/api/remote/servers/{uuid}/transfer/failure` | POST | `ServerTransferController::failure` | 服务器迁移失败 |
| `/api/remote/servers/{uuid}/transfer/success` | POST | `ServerTransferController::success` | 服务器迁移成功 |
| `/api/remote/backups/{backup}` | GET | `BackupRemoteUploadController::__invoke` | 备份下载重定向 |
| `/api/remote/backups/{backup}` | POST | `BackupStatusController::index` | 备份完成上报 |
| `/api/remote/backups/{backup}/restore` | POST | `BackupStatusController::restore` | 备份恢复完成上报 |
| `/api/remote/eggs/install` | POST | `EggInstallController::__invoke` | Egg 安装脚本获取 |

**控制器文件清单**（`app/Http/Controllers/Api/Remote/`）：
- `SftpAuthenticationController.php`
- `ServerDetailsController.php`
- `ActivityProcessingController.php`
- `ServerInstallController.php`
- `ServerTransferController.php`
- `BackupRemoteUploadController.php`
- `BackupStatusController.php`
- `EggInstallController.php`

### 3.4 凭据重置

**核心代码**：`app/Services/Nodes/NodeUpdateService.php:33-38`

```php
public function handle(Node $node, array $data, bool $resetToken = false): Node
{
    if ($resetToken) {
        $data['daemon_token'] = $this->encrypter->encrypt(Str::random(Node::DAEMON_TOKEN_LENGTH));
        $data['daemon_token_id'] = Str::random(Node::DAEMON_TOKEN_ID_LENGTH);
    }
    // ...
}
```

---

## 4. 心跳与版本兼容性校验

### 4.1 前端心跳（节点列表页）

**核心代码**：`resources/views/admin/nodes/index.blade.php:53,79-105`

```html
<td data-action="ping" 
    data-secret="{{ $node->getDecryptedKey() }}" 
    data-location="{{ $node->scheme }}://{{ $node->fqdn }}:{{ $node->daemonListen }}/api/system">
```

```javascript
(function pingNodes() {
    $('td[data-action="ping"]').each(function(i, element) {
        $.ajax({
            type: 'GET',
            url: $(element).data('location'),
            headers: {
                'Authorization': 'Bearer ' + $(element).data('secret'),
            },
            timeout: 5000
        }).done(function (data) {
            $(element).find('i').tooltip({ title: 'v' + data.version });
            $(element).removeClass('text-muted').find('i').removeClass().addClass('fa fa-fw fa-heartbeat faa-pulse animated').css('color', '#50af51');
        }).fail(function (error) {
            $(element).removeClass('text-muted').find('i').removeClass().addClass('fa fa-fw fa-heart-o').css('color', '#d9534f');
        });
    }).promise().done(function () {
        setTimeout(pingNodes, 10000);
    });
})();
```

**心跳频率**：每 10 秒
**目标端点**：Wings `/api/system`（注意：这是 Panel 前端直接访问 Wings，不经过 Panel 后端）
**认证方式**：直接使用解密后的 `daemon_token` 作为 Bearer Token
**版本展示**：仅在 tooltip 中显示 `v{version}`，不进行版本比较

### 4.2 后端系统信息获取（节点详情页）

**核心代码**：`app/Http/Controllers/Admin/Nodes/SystemInformationController.php:26-39`

```php
public function __invoke(Request $request, Node $node): JsonResponse
{
    $data = $this->repository->setNode($node)->getSystemInformation();

    return new JsonResponse([
        'version' => $data['version'] ?? '',
        'system' => [
            'type' => Str::title($data['os'] ?? 'Unknown'),
            'arch' => $data['architecture'] ?? '--',
            'release' => $data['kernel_version'] ?? '--',
            'cpus' => $data['cpu_count'] ?? 0,
        ],
    ]);
}
```

**Wings 调用**：`app/Repositories/Wings/DaemonConfigurationRepository.php:21-30`

```php
public function getSystemInformation(?int $version = null): array
{
    try {
        $response = $this->getHttpClient()->get('/api/system' . (!is_null($version) ? '?v=' . $version : ''));
    } catch (TransferException $exception) {
        throw new DaemonConnectionException($exception);
    }
    return json_decode($response->getBody()->__toString(), true);
}
```

**前端刷新**：`resources/views/admin/nodes/view/index.blade.php:154-168`

```javascript
(function getInformation() {
    $.ajax({
        method: 'GET',
        url: '/admin/nodes/view/{{ $node->id }}/system-information',
        timeout: 5000,
    }).done(function (data) {
        $('[data-attr="info-version"]').html(escapeHtml(data.version));
        $('[data-attr="info-system"]').html(...);
        $('[data-attr="info-cpus"]').html(data.system.cpus);
    }).always(function() {
        setTimeout(getInformation, 10000);
    });
})();
```

### 4.3 版本比较方法定义

**核心代码**：`app/Services/Helpers/SoftwareVersionService.php:74-81`

```php
public function isLatestDaemon(string $version): bool
{
    if ($version === 'develop') {
        return true;
    }
    return version_compare($version, $this->getDaemon()) >= 0;
}
```

**版本获取方法**：
```php
public function getPanel(): string  // 获取 Panel 最新版本
public function getDaemon(): string // 获取 Wings 最新版本
```

**版本比较逻辑**：
- 若当前版本为 `develop`，直接返回 `true`（视为最新）
- 使用 `version_compare($current, $latest) >= 0` 进行比较
- 返回 `true` 表示当前版本 >= 最新版本（已是最新）
- 返回 `false` 表示当前版本 < 最新版本（需要更新）

### 4.4 isLatestDaemon 实际调用情况

> **⚠️ 关键发现**：`isLatestDaemon()` 方法在代码库中**没有被任何地方调用**！

**实际调用搜索结果**：
- `isLatestDaemon` 仅在 `SoftwareVersionService.php:74` 中定义
- 在整个代码库中（`.php` 文件）没有找到任何调用该方法的代码
- CLI 命令 `InfoCommand.php:31-32` 只调用了 `isLatestPanel()`，**没有调用 `isLatestDaemon()`**

**getDaemon() 调用情况**：

| 文件位置 | 用途 | 是否调用 isLatestDaemon |
|---------|------|------------------------|
| `resources/views/admin/nodes/view/index.blade.php:42` | 显示最新版本号 | ❌ 只调用 `getDaemon()` |
| `app/Services/Telemetry/TelemetryCollectionService.php` | 遥测数据收集 | ❌ 直接比较版本 |

**节点详情页版本展示**：`resources/views/admin/nodes/view/index.blade.php:42`

```html
<td>Daemon Version</td>
<td>
    <code data-attr="info-version"><i class="fa fa-refresh fa-fw fa-spin"></i></code>
    (Latest: <code>{{ $version->getDaemon() }}</code>)
</td>
```

### 4.5 版本比较结果的消费路径

**版本数据完整流程**：

```
Wings 端:
  /api/system → 返回 { version: "1.7.0", ... }

Panel 后端:
  SystemInformationController.php → 从 Wings 获取 version
  → 返回 JSON: { "version": "1.7.0", "system": {...} }

Panel 前端:
  AJAX 请求 /admin/nodes/view/{id}/system-information
  → 成功回调: $('[data-attr="info-version"]').html(data.version)
  → 直接替换 HTML，不进行任何版本比较

页面展示:
  显示: 当前版本 <code data-attr="info-version">1.7.0</code>
        (Latest: <code>1.7.2</code>)  ← 由 {{ $version->getDaemon() }} 渲染

注意: 没有版本比较，没有根据比较结果改变显示样式
```

**版本比较缺失的影响**：
- 用户需要手动对比两个版本号来判断是否需要更新
- 没有自动的"需要更新"提示或警告样式
- `isLatestDaemon()` 方法实际上是死代码，定义但未使用

### 4.6 版本获取机制

**核心代码**：`app/Services/Helpers/SoftwareVersionService.php:38-41`

```php
public function getDaemon(): string
{
    return Arr::get(self::$result, 'wings') ?? 'error';
}
```

**缓存机制**：缓存 60 分钟，从 CDN 获取最新版本信息

**遥测服务使用 v2 版本**：`app/Services/Telemetry/TelemetryCollectionService.php:61`

```php
$info = $this->daemonConfigurationRepository->setNode($node)->getSystemInformation(2);
```

---

## 5. NodeJWTService 临时令牌服务

### 5.1 JWT 生成核心逻辑

**核心代码**：`app/Services/Nodes/NodeJWTService.php:63-102`

```php
public function handle(Node $node, ?string $identifiedBy, string $algo = 'md5'): UnencryptedToken
{
    $identifier = hash($algo, $identifiedBy);
    $config = Configuration::forSymmetricSigner(new Sha256(), InMemory::plainText($node->getDecryptedKey()));

    $builder = $config->builder(new TimestampDates())
        ->issuedBy(config('app.url'))
        ->permittedFor($node->getConnectionAddress())
        ->identifiedBy($identifier)
        ->withHeader('jti', $identifier)
        ->issuedAt(CarbonImmutable::now())
        ->canOnlyBeUsedAfter(CarbonImmutable::now()->subMinutes(5));

    if (isset($this->expiresAt)) {
        $builder = $builder->expiresAt($this->expiresAt);
    }

    if (!empty($this->subject)) {
        $builder = $builder->relatedTo($this->subject)->withHeader('sub', $this->subject);
    }

    foreach ($this->claims as $key => $value) {
        $builder = $builder->withClaim($key, $value);
    }

    if (!is_null($this->user)) {
        $builder = $builder
            ->withClaim('user_uuid', $this->user->uuid)
            // The "user_id" claim is deprecated and should not be referenced — it remains
            // here solely to ensure older versions of Wings are unaffected when the Panel
            // is updated.
            //
            // This claim will be removed in Panel@1.11 or later.
            ->withClaim('user_id', $this->user->id);
    }

    return $builder
        ->withClaim('unique_id', Str::random())
        ->getToken($config->signer(), $config->signingKey());
}
```

**JWT 结构**：
- **签名密钥**：节点的 `daemon_token`（解密后）
- **iss**：Panel URL
- **aud**：节点连接地址
- **jti**：基于标识的哈希值
- **iat**：签发时间
- **nbf**：签发前 5 分钟（允许时钟偏差）
- **exp**：过期时间（可选）
- **user_uuid**：用户 UUID
- **user_id**：用户 ID（已弃用，兼容旧版 Wings）
- **unique_id**：随机字符串

### 5.2 Websocket 认证场景

**核心代码**：`app/Http/Controllers/Api/Client/Servers/WebsocketController.php:33-72`

```php
public function __invoke(ClientApiRequest $request, Server $server): JsonResponse
{
    $user = $request->user();
    if ($user->cannot(Permission::ACTION_WEBSOCKET_CONNECT, $server)) {
        throw new HttpForbiddenException('You do not have permission to connect to this server\'s websocket.');
    }

    $permissions = $this->permissionsService->handle($server, $user);

    $node = $server->node;
    if (!is_null($server->transfer)) {
        // Check if the user has permissions to receive transfer logs.
        if (!in_array('admin.websocket.transfer', $permissions)) {
            throw new HttpForbiddenException('You do not have permission to view server transfer logs.');
        }

        // Redirect the websocket request to the new node if the server has been archived.
        if ($server->transfer->archived) {
            $node = $server->transfer->newNode;
        }
    }

    $token = $this->jwtService
        ->setExpiresAt(CarbonImmutable::now()->addMinutes(10))
        ->setUser($request->user())
        ->setClaims([
            'server_uuid' => $server->uuid,
            'permissions' => $permissions,
        ])
        ->handle($node, $user->id . $server->uuid);

    $socket = str_replace(['https://', 'http://'], ['wss://', 'ws://'], $node->getConnectionAddress());

    return new JsonResponse([
        'data' => [
            'token' => $token->toString(),
            'socket' => $socket . sprintf('/api/servers/%s/ws', $server->uuid),
        ],
    ]);
}
```

**JWT 有效期**：10 分钟
**包含信息**：用户信息、服务器 UUID、权限列表

### 5.3 其他 JWT 使用场景

| 场景 | 代码位置 | 用途 |
|------|----------|------|
| 文件下载 | `app/Http/Controllers/Api/Client/Servers/FileController.php` | 生成文件下载签名 |
| 文件上传 | `app/Http/Controllers/Api/Client/Servers/FileUploadController.php` | 生成上传签名 |
| 备份下载 | `app/Services/Backups/DownloadLinkService.php` | 备份下载链接 |
| 服务器迁移 | `app/Http/Controllers/Admin/Servers/ServerTransferController.php` | 迁移认证 |

---

## 6. 版本兼容性设计

### 6.1 向后兼容设计

**NodeJWTService.php:91-96**

```php
// The "user_id" claim is deprecated and should not be referenced — it remains
// here solely to ensure older versions of Wings are unaffected when the Panel
// is updated.
//
// This claim will be removed in Panel@1.11 or later.
->withClaim('user_id', $this->user->id);
```

### 6.2 系统信息版本化

**DaemonConfigurationRepository.php:21-24**

```php
public function getSystemInformation(?int $version = null): array
{
    $response = $this->getHttpClient()->get('/api/system' . (!is_null($version) ? '?v=' . $version : ''));
}
```

**遥测服务使用 v2 版本**：`app/Services/Telemetry/TelemetryCollectionService.php:61`

```php
$info = $this->daemonConfigurationRepository->setNode($node)->getSystemInformation(2);
```

---

## 7. 安全设计要点

### 7.1 凭据安全

1. **加密存储**：`daemon_token` 使用 Laravel Encrypter 加密存储
2. **隐藏字段**：`app/Models/Node.php:70`
   ```php
   protected $hidden = ['daemon_token_id', 'daemon_token'];
   ```
3. **定时哈希比较**：使用 `hash_equals()` 防止时序攻击
4. **不暴露节点存在**：认证失败时统一返回 403，不区分 token_id 是否存在

### 7.2 认证流程安全

1. **Bearer Token 格式**：`<token_id>.<token>` 分离设计
2. **JWT 签名**：使用节点密钥签名，Wings 可独立验证
3. **nbf 偏移**：允许 5 分钟时钟偏差
4. **临时令牌**：JWT 有效期短（10分钟），降低泄露风险

### 7.3 权限隔离

1. **Deployment Token**：仅授予节点读取权限（`r_nodes = 1`）
2. **Remote API**：仅节点可访问，通过 `DaemonAuthenticate` 中间件保护
3. **用户操作**：通过 JWT 中的 permissions 声明进行细粒度权限控制
4. **Application API**：双重检查（root_admin + ACL 权限位）

---

## 8. 完整认证时序图

```
Wings 自动配置流程:
┌─────────┐          ┌─────────┐          ┌─────────┐
│  Admin  │          │  Panel  │          │  Wings  │
└────┬────┘          └────┬────┘          └────┬────┘
     │  1. 创建节点        │                       │
     │────────────────────>│                       │
     │                     │ 生成 daemon_token_id  │
     │                     │ 生成 daemon_token     │
     │  2. 点击"自动部署"  │                       │
     │────────────────────>│                       │
     │                     │ 生成 API Key(r_nodes=1) │
     │  返回 deployment_token │                   │
     │<────────────────────│                       │
     │                                             │
     │  3. 运行自动配置命令                         │
     │  =========================================>│
     │                                             │
     │                                             │ 4. 请求节点配置
     │                                             │ GET /api/application/nodes/{id}/configuration
     │                                             │ Authorization: Bearer ptla_xxxxxxxxx
     │                                             │───────────────────────────────────────────────>│
     │                                             │                                               │
     │                                             │ 认证流程:                                     │
     │                                             │ 1. AuthenticateApplicationUser:               │
     │                                             │    检查 user.root_admin = true                │
     │                                             │ 2. GetNodeRequest.authorize():                │
     │                                             │    AdminAcl::check(key, RESOURCE_NODES, READ) │
     │                                             │    r_nodes & READ = 1 & 1 = 1 → 通过          │
     │                                             │ 返回配置（含 daemon_token）                   │
     │                                             │<───────────────────────────────────────────────│
     │                                             │
     │                                             │ 5. 写入 /etc/pterodactyl/config.yml
     │                                             │    token_id: <daemon_token_id>
     │                                             │    token: <daemon_token>
     │                                             │
     │                                             │ 6. 启动 Wings
     │                                             │    使用持久凭据通信
     │                                             │
     │
Wings 日常通信 (Wings → Panel):
┌─────────┐          ┌─────────┐
│  Wings  │          │  Panel  │
└────┬────┘          └────┬────┘
     │  Bearer <id>.<token> │
     │────────────────────>│ POST /api/remote/activity
     │                     │ DaemonAuthenticate 验证
     │                     │ 1. 拆分 token
     │                     │ 2. 根据 id 查节点
     │                     │ 3. 解密 token 比对 (hash_equals)
     │  返回数据             │
     │<────────────────────│
     │
     │
前端心跳 (Browser → Wings):
┌──────────┐          ┌─────────┐
│ Browser  │          │  Wings  │
└────┬─────┘          └────┬────┘
     │  Bearer <daemon_token> │
     │────────────────────>│ GET /api/system
     │  返回 {version: "1.7.0"} │
     │<────────────────────│
     │  直接显示版本号，不比较  │
     │
     │
用户 Websocket 连接:
┌─────────┐          ┌─────────┐          ┌─────────┐
│  User   │          │  Panel  │          │  Wings  │
└────┬────┘          └────┬────┘          └────┬────┘
     │  请求 Websocket       │                       │
     │────────────────────>│                       │
     │                       │ 1. 检查权限            │
     │                       │ 2. 生成 JWT(10min) │
     │  返回 JWT + socket │
     │<────────────────────│                       │
     │                                               │
     │  WSS 连接 + JWT                           │
     │───────────────────────────────────────────>│
     │                                               │ 验证 JWT 签名
     │                                               │ 检查权限声明
     │  建立连接                                       │
     │<───────────────────────────────────────────│
```

---

## 9. 关键代码勘误表（最终版）

| 原分析描述 | 实际情况 | 影响 |
|-----------|----------|------|
| `daemon.configuration` 是 Remote API 的例外路由，Wings 可无需认证获取配置 | 例外路由名称不匹配任何实际路由，所有 `/api/remote/*` 都需要认证 | 首次接入必须通过 Application API 或管理后台获取配置 |
| 节点配置获取通过 Remote API 进行 | 节点配置通过 Admin 后台或 Application API 获取，不属于 Remote API | Wings 无法主动"拉取"配置，必须通过管理员预先获取 |
| `isLatestDaemon()` 在节点心跳展示中被消费 | 该方法**定义但未被调用**，前端展示只显示版本号，不进行比较 | 版本比较逻辑实际上是死代码 |
| `isLatestDaemon()` 在 CLI 命令中使用 | CLI 命令只调用 `isLatestPanel()`，不调用 `isLatestDaemon()` | Wings 版本没有自动更新提示 |
| api-remote.php 中的路由有名称 | 所有路由都没有设置 `->name()`，`$route->getName()` 返回 `null` | 中间件例外路由机制无法生效 |
| Deployment Token 权限只有节点读取权限 | ✅ 正确，`r_nodes = 1`，使用位运算检查权限 | 遵循最小权限原则 |

---

## 10. 关键文件索引

| 文件路径 | 主要职责 |
|---------|----------|
| `app/Services/Nodes/NodeCreationService.php` | 节点创建，凭据生成 |
| `app/Services/Nodes/NodeJWTService.php` | JWT 临时令牌生成 |
| `app/Services/Nodes/NodeUpdateService.php` | 节点更新，凭据重置 |
| `app/Http/Middleware/Api/Daemon/DaemonAuthenticate.php` | Wings API 认证中间件 |
| `app/Http/Controllers/Admin/NodeAutoDeployController.php` | 自动部署 Token 生成 |
| `app/Http/Controllers/Api/Application/Nodes/NodeConfigurationController.php` | 节点配置输出（Application API） |
| `app/Http/Controllers/Admin/Nodes/SystemInformationController.php` | 系统信息获取（管理后台） |
| `app/Http/Controllers/Admin/Nodes/NodeViewController.php` | 节点视图控制器 |
| `app/Models/Node.php` | 节点模型，凭据解密 |
| `app/Models/ApiKey.php` | API Key 模型 |
| `app/Services/Acl/Api/AdminAcl.php` | ACL 权限系统 |
| `app/Http/Requests/Api/Application/ApplicationApiRequest.php` | Application API 请求基类 |
| `app/Http/Middleware/Api/Application/AuthenticateApplicationUser.php` | Application API 认证中间件 |
| `app/Services/Helpers/SoftwareVersionService.php` | 版本兼容性检查 |
| `app/Repositories/Wings/DaemonRepository.php` | Wings API 客户端基类 |
| `app/Repositories/Wings/DaemonConfigurationRepository.php` | Wings 配置/系统信息 |
| `routes/api-remote.php` | Wings Remote API 路由（均无名称） |
| `routes/api-application.php` | Application API 路由 |
| `routes/admin.php` | 管理后台路由 |
