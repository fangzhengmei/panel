# Pterodactyl Panel 文件管理器与 Wings 同步实现分析

## 概述

Pterodactyl Panel 的文件管理器采用 **"Panel 做认证授权，Wings 做实际文件操作"** 的架构模式。所有文件操作请求最终都由 Wings 守护进程直接执行，Panel 主要负责：
- 用户身份认证与权限校验
- 请求转发与 JWT 签名
- API 接口聚合与数据转换

---

## 一、完整请求链路：从路由到 Wings

### 1.1 路由与中间件链

**文件**: `routes/api-client.php:57-98`

所有服务器相关 API 都经过三层核心中间件：

```php
Route::group([
    'prefix' => '/servers/{server}',
    'middleware' => [
        ServerSubject::class,           // 1. 设置活动日志主体
        AuthenticateServerAccess::class, // 2. 验证服务器访问权限
        ResourceBelongsToServer::class,  // 3. 验证资源归属
    ],
], function () {
    // 文件管理器路由组
    Route::group(['prefix' => '/files'], function () {
        Route::get('/list', [FileController::class, 'directory']);
        Route::get('/contents', [FileController::class, 'contents']);
        Route::get('/download', [FileController::class, 'download']);
        Route::put('/rename', [FileController::class, 'rename']);
        Route::post('/copy', [FileController::class, 'copy']);
        Route::post('/write', [FileController::class, 'write']);
        Route::post('/compress', [FileController::class, 'compress']);
        Route::post('/decompress', [FileController::class, 'decompress']);
        Route::post('/delete', [FileController::class, 'delete']);
        Route::post('/create-folder', [FileController::class, 'create']);
        Route::post('/chmod', [FileController::class, 'chmod']);
        Route::post('/pull', [FileController::class, 'pull'])->middleware(ResourceLimit::FilePull->middleware());
        Route::get('/upload', FileUploadController::class);
    });
});
```

### 1.2 中间件详解

**ServerSubject 中间件**
**文件**: `app/Http/Middleware/Activity/ServerSubject.php:19-28`

```php
public function handle(Request $request, \Closure $next)
{
    $server = $request->route()->parameter('server');
    if ($server instanceof Server) {
        LogTarget::setActor($request->user());    // 设置日志操作者
        LogTarget::setSubject($server);            // 设置日志主体
    }

    return $next($request);
}
```

**AuthenticateServerAccess 中间件**
**文件**: `app/Http/Middleware/Api/Client/Server/AuthenticateServerAccess.php:29-67`

```php
public function handle(Request $request, \Closure $next): mixed
{
    $user = $request->user();
    $server = $request->route()->parameter('server');

    // 1. 验证服务器存在
    if (!$server instanceof Server) {
        throw new NotFoundHttpException();
    }

    // 2. 验证用户权限：所有者、管理员、或子用户
    if ($user->id !== $server->owner_id && !$user->root_admin) {
        if (!$server->subusers->contains('user_id', $user->id)) {
            throw new NotFoundHttpException();
        }
    }

    // 3. 验证服务器状态：非暂停、非安装中
    $server->validateCurrentState();

    // 4. 将服务器实例注入请求属性
    $request->attributes->set('server', $server);

    return $next($request);
}
```

**ResourceBelongsToServer 中间件**
**文件**: `app/Http/Middleware/Api/Client/Server/ResourceBelongsToServer.php:27-85`

```php
public function handle(Request $request, \Closure $next): mixed
{
    $server = $request->route()->parameter('server');
    
    // 遍历路由参数，检查除 server 外的其他 Model 是否属于该服务器
    foreach ($params as $key => $model) {
        if ($key === 'server' || !$model instanceof Model) {
            continue;
        }

        switch (get_class($model)) {
            case Allocation::class:
            case Backup::class:
            case Database::class:
            case Schedule::class:
            case Subuser::class:
                if ($model->server_id !== $server->id) {
                    throw $exception;
                }
                break;
            // ... 其他类型检查
        }
    }

    return $next($request);
}
```

> **代码证据说明**：对于纯文件操作路由（如 `/files/list`、`/files/upload`），路由参数中只有 `server`，没有其他 Model。因此 `ResourceBelongsToServer` 中间件在这些路由上实际不执行任何校验。

### 1.3 FormRequest 权限校验

**ClientPermissionsRequest 接口**
**文件**: `app/Contracts/Http/ClientPermissionsRequest.php:5-13`

