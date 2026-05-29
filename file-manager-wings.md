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

### 1.3 FormRequest 权限校验

**ClientApiRequest 基类**
**文件**: `app/Http/Requests/Api/Client/ClientApiRequest.php:17-32`

```php
public function authorize(): bool
{
    // 如果请求类定义了 permission() 方法，则执行细粒度权限检查
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

**具体权限示例**
**文件**: `app/Http/Requests/Api/Client/Servers/Files/UploadFileRequest.php:8-13`

```php
class UploadFileRequest extends ClientApiRequest
{
    public function permission(): string
    {
        return Permission::ACTION_FILE_CREATE; // 'file.create'
    }
}
```

**文件**: `app/Http/Requests/Api/Client/Servers/Files/ListFilesRequest.php:14-17`

```php
public function permission(): string
{
    return Permission::ACTION_FILE_READ; // 'file.read'
}
```

### 1.4 完整调用链示例 (列出目录)

```
1. 用户请求 GET /api/client/servers/{uuid}/files/list
   ↓
2. 路由匹配，进入中间件链
   ├─ ServerSubject: 设置日志上下文
   ├─ AuthenticateServerAccess: 验证服务器访问权限
   └─ ResourceBelongsToServer: 验证资源归属
   ↓
3. FormRequest 权限校验
   └─ ListFilesRequest::permission() → 'file.read'
   ↓
4. FileController::directory() 执行
   └─ DaemonFileRepository::getDirectory()
      ↓
5. Guzzle HTTP 请求到 Wings
   └─ GET /api/servers/{uuid}/files/list-directory
      ↓
6. Wings 访问容器文件系统，返回 JSON
   ↓
7. Fractal 转换器转换数据格式
   ↓
8. 返回响应给用户
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

**关键点**:
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

这是 Panel 与 Wings 文件系统通信的核心类，封装了所有文件操作：

| 方法 | Wings API 端点 | 所需权限 | 功能 |
|------|---------------|----------|------|
| `getContent()` | `GET /api/servers/{uuid}/files/contents` | `file.read-content` | 读取文件内容 |
| `putContent()` | `POST /api/servers/{uuid}/files/write` | `file.update` | 写入文件内容 |
| `getDirectory()` | `GET /api/servers/{uuid}/files/list-directory` | `file.read` | 列出目录内容 |
| `createDirectory()` | `POST /api/servers/{uuid}/files/create-directory` | `file.create` | 创建目录 |
| `renameFiles()` | `PUT /api/servers/{uuid}/files/rename` | `file.update` | 重命名/移动文件 |
| `copyFile()` | `POST /api/servers/{uuid}/files/copy` | `file.create` | 复制文件 |
| `deleteFiles()` | `POST /api/servers/{uuid}/files/delete` | `file.delete` | 删除文件/目录 |
| `compressFiles()` | `POST /api/servers/{uuid}/files/compress` | `file.archive` | 压缩文件 |
| `decompressFile()` | `POST /api/servers/{uuid}/files/decompress` | `file.archive` | 解压文件 |
| `chmodFiles()` | `POST /api/servers/{uuid}/files/chmod` | `file.update` | 修改文件权限 |
| `pull()` | `POST /api/servers/{uuid}/files/pull` | `file.create` | 从 URL 拉取文件 |

### 3.2 FileController 控制器

**文件**: `app/Http/Controllers/Api/Client/Servers/FileController.php:26-265`

控制器负责：
1. 接收用户请求
2. FormRequest 已完成权限校验
3. 调用 `DaemonFileRepository` 转发请求
4. 记录操作日志 (Activity Log)
5. 返回转换后的数据

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

