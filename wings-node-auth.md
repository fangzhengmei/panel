# Wings 节点接入与 Panel 端持久凭据签发协作分析

## 1. 架构概览

Wings（原 Daemon）与 Panel 的认证体系采用**双轨制认证**：
- **持久凭据**：用于节点与 Panel 之间的 API 通信（Wings → Panel）
- **临时 JWT**：用于用户操作鉴权（Panel → Wings）

---

## 2. Deployment Token 生成机制

### 2.1 节点创建时的持久凭据生成

**核心代码**：`app/Services/Nodes/NodeCreationService.php:25-32**

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

### 2.3 节点配置获取

**核心代码**：`app/Http/Controllers/Api/Application/Nodes/NodeConfigurationController.php:17-20**

```php
public function __invoke(GetNodeRequest $request, Node $node): JsonResponse
{
    return new JsonResponse($node->getConfiguration());
}
```

**配置输出**（含解密后的完整凭据：`app/Models/Node.php:142-168`

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
    if (count($parts) !== 2 || empty($parts[0]) || empty($parts[1]))) {
        throw new BadRequestHttpException('The Authorization header provided was not in a valid format.');
    }

    try {
        $node = $this->repository->findFirstWhere([
            'daemon_token_id' => $parts[0],
        ]);

        if (hash_equals((string) $this->encrypter->decrypt($node->daemon_token), $parts[1]))) {
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

**例外路由**（无需认证的路由：
- `daemon.configuration` - 节点配置获取（用于首次接入）

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

### 3.3 Remote API 端点

**核心代码**：`routes/api-remote.php`

| 端点 | 方法 | 用途 |
|------|------|------|
| `/api/remote/sftp/auth` | POST | SFTP 认证 |
| `/api/remote/servers` | GET | 获取节点服务器列表 |
| `/api/remote/servers/reset` | POST | 重置服务器状态 |
| `/api/remote/servers/{uuid}` | GET | 获取单个服务器配置 |
| `/api/remote/servers/{uuid}/install` | GET/POST | 服务器安装 |
| `/api/remote/servers/{uuid}/transfer/*` | POST | 服务器迁移 |
| `/api/remote/backups/*` | GET/POST | 备份操作 |
| `/api/remote/activity` | POST | 活动日志上报 |
| `/api/remote/eggs/install` | POST | Egg 安装脚本 |

### 3.4 凭据重置

**核心代码**：`app/Services/Nodes/NodeUpdateService.php:33-38`

```php
public function handle(Node $node, array $data, bool $resetToken = false): Node
{
    if ($resetToken) {
        $data['daemon_token'] = $this->encrypter->encrypt(Str::random(Node::DAEMON_TOKEN_LENGTH));
        $data['daemon_token_id'] = Str::random(Node::DAEMON_TOKEN_ID_LENGTH));
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
**目标端点**：Wings `/api/system`
**认证方式**：直接使用解密后的 daemon_token 作为 Bearer Token

### 4.2 后端系统信息获取

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
        throw new DaemonConnectionException($exception));
    }
    return json_decode($response->getBody()->__toString(), true));
}
```

### 4.3 版本兼容性校验

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

**版本获取**：`app/Services/Helpers/SoftwareVersionService.php:38-41`

```php
public function getDaemon(): string
{
    return Arr::get(self::$result, 'wings') ?? 'error';
}
```

**缓存机制**：缓存 60 分钟，从 CDN 获取最新版本信息

**前端展示**：`resources/views/admin/nodes/view/index.blade.php:42`

```html
<td>Daemon Version</td>
<td><code data-attr="info-version"><i class="fa fa-refresh fa-fw fa-spin"></i></code> (Latest: <code>{{ $version->getDaemon() }}</code></td>
```

**前端自动刷新**：每 10 秒调用一次 `/admin/nodes/view/{node}/system-information

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
        ->permittedFor($node->getConnectionAddress()))
        ->identifiedBy($identifier)
        ->withHeader('jti', $identifier)
        ->issuedAt(CarbonImmutable::now())
        ->canOnlyBeUsedAfter(CarbonImmutable::now()->subMinutes(5));

    if (isset($this->expiresAt)) {
        $builder = $builder->expiresAt($this->expiresAt));
    }

    if (!empty($this->subject)) {
        $builder = $builder->relatedTo($this->subject))->withHeader('sub', $this->subject));
    }

    foreach ($this->claims as $key => $value) {
        $builder = $builder->withClaim($key, $value));
    }

    if (!is_null($this->user)) {
        $builder = $builder
            ->withClaim('user_uuid', $this->user->uuid)
            ->withClaim('user_id', $this->user->id); //  // 向后兼容，1.11 后移除
    }

    return $builder
        ->withClaim('unique_id', Str::random())
        ->getToken($config->signer(), $config->signingKey()));
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
        throw new HttpForbiddenException('You do not have permission to connect to this server\'s websocket.'));
    }

    $permissions = $this->permissionsService->handle($server, $user);

    $token = $this->jwtService
        ->setExpiresAt(CarbonImmutable::now()->addMinutes(10))
        ->setUser($request->user()))
        ->setClaims([
            'server_uuid' => $server->uuid,
            'permissions' => $permissions,
        ])
        ->handle($node, $user->id . $server->uuid);

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
| 文件下载 | `app/Http/Controllers/Api/Client/Servers/FileController.php | 生成文件下载签名 |
| 文件上传 | `app/Http/Controllers/Api/Client/Servers/FileUploadController.php | 生成上传签名 |
| 备份下载 | `app/Services/Backups/DownloadLinkService.php | 备份下载链接 |
| 服务器迁移 | `app/Http/Controllers/Admin/Servers/ServerTransferController.php | 迁移认证 |

---

## 6. 版本兼容性设计

### 6.1 向后兼容设计

**NodeJWTService.php:91-96`

```php
// The "user_id" claim is deprecated and should not be referenced — it remains
// here solely to ensure older versions of Wings are unaffected when the Panel
// is updated.
//
// This claim will be removed in Panel@1.11 or later.
->withClaim('user_id', $this->user->id);
```

### 6.2 系统信息版本化

**DaemonConfigurationRepository.php:21-24`

```php
public function getSystemInformation(?int $version = null): array
{
    $response = $this->getHttpClient()->get('/api/system' . (!is_null($version) ? '?v=' . $version : ''));
}
```

**遥测服务使用 v2 版本：`app/Services/Telemetry/TelemetryCollectionService.php:61`

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
     │──────────────────────>│
     │                       │ 生成 daemon_token_id(16)
     │                       │ 生成 daemon_token(64)
     │  2. 获取配置       │
     │<──────────────────────│
     │                       │ 返回含解密后token
     │
     │
Wings 日常通信:
┌─────────┐          ┌─────────┐
│  Wings  │          │  Panel  │
└────┬────┘          └────┬────┘
     │  Bearer <id>.<token> │
     │──────────────────────>│
     │                       │ DaemonAuthenticate 验证
     │                       │ 1. 拆分 token
     │                       │ 2. 根据 id 查节点
     │                       │ 3. 解密 token 比对
     │  返回数据             │
     │<──────────────────────│
     │
     │
用户 Websocket 连接:
┌─────────┐          ┌─────────┐          ┌─────────┐
│  User   │          │  Panel  │          │  Wings  │
└────┬────┘          └────┬────┘          └────┬────┘
     │  请求 Websocket       │                       │
     │──────────────────────>│                       │
     │                       │ 1. 检查权限            │
     │                       │ 2. 生成 JWT(10min)│
     │  返回 JWT + socket │
     │<──────────────────────│                       │
     │                                               │
     │  WSS 连接 + JWT                           │
     │──────────────────────────────────────────────>│
     │                                               │ 验证 JWT 签名
     │                                               │ 检查权限声明
     │  建立连接                                       │
     │<──────────────────────────────────────────────│
```

---

## 9. 关键文件索引

| 文件路径 | 主要职责 |
|---------|----------|
| `app/Services/Nodes/NodeCreationService.php` | 节点创建，凭据生成 |
| `app/Services/Nodes/NodeJWTService.php` | JWT 临时令牌生成 |
| `app/Services/Nodes/NodeUpdateService.php` | 节点更新，凭据重置 |
| `app/Http/Middleware/Api/Daemon/DaemonAuthenticate.php` | Wings API 认证中间件 |
| `app/Http/Controllers/Admin/NodeAutoDeployController.php` | 自动部署 Token 生成 |
| `app/Http/Controllers/Api/Application/Nodes/NodeConfigurationController.php` | 节点配置输出 |
| `app/Http/Controllers/Admin/Nodes/SystemInformationController.php` | 系统信息获取 |
| `app/Models/Node.php` | 节点模型，凭据解密 |
| `app/Services/Helpers/SoftwareVersionService.php` | 版本兼容性检查 |
| `app/Repositories/Wings/DaemonRepository.php` | Wings API 客户端基类 |
| `app/Repositories/Wings/DaemonConfigurationRepository.php` | Wings 配置/系统信息 |
| `routes/api-remote.php` | Wings Remote API 路由 |
| `routes/api-application.php` | Application API 路由 |