```php
interface ClientPermissionsRequest
{
    public function permission(): string;
}
```

**ClientApiRequest 基类**
**文件**: `app/Http/Requests/Api/Client/ClientApiRequest.php:17-32`

```php
public function authorize(): bool
{
    // 如果请求类实现了 ClientPermissionsRequest 接口或定义了 permission() 方法
    if ($this instanceof ClientPermissionsRequest || method_exists($this, 'permission')) {
        $server = $this->route()->parameter('server');

        if ($server instanceof Server) {
            // 调用 Laravel Gate 检查用户是否有该权限
            return $this->user()->can($this->permission(), $server);
        }

        return false;
    }

    return true;
}
```

> **代码证据说明**：`FileController::download()` 方法签名使用 `GetFileContentsRequest`。虽然存在名为 `DownloadFileRequest` 的类（重写了 `authorize()` 硬编码 `'file.read'`），但**在当前代码中未被路由使用**。

### 1.4 完整调用链示例

**示例 1: 列出目录**

```
1. 用户请求 GET /api/client/servers/{uuid}/files/list
   ↓
2. 路由匹配，进入中间件链
   ├─ ServerSubject: 设置 LogTarget::actor 和 LogTarget::subject
   ├─ AuthenticateServerAccess: 验证服务器存在、用户权限、服务器状态
   └─ ResourceBelongsToServer: 无额外路由参数，直接通过
   ↓
3. FormRequest 权限校验
   └─ ListFilesRequest::permission() → Permission::ACTION_FILE_READ → 'file.read'
      └─ $user->can('file.read', $server) → Laravel Gate 校验
   ↓
4. FileController::directory() 执行
   └─ DaemonFileRepository::setServer($server)->getDirectory()
      ↓
5. Guzzle HTTP 请求到 Wings
   └─ GET /api/servers/{uuid}/files/list-directory
      ├─ Authorization: Bearer {node.daemon_token}
      └─ query: directory={path}
      ↓
6. Wings 访问容器文件系统，返回 JSON
   ↓
7. Fractal 转换器转换数据格式 (FileObjectTransformer)
   ↓
8. 返回响应给用户
```

**示例 2: 获取下载链接**

```
1. 用户请求 GET /api/client/servers/{uuid}/files/download?file={path}
   ↓
2. 路由匹配，进入中间件链（同上）
   ↓
3. FormRequest 权限校验
   └─ GetFileContentsRequest::permission() → Permission::ACTION_FILE_READ_CONTENT → 'file.read-content'
      └─ $user->can('file.read-content', $server) → Laravel Gate 校验
   ↓
4. FileController::download() 执行
   ├─ 生成 JWT 令牌（含 file_path、server_uuid、user_uuid）
   ├─ 记录 Activity Log: server:file.download
   └─ 返回签名的 Wings 下载 URL
   ↓
5. 用户浏览器直接访问 Wings URL
   └─ GET {node_address}/download/file?token={jwt}
      ↓
6. Wings 验证 JWT 并读取容器文件系统
   ↓
7. Wings 返回文件流给用户浏览器
```

---

## 二、请求转发机制

### 2.1 DaemonRepository 基类

**文件**: `app/Repositories/Wings/DaemonRepository.php:11-65`

所有 Wings 通信的基类，提供统一的 HTTP 客户端配置：

```php
abstract class DaemonRepository
{
    protected ?Server $server;
    protected ?Node $node;

    public function setServer(Server $server): self
    {
        $this->server = $server;
        $this->setNode($this->server->node);
        return $this;
    }

    public function getHttpClient(array $headers = []): Client
    {
        return new Client([
            'verify' => $this->app->environment('production'),
            'base_uri' => $this->node->getConnectionAddress(),
            'timeout' => config('pterodactyl.guzzle.timeout'),
            'connect_timeout' => config('pterodactyl.guzzle.connect_timeout'),
            'headers' => array_merge($headers, [
                'Authorization' => 'Bearer ' . $this->node->getDecryptedKey(),
                'Accept' => 'application/json',
                'Content-Type' => 'application/json',
            ]),
        ]);
    }
}
```

**关键点（代码证据）**:
- 使用节点的 `daemon_token`（解密后）作为 Bearer Token
- `base_uri` 动态指向节点地址：`scheme://fqdn:daemonListen`
- 支持超时配置（默认 15s 超时，5s 连接超时）