### 3.3 文件大小限制

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
    // jti: 基于传入标识的哈希值，可重复生成
    $identifier = hash($algo, $identifiedBy);
    
    $config = Configuration::forSymmetricSigner(
        new Sha256(), 
        InMemory::plainText($node->getDecryptedKey())
    );

    $builder = $config->builder(new TimestampDates())
        ->issuedBy(config('app.url'))                    // iss: Panel URL
        ->permittedFor($node->getConnectionAddress())     // aud: 节点地址
        ->identifiedBy($identifier)                       // jti: 可重复的标识
        ->withHeader('jti', $identifier)                  // 额外在 header 中设置
        ->issuedAt(CarbonImmutable::now())                // iat: 签发时间
        ->canOnlyBeUsedAfter(CarbonImmutable::now()->subMinutes(5)); // nbf: 5分钟前

    // ... 添加自定义 claims

    return $builder
        ->withClaim('unique_id', Str::random())           // unique_id: 完全随机
        ->getToken($config->signer(), $config->signingKey());
}
```

### 4.2 jti 与 unique_id 的语义边界

| 字段 | 来源 | 语义 | 用途 |
|------|------|------|------|
| **jti** | `hash($algo, $identifiedBy)` | 基于用户ID和服务器UUID的可重复哈希值 | JWT 标准声明，用于标识令牌主体，相同用户+服务器组合会生成相同的 jti |
| **unique_id** | `Str::random()` | 完全随机的字符串 | 额外的随机性保证，确保即使 jti 相同，令牌也不会完全一致 |

**关键结论**：
- jti **不是** "单次使用" 标识，相同输入会生成相同 jti
- unique_id **不是** JWT 标准声明，是 Pterodactyl 额外添加的随机字段
- 两者结合提供了"可识别主体" + "随机唯一"的双重特性
- 代码中**没有**任何基于 jti 或 unique_id 的"令牌已使用"校验逻辑

### 4.3 JWT 载荷完整结构

标准 JWT 声明：
- `iss`: Panel URL (签发者)
- `aud`: 节点连接地址 (接收者)
- `jti`: 基于输入的哈希标识
- `iat`: 签发时间
- `nbf`: 最早使用时间 (签发前5分钟)
- `exp`: 过期时间 (通常15分钟)

自定义 claims：
- `user_uuid`: 用户 UUID
- `user_id`: 用户 ID (已弃用，兼容旧版 Wings)
- `server_uuid`: 服务器 UUID (仅上传时)
- `file_path`: 文件路径 (仅下载时)
- `unique_id`: 随机字符串

---

## 五、文件上传真实机制：非分片，直连 Wings

### 5.1 上传流程架构

**重要更正**：文件上传**不存在**大文件分片实现。当前实现为 **"前端直连 Wings 的单段上传"**：

```
1. 前端请求 Panel 获取签名上传 URL  (GET /api/client/servers/{uuid}/files/upload)
   ↓
2. Panel 生成 JWT 签名的 Wings URL
   ├─ 权限校验: UploadFileRequest::permission() → 'file.create'
   └─ JWT 包含: server_uuid, user_uuid, 15分钟过期
   ↓
3. 前端直接 POST 文件到 Wings  (multipart/form-data)
   ├─ URL: {node_address}/upload/file?token={jwt}
   ├─ 参数: directory (目标目录)
   └─ Body: files (文件内容)
   ↓
4. Wings 验证 JWT 并写入容器文件系统
   ↓
5. 前端通过 axios 回调显示上传进度
```

### 5.2 步骤详解

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
                            params: { directory },  // 目标目录
                            onUploadProgress: (data) => onUploadProgress(data, file.name),
                        }
                    )
                    .then(() => timeouts.value.push(setTimeout(() => removeFileUpload(file.name), 500)))
            );
    });

    Promise.all(uploads.map((fn) => fn()))
        .then(() => mutate())  // 刷新文件列表
        .catch((error) => {
            clearFileUploads();  // ⚠️ 失败时清空所有上传进度
            clearAndAddHttpError(error);
        });
};
```

### 5.3 上传进度反馈

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
    
    pushFileUpload: Action<...>;
    setUploadProgress: Action<...>;
    clearFileUploads: Action<...>;
    removeFileUpload: Action<...>;
    cancelFileUpload: Action<...>;
}
```

**进度更新回调**:
**文件**: `resources/scripts/components/server/files/UploadButton.tsx:60-62`

```typescript
const onUploadProgress = (data: AxiosProgressEvent, name: string) => {
    setUploadProgress({ name, loaded: data.loaded });
};
```

**进度计算**:
- `total`: 文件总大小 (来自 `file.size`)
- `loaded`: 已上传字节数 (来自 axios `progressEvent.loaded`)
- 进度百分比: `(loaded / total) * 100`

### 5.4 上传失败时的进度清空影响

**文件**: `resources/scripts/components/server/files/UploadButton.tsx:95-100`

```typescript
Promise.all(uploads.map((fn) => fn()))
    .then(() => mutate())
    .catch((error) => {
        clearFileUploads();  // 清空所有上传状态
        clearAndAddHttpError(error);
    });
