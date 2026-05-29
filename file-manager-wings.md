# Pterodactyl Panel 文件管理器与 Wings 同步实现分析

## 概述

Pterodactyl Panel 的文件管理器采用 **"Panel 做认证授权，Wings 做实际文件操作"** 的架构模式。所有文件操作请求最终都由 Wings 守护进程直接执行，Panel 主要负责：
- 用户身份认证与权限校验
- 请求转发与 JWT 签名
- API 接口聚合与数据转换

---

## 一、请求转发机制

### 1.1 核心架构

```
用户浏览器 → Panel API (认证) → Wings Daemon (实际操作) → 容器文件系统
```

### 1.2 DaemonRepository 基类

**文件**: `app/Repositories/Wings/DaemonRepository.php:11-65`

所有 Wings 通信的基类，提供统一的 HTTP 客户端配置：

```php
abstract class DaemonRepository
{
    protected ?Server $server;
    protected ?Node $node;

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

### 1.3 节点连接地址

**文件**: `app/Models/Node.php:134-137`

```php
public function getConnectionAddress(): string
{
    return sprintf('%s://%s:%s', $this->scheme, $this->fqdn, $this->daemonListen);
}
```

---

## 二、容器文件系统访问

### 2.1 DaemonFileRepository 文件操作仓库

**文件**: `app/Repositories/Wings/DaemonFileRepository.php:18-301`

这是 Panel 与 Wings 文件系统通信的核心类，封装了所有文件操作：

| 方法 | Wings API 端点 | 功能 |
|------|---------------|------|
| `getContent()` | `GET /api/servers/{uuid}/files/contents` | 读取文件内容 |
| `putContent()` | `POST /api/servers/{uuid}/files/write` | 写入文件内容 |
| `getDirectory()` | `GET /api/servers/{uuid}/files/list-directory` | 列出目录内容 |
| `createDirectory()` | `POST /api/servers/{uuid}/files/create-directory` | 创建目录 |
| `renameFiles()` | `PUT /api/servers/{uuid}/files/rename` | 重命名/移动文件 |
| `copyFile()` | `POST /api/servers/{uuid}/files/copy` | 复制文件 |
| `deleteFiles()` | `POST /api/servers/{uuid}/files/delete` | 删除文件/目录 |
| `compressFiles()` | `POST /api/servers/{uuid}/files/compress` | 压缩文件 |
| `decompressFile()` | `POST /api/servers/{uuid}/files/decompress` | 解压文件 |
| `chmodFiles()` | `POST /api/servers/{uuid}/files/chmod` | 修改文件权限 |
| `pull()` | `POST /api/servers/{uuid}/files/pull` | 从 URL 拉取文件 |

### 2.2 FileController 控制器

**文件**: `app/Http/Controllers/Api/Client/Servers/FileController.php:26-265`

控制器负责：
1. 接收用户请求
2. 验证用户权限（通过 FormRequest）
3. 调用 `DaemonFileRepository` 转发请求
4. 记录操作日志
5. 返回转换后的数据

**典型调用链示例 (读取目录)**:

```
用户请求 GET /api/client/servers/{uuid}/files/list
        ↓
FileController::directory() [Line 43-52]
        ↓
DaemonFileRepository::getDirectory() [Line 80-96]
        ↓
Wings API GET /api/servers/{uuid}/files/list-directory
        ↓
Wings 访问容器文件系统
        ↓