### 2.2 节点连接地址

**文件**: `app/Models/Node.php:134-137`

```php
public function getConnectionAddress(): string
{
    return sprintf('%s://%s:%s', $this->scheme, $this->fqdn, $this->daemonListen);
}
```

---

## 三、容器文件系统访问

### 3.1 DaemonFileRepository 文件操作仓库

**文件**: `app/Repositories/Wings/DaemonFileRepository.php:18-301`

这是 Panel 与 Wings 文件系统通信的核心类，封装了所有文件操作。

### 3.2 操作权限对照表（代码核对结果）

以下权限均来自各 FormRequest 类的 `permission()` 方法或 `authorize()` 方法：

| 操作方法 | Wings API 端点 | FormRequest 类 | 代码中的权限常量 | 实际权限值 |
|---------|---------------|----------------|-----------------|------------|
| `getDirectory()` | `GET /api/servers/{uuid}/files/list-directory` | `ListFilesRequest` | `Permission::ACTION_FILE_READ` | `'file.read'` |
| `getContent()` | `GET /api/servers/{uuid}/files/contents` | `GetFileContentsRequest` | `Permission::ACTION_FILE_READ_CONTENT` | `'file.read-content'` |
| `download()` | 直连 Wings `/download/file` | `GetFileContentsRequest` | `Permission::ACTION_FILE_READ_CONTENT` | `'file.read-content'` |
| `upload` | 直连 Wings `/upload/file` | `UploadFileRequest` | `Permission::ACTION_FILE_CREATE` | `'file.create'` |
| `putContent()` | `POST /api/servers/{uuid}/files/write` | `WriteFileContentRequest` | `Permission::ACTION_FILE_CREATE` | `'file.create'` |
| `createDirectory()` | `POST /api/servers/{uuid}/files/create-directory` | `CreateFolderRequest` | `Permission::ACTION_FILE_CREATE` | `'file.create'` |
| `copyFile()` | `POST /api/servers/{uuid}/files/copy` | `CopyFileRequest` | `Permission::ACTION_FILE_CREATE` | `'file.create'` |
| `decompressFile()` | `POST /api/servers/{uuid}/files/decompress` | `DecompressFilesRequest` | `Permission::ACTION_FILE_CREATE` | `'file.create'` |
| `pull()` | `POST /api/servers/{uuid}/files/pull` | `PullFileRequest` | `Permission::ACTION_FILE_CREATE` | `'file.create'` |
| `renameFiles()` | `PUT /api/servers/{uuid}/files/rename` | `RenameFileRequest` | `Permission::ACTION_FILE_UPDATE` | `'file.update'` |
| `chmodFiles()` | `POST /api/servers/{uuid}/files/chmod` | `ChmodFilesRequest` | `Permission::ACTION_FILE_UPDATE` | `'file.update'` |
| `deleteFiles()` | `POST /api/servers/{uuid}/files/delete` | `DeleteFileRequest` | `Permission::ACTION_FILE_DELETE` | `'file.delete'` |
| `compressFiles()` | `POST /api/servers/{uuid}/files/compress` | `CompressFilesRequest` | `Permission::ACTION_FILE_ARCHIVE` | `'file.archive'` |

**权限常量定义**
**文件**: `app/Models/Permission.php:51-57`

```php
public const ACTION_FILE_READ = 'file.read';
public const ACTION_FILE_READ_CONTENT = 'file.read-content';
public const ACTION_FILE_CREATE = 'file.create';
public const ACTION_FILE_UPDATE = 'file.update';
public const ACTION_FILE_DELETE = 'file.delete';
public const ACTION_FILE_ARCHIVE = 'file.archive';
public const ACTION_FILE_SFTP = 'file.sftp';
```

### 3.3 FileController 控制器

**文件**: `app/Http/Controllers/Api/Client/Servers/FileController.php:26-265`

控制器负责：
1. 接收用户请求（FormRequest 已完成权限校验）
2. 调用 `DaemonFileRepository` 转发请求
3. 记录操作日志 (Activity Log)
4. 返回转换后的数据

**代码示例**:
```php
public function directory(ListFilesRequest $request, Server $server): array
{
    $contents = $this->fileRepository
        ->setServer($server)
        ->getDirectory($request->get('directory') ?? '/');

    return $this->fractal->collection($contents)
        ->transformWith($this->getTransformer(FileObjectTransformer::class))
        ->toArray();
}
```