```

**对反馈语义的影响**:

1. **全部或全无**：只要有一个文件上传失败，所有文件的进度状态都会被清除
2. **用户无法获知部分成功**：多文件上传时，用户无法知道哪些文件已成功上传、哪些失败
3. **进度丢失**：已上传的部分进度信息完全丢失，用户体验上表现为"突然消失"
4. **重试成本**：用户需要重新选择所有文件并从头开始上传

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

### 5.5 上传取消机制

```typescript
cancelFileUpload: action((state, payload) => {
    if (state.uploads[payload]) {
        state.uploads[payload].abort.abort();  // 调用 AbortController.abort()
        delete state.uploads[payload];
    }
}),
```

---

## 六、对比：备份多段上传链路

**重要区分**：只有备份上传才实现了真正的多段 (multipart) 上传。

### 6.1 备份多段上传架构

**文件**: `app/Http/Controllers/Api/Remote/Backups/BackupRemoteUploadController.php:16-134`

```
Wings 请求 Panel 获取多段上传信息
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

### 6.2 文件上传 vs 备份上传 对比

| 特性 | 文件管理器上传 | 备份 S3 多段上传 |
|------|--------------|-----------------|
| **实现方式** | 单段 multipart/form-data | S3 Multipart Upload API |
| **分片支持** | ❌ 无 | ✅ 有 (默认 5GB/片) |
| **数据流向** | 前端 → Wings | Wings → S3 (通过 presigned URL) |
| **进度反馈** | axios onUploadProgress | 由 Wings 内部处理 |
| **端点位置** | Wings `/upload/file` | S3 各分片 URL |
| **认证方式** | JWT in query string | S3 presigned URL |
| **Panel 角色** | 生成签名 URL | 生成 S3 presigned URL 列表 |

---

## 七、文件下载流程

**文件**: `app/Http/Controllers/Api/Client/Servers/FileController.php:77-100`

下载同样采用 JWT 签名直连模式：

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

## 八、设计亮点与权衡

### 8.1 优点

1. **流量卸载**: 大文件上传/下载直接走 Wings，不占用 Panel 带宽
2. **分层权限**: 中间件粗粒度校验 + FormRequest 细粒度权限检查
3. **实时进度**: 前端直连可获得精确的上传进度
4. **可取消**: 基于 `AbortController` 支持上传取消
5. **超时保护**: 压缩/解压操作设置 15 分钟长超时

### 8.2 权衡点

1. **跨域问题**: 前端直接请求 Wings 需要 CORS 配置
2. **Token 时效**: 15 分钟过期，超大文件上传可能超时
3. **单文件签名**: 每个文件单独请求签名 URL，可优化批量签名
4. **失败处理粗糙**: 上传失败时清空所有进度，用户体验不佳
5. **无断点续传**: 非分片上传，失败后需从头开始

### 8.3 安全机制

- **节点密钥认证**: Panel → Wings 内部通信使用对称密钥
- **JWT 用户认证**: 上传/下载时验证用户身份和操作权限
- **时间窗口**: `nbf` (not before) 防止令牌重放攻击
- **随机因子**: `unique_id` 增加令牌不可预测性
- **权限分层**: 中间件 + FormRequest + Laravel Gate 三层校验

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
| 客户端 API 请求基类 | `app/Http/Requests/Api/Client/ClientApiRequest.php` |
| JWT 签名服务 | `app/Services/Nodes/NodeJWTService.php` |
| 前端上传组件 | `resources/scripts/components/server/files/UploadButton.tsx` |
| 前端上传状态 | `resources/scripts/state/server/files.ts` |
| 节点模型 | `app/Models/Node.php` |
| API 路由定义 | `routes/api-client.php` |