返回 JSON 数据 → Fractal 转换 → 用户
```

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

### 2.3 文件大小限制

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

## 三、JWT 认证机制

### 3.1 NodeJWTService 服务

**文件**: `app/Services/Nodes/NodeJWTService.php:15-103`

用于生成 Wings 可验证的 JWT 令牌，主要用于文件下载和上传的授权。

```php
public function handle(Node $node, ?string $identifiedBy, string $algo = 'md5'): UnencryptedToken
{
    $identifier = hash($algo, $identifiedBy);
    $config = Configuration::forSymmetricSigner(
        new Sha256(), 
        InMemory::plainText($node->getDecryptedKey())
    );

    $builder = $config->builder(new TimestampDates())
        ->issuedBy(config('app.url'))
        ->permittedFor($node->getConnectionAddress())
        ->identifiedBy($identifier)
        ->issuedAt(CarbonImmutable::now())
        ->canOnlyBeUsedAfter(CarbonImmutable::now()->subMinutes(5));

    // 添加自定义 claims...
    foreach ($this->claims as $key => $value) {
        $builder = $builder->withClaim($key, $value);
    }

    return $builder
        ->withClaim('unique_id', Str::random())
        ->getToken($config->signer(), $config->signingKey());
}
```

**JWT 载荷包含**:
- `iss`: Panel URL (签发者)
- `aud`: 节点连接地址 (接收者)
- `jti`: 唯一标识符
- `iat`: 签发时间
- `nbf`: 最早使用时间 (签发前5分钟)
- `exp`: 过期时间 (通常15分钟)
- `user_uuid`: 用户 UUID
- `server_uuid`: 服务器 UUID
- `file_path`: 文件路径 (仅下载时)
- `unique_id`: 随机字符串

---

## 四、大文件上传实现

### 4.1 上传流程架构

与常规文件操作不同，**文件上传采用"直连 Wings"模式**，不经过 Panel 中转：

```
1. 前端请求 Panel 获取签名上传 URL
   ↓
2. Panel 生成 JWT 签名的 Wings URL
   ↓
3. 前端直接上传文件到 Wings
   ↓
4. Wings 验证 JWT 并写入容器文件系统
   ↓
5. 前端实时显示上传进度
```

### 4.2 步骤详解

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
    // ...
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

### 4.3 上传进度反馈

**状态管理**:
**文件**: `resources/scripts/state/server/files.ts:4-79`

```typescript
export interface FileUploadData {
    loaded: number;
    readonly abort: AbortController;
    readonly total: number;
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

### 4.4 上传取消机制

```typescript
cancelFileUpload: action((state, payload) => {
    if (state.uploads[payload]) {
        state.uploads[payload].abort.abort();  // 调用 AbortController.abort()
        delete state.uploads[payload];
    }
}),
```

---

## 五、文件下载流程

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

## 六、设计亮点与权衡

### 6.1 优点

1. **流量卸载**: 大文件上传/下载直接走 Wings，不占用 Panel 带宽
2. **统一认证**: JWT + 节点密钥双重认证，安全可靠
3. **实时进度**: 前端直连可获得精确的上传进度
4. **可取消**: 基于 `AbortController` 支持上传取消
5. **超时保护**: 压缩/解压操作设置 15 分钟长超时

### 6.2 权衡点

1. **跨域问题**: 前端直接请求 Wings 需要 CORS 配置
2. **Token 时效**: 15 分钟过期，超大文件上传可能超时
3. **单文件上传**: 目前每个文件单独请求签名 URL，可优化批量签名

### 6.3 安全机制

- **节点密钥认证**: Panel → Wings 内部通信使用对称密钥
- **JWT 用户认证**: 上传/下载时验证用户身份和操作权限
- **时间窗口**: `nbf` (not before) 防止令牌重放攻击
- **唯一标识**: `jti` + `unique_id` 确保单次使用

---

## 七、关键代码位置速查表

| 功能 | 文件路径 |
|------|---------|
| Wings 文件操作仓库 | `app/Repositories/Wings/DaemonFileRepository.php` |
| Wings 通信基类 | `app/Repositories/Wings/DaemonRepository.php` |
| 文件 API 控制器 | `app/Http/Controllers/Api/Client/Servers/FileController.php` |
| 文件上传控制器 | `app/Http/Controllers/Api/Client/Servers/FileUploadController.php` |
| JWT 签名服务 | `app/Services/Nodes/NodeJWTService.php` |
| 前端上传组件 | `resources/scripts/components/server/files/UploadButton.tsx` |
| 前端上传状态 | `resources/scripts/state/server/files.ts` |
| 节点模型 | `app/Models/Node.php` |