### 3.4 文件大小限制

**文件**: `app/Repositories/Wings/DaemonFileRepository.php:29-50`

读取文件时有大小限制，防止大文件导致内存溢出：

```php
public function getContent(string $path, ?int $notLargerThan = null): string
{
    // ...
    $length = (int) Arr::get($response->getHeader('Content-Length'), 0, 0);
    if ($notLargerThan && $length > $notLargerThan) {
        throw new FileSizeTooLargeException();
    }
    // ...
}
```

配置项: `config('pterodactyl.files.max_edit_size')`

---

## 四、JWT 认证机制：jti 与 unique_id 的语义边界

### 4.1 NodeJWTService 服务

**文件**: `app/Services/Nodes/NodeJWTService.php:15-103`

用于生成 Wings 可验证的 JWT 令牌，主要用于文件下载和上传的授权。

```php
public function handle(Node $node, ?string $identifiedBy, string $algo = 'md5'): UnencryptedToken
{
    // jti: 基于传入标识的哈希值
    $identifier = hash($algo, $identifiedBy);
    
    $config = Configuration::forSymmetricSigner(
        new Sha256(), 
        InMemory::plainText($node->getDecryptedKey())
    );

    $builder = $config->builder(new TimestampDates())
        ->issuedBy(config('app.url'))                    // iss: Panel URL
        ->permittedFor($node->getConnectionAddress())     // aud: 节点地址
        ->identifiedBy($identifier)                       // jti: 标准声明
        ->withHeader('jti', $identifier)                  // 额外在 header 中设置
        ->issuedAt(CarbonImmutable::now())                // iat: 签发时间
        ->canOnlyBeUsedAfter(CarbonImmutable::now()->subMinutes(5)); // nbf: 5分钟前

    if (isset($this->expiresAt)) {
        $builder = $builder->expiresAt($this->expiresAt); // exp: 过期时间
    }

    // ... 添加自定义 claims

    return $builder
        ->withClaim('unique_id', Str::random())           // unique_id: 自定义随机字段
        ->getToken($config->signer(), $config->signingKey());
}
```

### 4.2 jti 与 unique_id 的语义边界（基于代码证据）

| 字段 | 代码来源 | 语义 | 代码证据 |
|------|---------|------|---------|
| **jti** | `identifiedBy($identifier)`，其中 `$identifier = hash($algo, $identifiedBy)` | 基于传入字符串的哈希值，相同输入会生成相同的 jti | `app/Services/Nodes/NodeJWTService.php:65: $identifier = hash($algo, $identifiedBy);` |
| **unique_id** | `withClaim('unique_id', Str::random())` | 完全随机的字符串，每次调用都不同 | `app/Services/Nodes/NodeJWTService.php:100: ->withClaim('unique_id', Str::random())` |

**可被代码直接证明的结论**：
1. jti 是可重复的：相同的 `$identifiedBy` 输入会生成相同的 jti
   - 证据：`$identifier = hash($algo, $identifiedBy)`，哈希函数是确定性的
2. unique_id 不是 JWT 标准声明：它是通过 `withClaim()` 方法添加的自定义字段
   - 证据：使用 `->withClaim('unique_id', Str::random())` 而非 `identifiedBy()` 等标准方法
3. Panel 代码中**没有**任何基于 jti 或 unique_id 的"令牌已使用"校验逻辑
   - 证据：在 Panel 代码库中未找到任何查询、存储或比较 jti/unique_id 的逻辑
4. Panel 代码中**没有**任何存储 jti/unique_id 的数据库表或缓存逻辑
   - 证据：数据库迁移文件和模型中无相关表定义

### 4.3 "one-time token" 语义辨析（代码证据）

**PHPDoc 注释证据**：
**文件**: `app/Http/Controllers/Api/Client/Servers/FileController.php:72`

```php
/**
 * Generates a one-time token with a link that the user can use to
 * download a given file.
 */
```

同样的注释也出现在 WebSocket 令牌生成处：
**文件**: `app/Http/Controllers/Api/Client/Servers/WebsocketController.php:28`

```php
/**
 * Generates a one-time token that is sent along in every websocket call to the Daemon.
 */
```

**JTI 撤销机制证据**（用于 WebSocket 令牌）：
**文件**: `app/Repositories/Wings/DaemonServerRepository.php:129-164`

