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

**用途**：生成仅具有节点读取权限的 Application API Key，用于一键部署命令：
```bash
cd /etc/pterodactyl && sudo wings configure --panel-url <panel_url> --token <deployment_token> --node <node_id>
```

### 2.3 节点配置获取（首次接入）

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

**首次接入流程**：
1. 管理员在 Panel 创建节点 → 生成 `daemon_token_id` + `daemon_token`
2. 管理员通过管理后台或 Application API 获取节点配置（含明文 token）
3. 将配置复制到 Wings 服务器的 `/etc/pterodactyl/config.yml`
4. Wings 启动后，使用配置中的 token 与 Panel 通信

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

### 4.3 版本比较结果的消费路径

> **⚠️ 关键发现**：`isLatestDaemon()` 方法**没有在节点心跳展示中被消费**！

**版本比较方法定义**：`app/Services/Helpers/SoftwareVersionService.php:74-81`

```php
public function isLatestDaemon(string $version): bool
{
    if ($version === 'develop') {
        return true;
    }
    return version_compare($version, $this->getDaemon()) >= 0;
}
```

**实际消费情况**：

| 位置 | 消费方式 |
|------|----------|
| `app/Console/Commands/InfoCommand.php:32` | CLI 命令中显示 Panel 版本信息（只调用 `isLatestPanel()`） |
| `resources/views/admin/index.blade.php:19,29` | 管理首页显示 Panel 版本状态（只调用 `isLatestPanel()`） |
| `resources/views/admin/nodes/view/index.blade.php:42` | 节点详情页只显示最新版本号（调用 `getDaemon()`，不调用 `isLatestDaemon()`） |

**节点详情页展示**：`resources/views/admin/nodes/view/index.blade.php:42`

```html
<td>Daemon Version</td>
<td><code data-attr="info-version"><i class="fa fa-refresh fa-fw fa-spin"></i></code> (Latest: <code>{{ $version->getDaemon() }}</code>)</td>
```

**版本数据流程**：
1. 后端 `SystemInformationController` → 从 Wings 获取 `version` 字段
2. 前端 AJAX 获取 → 直接替换 `[data-attr="info-version"]` 的 HTML
3. 页面渲染 → 显示当前版本 + `$version->getDaemon()`（最新版本号）
4. **没有进行版本比较**，也没有根据版本是否最新显示不同状态

### 4.4 版本获取机制

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

1. **Deployment Token**：仅授予节点读取权限
2. **Remote API**：仅节点可访问，通过 `DaemonAuthenticate` 中间件保护
3. **用户操作**：通过 JWT 中的 permissions 声明进行细粒度权限控制

---

## 8. 完整认证时序图

```
Wings 首次接入流程:
┌─────────┐          ┌─────────┐
│  Admin  │          │  Panel  │
└────┬────┘          └────┬────┘
     │  1. 创建节点        │
     │────────────────────>│
     │                     │ 生成 daemon_token_id(16)
     │                     │ 生成 daemon_token(64)
     │  2. 获取配置       │
     │────────────────────>│ GET /admin/nodes/view/{id}/configuration
     │                     │ 或 GET /api/application/nodes/{id}/configuration
     │  返回含解密token    │
     │<────────────────────│
     │
     │  3. 复制配置到 Wings 服务器
     │  ==================> /etc/pterodactyl/config.yml
     │
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
     │                     │ 3. 解密 token 比对
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
     │  返回版本/系统信息      │
     │<────────────────────│
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

## 9. 关键代码勘误表

| 原分析描述 | 实际情况 | 影响 |
|-----------|----------|------|
| `daemon.configuration` 是 Remote API 的例外路由，无需认证 | 例外路由名称不匹配任何实际路由，所有 `/api/remote/*` 都需要认证 | 首次接入必须通过管理后台或 Application API 获取配置 |
| 节点配置获取通过 Remote API 进行 | 节点配置通过 Admin 后台或 Application API 获取，不属于 Remote API | Wings 无法主动"拉取"配置，必须管理员预先写入 |
| `isLatestDaemon()` 在节点心跳展示中被消费 | 该方法只在 CLI 命令中使用，前端展示只显示版本号，不进行比较 | 版本比较逻辑实际上未被前端使用 |
| api-remote.php 中的路由有名称 | 所有路由都没有设置 `->name()`，`$route->getName()` 返回 `null` | 中间件例外路由机制无法生效 |

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
| `app/Services/Helpers/SoftwareVersionService.php` | 版本兼容性检查 |
| `app/Repositories/Wings/DaemonRepository.php` | Wings API 客户端基类 |
| `app/Repositories/Wings/DaemonConfigurationRepository.php` | Wings 配置/系统信息 |
| `routes/api-remote.php` | Wings Remote API 路由（均无名称） |
| `routes/api-application.php` | Application API 路由 |
| `routes/admin.php` | 管理后台路由 |