```php
/**
 * Revokes a single user's JTI by using their ID.
 *
 * @deprecated
 * @see \Pterodactyl\Repositories\Wings\DaemonRevocationRepository::deauthorize()
 */
public function revokeUserJTI(int $id): void
{
    $this->revokeJTIs([md5($id . $this->server->uuid)]);
}

/**
 * Revokes an array of JWT JTI's by marking any token generated before the current time on
 * the Wings instance as being invalid.
 */
protected function revokeJTIs(array $jtis): void
{
    $this->getHttpClient()
        ->post(sprintf('/api/servers/%s/ws/deny', $this->server->uuid), [
            'json' => ['jtis' => $jtis],
        ]);
}
```

**结论（基于可证明的代码事实）**：
1. "one-time" 是 PHPDoc 中的注释表述，而非代码强制执行的机制
2. Panel 端**没有**令牌使用跟踪逻辑，令牌是否"单次使用"完全由 Wings 端决定
3. JTI 撤销机制存在但**仅用于 WebSocket 令牌**（通过 `/api/servers/{uuid}/ws/deny` 端点），代码中未见用于文件上传/下载令牌的撤销调用
4. 文件上传/下载令牌的"单次使用"语义**在 Panel 代码中无直接证据**，需查看 Wings 代码确认

### 4.4 JWT 载荷结构（代码证据）

**文件上传令牌**（`FileUploadController.php:42-46`）:
```php
$token = $this->jwtService
    ->setExpiresAt(CarbonImmutable::now()->addMinutes(15))
    ->setUser($user)
    ->setClaims(['server_uuid' => $server->uuid])
    ->handle($server->node, $user->id . $server->uuid);
```

**文件下载令牌**（`FileController.php:79-86`）:
```php
$token = $this->jwtService
    ->setExpiresAt(CarbonImmutable::now()->addMinutes(15))
    ->setUser($request->user())
    ->setClaims([
        'file_path' => rawurldecode($request->get('file')),
        'server_uuid' => $server->uuid,
    ])
    ->handle($server->node, $request->user()->id . $server->uuid);
```

**标准 JWT 声明**（代码证据）：
- `iss`: `config('app.url')` - Panel URL (签发者)
- `aud`: `$node->getConnectionAddress()` - 节点连接地址 (接收者)
- `jti`: `hash($algo, $identifiedBy)` - 基于输入的哈希标识
- `iat`: `CarbonImmutable::now()` - 签发时间
- `nbf`: `CarbonImmutable::now()->subMinutes(5)` - 最早使用时间
- `exp`: `$this->expiresAt` - 过期时间（通常15分钟）

**自定义 claims**（代码证据）：
- `user_uuid`: `$this->user->uuid` - 用户 UUID
- `user_id`: `$this->user->id` - 用户 ID（注释说明已弃用）
- `server_uuid`: 仅上传/下载时设置
- `file_path`: 仅下载时设置
- `unique_id`: `Str::random()` - 随机字符串

---

## 五、文件上传真实机制：非分片，直连 Wings

### 5.1 上传流程架构（代码证据）

**重要更正**：文件上传**不存在**大文件分片实现。当前实现为 **"前端直连 Wings 的单段上传"**：

```
1. 前端请求 Panel 获取签名上传 URL
   (GET /api/client/servers/{uuid}/files/upload)
   ↓
2. Panel 生成 JWT 签名的 Wings URL
   ├─ 中间件链: ServerSubject → AuthenticateServerAccess → ResourceBelongsToServer
   ├─ 权限校验: UploadFileRequest::permission() → 'file.create'
   └─ JWT 包含: server_uuid, user_uuid, exp=15分钟
   ↓
3. 前端直接 POST 文件到 Wings
   ├─ URL: {node_address}/upload/file?token={jwt}
   ├─ Headers: Content-Type: multipart/form-data
   ├─ Query: directory={target_directory}
   └─ Body: files={file_content}
   ↓
4. Wings 验证 JWT 并写入容器文件系统
   ↓
5. 前端通过 axios onUploadProgress 回调显示上传进度
```

### 5.2 步骤详解（代码证据）

**步骤 1: 获取签名上传 URL**

**文件**: `app/Http/Controllers/Api/Client/Servers/FileUploadController.php:27-53`

```php
public function __invoke(UploadFileRequest $request, Server $server): JsonResponse
{
    return new JsonResponse([
        'object' => 'signed_url',
        'attributes' => [
            'url' => $this->getUploadUrl($server, $request->user()),
        ],
    ]);
}

protected function getUploadUrl(Server $server, User $user): string
{
    $token = $this->jwtService
        ->setExpiresAt(CarbonImmutable::now()->addMinutes(15))
        ->setUser($user)
        ->setClaims(['server_uuid' => $server->uuid])
        ->handle($server->node, $user->id . $server->uuid);

    return sprintf(
        '%s/upload/file?token=%s',
        $server->node->getConnectionAddress(),
        $token->toString()
    );
}
```

**前端 API 调用**:
**文件**: `resources/scripts/api/server/files/getFileUploadUrl.ts:1-9`

```typescript
export default (uuid: string): Promise<string> => {
    return http.get(`/api/client/servers/${uuid}/files/upload`)
        .then(({ data }) => data.attributes.url);
};
```

**步骤 2: 前端上传到 Wings**

**文件**: `resources/scripts/components/server/files/UploadButton.tsx:64-101`

```typescript
const onFileSubmission = (files: FileList) => {
    clearAndAddHttpError();
    const list = Array.from(files);
    
    // 文件夹检测 (4096字节是典型目录大小)
    if (list.some((file) => !file.type && (!file.size || file.size === 4096))) {
        return addError('Folder uploads are not supported.', 'Error');
    }

    const uploads = list.map((file) => {
        const controller = new AbortController();
        pushFileUpload({
            name: file.name,
            data: { abort: controller, loaded: 0, total: file.size },
        });

        return () =>
            getFileUploadUrl(uuid).then((url) =>
                axios
                    .post(
                        url,
                        { files: file },
                        {
                            signal: controller.signal,
                            headers: { 'Content-Type': 'multipart/form-data' },
                            params: { directory },
                            onUploadProgress: (data) => onUploadProgress(data, file.name),
                        }
                    )
                    .then(() => timeouts.value.push(setTimeout(() => removeFileUpload(file.name), 500)))
            );
    });

    Promise.all(uploads.map((fn) => fn()))
        .then(() => mutate())
        .catch((error) => {
            clearFileUploads();
            clearAndAddHttpError(error);
        });
};
```

> **代码证据说明**：axios `post` 直接发送完整的 `file` 对象，没有任何分片逻辑。`onUploadProgress` 是浏览器 XMLHttpRequest 的原生进度事件。

### 5.3 上传进度反馈（代码证据）

**状态管理**:
**文件**: `resources/scripts/state/server/files.ts:4-79`

```typescript
export interface FileUploadData {
    loaded: number;           // 已上传字节数
    readonly abort: AbortController;
    readonly total: number;   // 文件总大小
}

export interface ServerFileStore {
    uploads: Record<string, FileUploadData>;
    
    pushFileUpload: Action<ServerFileStore, { name: string; data: FileUploadData }>;
    setUploadProgress: Action<ServerFileStore, { name: string; loaded: number }>;
    clearFileUploads: Action<ServerFileStore>;
    removeFileUpload: Action<ServerFileStore, string>;
    cancelFileUpload: Action<ServerFileStore, string>;
}
```

**进度更新回调**:
**文件**: `resources/scripts/components/server/files/UploadButton.tsx:60-62`

```typescript
const onUploadProgress = (data: AxiosProgressEvent, name: string) => {
    setUploadProgress({ name, loaded: data.loaded });
};
```

**进度计算（代码证据）**:
- `total`: `file.size` - 文件总大小
- `loaded`: `AxiosProgressEvent.loaded` - 浏览器已发送字节数
- 进度百分比: `(loaded / total) * 100`

### 5.4 上传失败时的进度清空影响（代码证据）

**文件**: `resources/scripts/components/server/files/UploadButton.tsx:95-100`

```typescript
Promise.all(uploads.map((fn) => fn()))
    .then(() => mutate())
    .catch((error) => {
        clearFileUploads();  // 清空所有上传状态
        clearAndAddHttpError(error);
    });
```

**状态清空实现**:
**文件**: `resources/scripts/state/server/files.ts:48-52`

```typescript
clearFileUploads: action((state) => {
    // 中止所有正在进行的请求
    Object.values(state.uploads).forEach((upload) => upload.abort.abort());
    // 清空状态
    state.uploads = {};
}),
```

**对反馈语义的影响（可被代码证明）**：

1. **全部或全无**：`Promise.all()` 的特性是只要有一个 Promise reject，整个 Promise 就 reject，因此只要有一个文件上传失败，就会进入 `catch` 块调用 `clearFileUploads()`
2. **用户无法获知部分成功**：`clearFileUploads()` 会清空所有上传记录，包括已成功的
3. **进度丢失**：所有 `loaded` 值被清空
4. **重试成本**：用户需要重新选择所有文件

### 5.5 上传取消机制（代码证据）

**文件**: `resources/scripts/state/server/files.ts:70-78`

```typescript
cancelFileUpload: action((state, payload) => {
    if (state.uploads[payload]) {
        state.uploads[payload].abort.abort();  // 调用 AbortController.abort()
        delete state.uploads[payload];
    }
}),
```

---

## 六、对比：备份多段上传链路（代码证据）

**重要区分**：只有备份上传才实现了真正的多段 (multipart) 上传。

### 6.1 备份多段上传架构

**文件**: `app/Http/Controllers/Api/Remote\Backups\BackupRemoteUploadController.php:16-134`

```
Wings 请求 Panel 获取多段上传信息
   (GET /api/remote/backups/{backup}/upload?size={size})
   ↓
Panel 调用 S3 CreateMultipartUpload API
   ↓
Panel 返回多个 presigned URL (每个分片一个)
   ↓
Wings 使用 presigned URL 分片上传到 S3
   ↓
Wings 通知 Panel 上传完成
```

**核心代码**:
```php
public const DEFAULT_MAX_PART_SIZE = 5 * 1024 * 1024 * 1024; // 5GB per part

public function __invoke(Request $request, string $backup): JsonResponse
{
    $size = (int) $request->query('size');
    
    // 1. 调用 S3 CreateMultipartUpload
    $result = $client->execute($client->getCommand('CreateMultipartUpload', $params));
    $params['UploadId'] = $result->get('UploadId');
    
    // 2. 为每个分片生成 presigned URL
    $maxPartSize = $this->getConfiguredMaxPartSize();
    $parts = [];
    for ($i = 0; $i < ($size / $maxPartSize); ++$i) {
        $parts[] = $client->createPresignedRequest(
            $client->getCommand('UploadPart', array_merge($params, ['PartNumber' => $i + 1])),
            $expires
        )->getUri()->__toString();
    }
    
    return new JsonResponse([
        'parts' => $parts,
        'part_size' => $maxPartSize,
    ]);
}
```

### 6.2 文件上传 vs 备份上传 对比表

| 特性 | 文件管理器上传（代码证据） | 备份 S3 多段上传（代码证据） |
|------|---------------------------|-----------------------------|
| **实现方式** | `axios.post(url, { files: file })` 单段发送 | S3 `CreateMultipartUpload` + 多个 `UploadPart` |
| **分片支持** | ❌ 无分片代码 | ✅ `for ($i = 0; $i < ($size / $maxPartSize); ++$i)` 循环生成多片 URL |
| **数据流向** | 前端 → Wings | Wings → S3 (通过 presigned URL) |
| **进度反馈** | `onUploadProgress` 浏览器原生事件 | 由 Wings 内部处理，无进度回调到 Panel |
| **端点位置** | Wings `/upload/file` | S3 各分片 URL |
| **认证方式** | JWT in query string | S3 presigned URL |
| **Panel 角色** | 生成签名 URL | 生成 S3 presigned URL 列表 |
| **分片大小** | 无 | 默认 5GB (`DEFAULT_MAX_PART_SIZE`) |

---

## 七、文件下载流程（代码证据）

**文件**: `app/Http/Controllers/Api/Client/Servers/FileController.php:71-100`

### 7.1 权限校验入口

```php
/**
 * Generates a one-time token with a link that the user can use to
 * download a given file.
 *
 * @throws \Throwable
 */
public function download(GetFileContentsRequest $request, Server $server): array
{
```

> **代码证据**：
> - 方法签名使用 `GetFileContentsRequest`，不是 `DownloadFileRequest`
> - 权限校验入口为 `GetFileContentsRequest::permission()` → `Permission::ACTION_FILE_READ_CONTENT` → `'file.read-content'`
> - 路由定义：`routes/api-client.php:86: Route::get('/download', [FileController::class, 'download'])`

### 7.2 JWT 生成与直连模式

```php
public function download(GetFileContentsRequest $request, Server $server): array
{
    $token = $this->jwtService
        ->setExpiresAt(CarbonImmutable::now()->addMinutes(15))
        ->setUser($request->user())
        ->setClaims([
            'file_path' => rawurldecode($request->get('file')),
            'server_uuid' => $server->uuid,
        ])
        ->handle($server->node, $request->user()->id . $server->uuid);

    Activity::event('server:file.download')->property('file', $request->get('file'))->log();

    return [
        'object' => 'signed_url',
        'attributes' => [
            'url' => sprintf(
                '%s/download/file?token=%s',
                $server->node->getConnectionAddress(),
                $token->toString()
            ),
        ],
    ];
}
```

---

## 八、设计亮点与权衡（基于代码证据）

### 8.1 优点

1. **流量卸载**: 大文件上传/下载直接走 Wings，不占用 Panel 带宽
2. **分层权限**: 中间件粗粒度校验 + FormRequest 细粒度权限检查
3. **实时进度**: 前端直连可获得精确的上传进度（浏览器原生 `progress` 事件）
4. **可取消**: 基于 `AbortController` 支持上传取消
5. **超时保护**: 压缩/解压操作设置 15 分钟长超时（`DaemonFileRepository.php:212: 'timeout' => 60 * 15`）

### 8.2 权衡点（代码证据）

1. **跨域问题**: 前端直接请求 Wings 需要 CORS 配置（代码中未体现 Panel 端的 CORS 处理）
2. **Token 时效**: 15 分钟过期，超大文件上传可能超时（`CarbonImmutable::now()->addMinutes(15)`）
3. **单文件签名**: 每个文件单独请求签名 URL（`uploads.map((file) => getFileUploadUrl(uuid))`）
4. **失败处理粗糙**: 上传失败时清空所有进度（`Promise.all()` + `clearFileUploads()`）
5. **无断点续传**: 非分片上传，失败后需从头开始（无分片/断点相关代码）

### 8.3 安全机制（代码证据）

- **节点密钥认证**: Panel → Wings 内部通信使用 `$this->node->getDecryptedKey()` 作为 Bearer Token
- **JWT 用户认证**: 上传/下载时 JWT 包含 `user_uuid` 和 `server_uuid`，Wings 可验证
- **时间窗口**: `nbf` 设置为签发前 5 分钟（`canOnlyBeUsedAfter(CarbonImmutable::now()->subMinutes(5))`）
- **过期机制**: `exp` 设置为 15 分钟后过期（`CarbonImmutable::now()->addMinutes(15)`）
- **随机因子**: `unique_id` 使用 `Str::random()` 增加令牌不可预测性
- **权限分层**: 中间件 + FormRequest + Laravel Gate 三层校验
- **JTI 撤销（WebSocket 专用）**: 存在 JTI 撤销机制（`/api/servers/{uuid}/ws/deny`），但仅用于 WebSocket 令牌，代码中未见用于文件上传/下载令牌

---

## 九、关键代码位置速查表

| 功能 | 文件路径 |
|------|---------|
| Wings 文件操作仓库 | `app/Repositories/Wings/DaemonFileRepository.php` |
| Wings 通信基类 | `app/Repositories/Wings/DaemonRepository.php` |
| 文件 API 控制器 | `app/Http/Controllers/Api/Client/Servers/FileController.php` |
| 文件上传控制器 | `app/Http/Controllers/Api/Client/Servers/FileUploadController.php` |
| 备份多段上传控制器 | `app/Http/Controllers/Api/Remote/Backups/BackupRemoteUploadController.php` |
| 服务器访问权限中间件 | `app/Http/Middleware/Api/Client/Server/AuthenticateServerAccess.php` |
| 资源归属校验中间件 | `app/Http/Middleware/Api/Client/Server/ResourceBelongsToServer.php` |
| 活动日志主体中间件 | `app/Http/Middleware/Activity/ServerSubject.php` |
| 客户端 API 请求基类 | `app/Http/Requests/Api/Client/ClientApiRequest.php` |
| 权限常量定义 | `app/Models/Permission.php` |
| JWT 签名服务 | `app/Services/Nodes/NodeJWTService.php` |
| 前端上传组件 | `resources/scripts/components/server/files/UploadButton.tsx` |
| 前端上传状态 | `resources/scripts/state/server/files.ts` |
| 节点模型 | `app/Models/Node.php` |
| API 路由定义 | `routes/api-client.php` |
