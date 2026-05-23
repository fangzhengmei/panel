# 备份与恢复任务链路分析

## 一、整体架构概览

Pterodactyl 面板采用"面板（Panel）+ 守护进程（Wings）"的分布式架构。备份与恢复任务的执行链路如下：

```
用户操作 → Panel API → 业务校验 → Wings 通信 → Wings 执行 → 回调 Panel → 状态更新
```

**核心组件：**
- 面板端：业务校验、数据持久化、Wings 通信、状态管理
- Wings 端：实际执行备份打包、上传、解压恢复
- 存储层：本地 Wings 存储 或 S3 兼容云存储

---

## 二、备份任务完整链路

### 2.1 API 入口

**路由**：`POST /api/client/servers/{server}/backups`
**控制器**：`app/Http/Controllers/Api/Client/Servers/BackupController.php:67`

```php
public function store(StoreBackupRequest $request, Server $server): array
```

### 2.2 四层限流/校验机制

#### 第零层：api.client 全局限流（创建与恢复共用）

> ⚠️ **全局共用限流**：此限流作用于所有 `/api/client` 路由，创建备份和恢复备份共用。

**配置落点**：`app/Providers/RouteServiceProvider.php:56`
```php
Route::middleware(['client-api', 'throttle:api.client'])
    ->prefix('/api/client')
    ->group(base_path('routes/api-client.php'));
```

**限流规则**：`config/http.php:14-16`
```php
'rate_limit' => [
    'client_period' => 1,          // 周期：1 分钟
    'client' => 256,                // 次数：256 次/分钟
],
```

**限流 Key**：`app/Providers/RouteServiceProvider.php:93-94`
```php
$key = optional($request->user())->uuid ?: $request->ip();
```
> 按用户 UUID 限流（未登录时按 IP），避免用户通过切换 IP 绕过限制。

**排障口径**：HTTP 429，响应头 `X-RateLimit-Limit: 256`、`X-RateLimit-Remaining: 0`

#### 第一层：权限与参数校验
- **权限校验**：`Permission::ACTION_BACKUP_CREATE`，通过 `StoreBackupRequest` 前置检查
- **参数校验**：
  - `name`：可选，字符串，最大 191 字符
  - `is_locked`：可选，布尔值（需同时拥有删除权限才能设置）
  - `ignored`：可选，换行分隔的忽略文件列表

**代码位置**：`app/Http/Requests/Api/Client/Servers/Backups/StoreBackupRequest.php`

#### 第二层：频率节流校验（Throttles）

> ⚠️ **限流落点说明**：此限流仅作用于**创建备份**，在 `InitiateBackupService` 内部实现，路由层面无中间件。

**配置**：`config/backups.php:29`
```php
'throttles' => [
    'limit' => env('BACKUP_THROTTLE_LIMIT', 2),    // 默认 2 次
    'period' => env('BACKUP_THROTTLE_PERIOD', 600), // 默认 600 秒（10分钟）
],
```

**校验逻辑**：`app/Services/Backups/InitiateBackupService.php:78`
```php
$previous = $this->repository->getBackupsGeneratedDuringTimespan($server->id, $period);
if ($previous->count() >= $limit) {
    throw new TooManyRequestsHttpException($retryAfter, $message);
}
```

**查询范围**：包含进行中（`completed_at is null`）和已成功（`is_successful = true`）的备份。

**排障口径**：HTTP 429，响应体含 `Retry-After` 秒数，错误消息含时间窗口信息。

#### 第三层：资源配额校验（backup_limit）

**代码位置**：`app/Services/Backups/InitiateBackupService.php:91`

```php
$successful = $this->repository->getNonFailedBackups($server);
if (!$server->backup_limit || $successful->count() >= $server->backup_limit) {
    if (!$override || $server->backup_limit <= 0) {
        throw new TooManyBackupsException($server->backup_limit);
    }
    // 尝试自动清理最旧非锁定备份...
}
```

**⚠️ backup_limit 语义修正**：
- `backup_limit > 0`：允许创建最多 N 个备份
- `backup_limit = 0`：**完全禁止创建备份**（不是无限制！）
  - 证据1：`app/Services/Backups/InitiateBackupService.php:94` 中 `$server->backup_limit <= 0` 直接抛异常
  - 证据2：`app/Http/Controllers/Api/Client/Servers/ScheduleTaskController.php:47` 禁止创建 backup_limit=0 的定时备份任务
  - 证据3：测试用例 `'A backup task cannot be created when the server\'s backup limit is set to 0.'`
  - 证据4：前端仅在 `backupLimit > 0` 时显示创建按钮

> 注意：与 memory/disk 等资源不同（0 表示无限制），backup_limit=0 是显式禁用。

### 2.3 配额超限处理

**超限策略**：`app/Services/Backups/InitiateBackupService.php:91`
1. 检查非失败备份数量是否达到上限
2. 若 `override=true` 且 `backup_limit > 0`，尝试自动清理
3. 查找最旧的**非锁定**备份自动删除
4. 若无可用备份（全部锁定），抛出 `TooManyBackupsException`

### 2.4 数据库事务与 Wings 外部调用的一致性边界

**代码位置**：`app/Services/Backups/InitiateBackupService.php:109`

```php
return $this->connection->transaction(function () use ($server, $name) {
    /** @var Backup $backup */
    $backup = $this->repository->create([
        'server_id' => $server->id,
        'uuid' => Uuid::uuid4()->toString(),
        'name' => trim($name) ?: sprintf('Backup at %s', CarbonImmutable::now()->toDateTimeString()),
        'ignored_files' => array_values($this->ignoredFiles),
        'disk' => $this->backupManager->getDefaultAdapter(),
        'is_locked' => $this->isLocked,
    ], true, true);

    // ⚠️ 外部 HTTP 调用在数据库事务内部
    $this->daemonBackupRepository->setServer($server)
        ->setBackupAdapter($this->backupManager->getDefaultAdapter())
        ->backup($backup);

    return $backup;
});
```

**⚠️ 一致性边界的准确表述**：

| 操作类型 | 是否受事务保护 | 失败时行为 |
|---------|---------------|-----------|
| `$this->repository->create()` 插入备份记录 | ✅ 受保护 | 事务回滚，记录被撤销 |
| `$this->daemonBackupRepository->backup()` 调用 Wings | ❌ 不受保护 | **Wings 动作不会回滚** |

**核心语义**：数据库事务**仅保护数据库操作**，不保护外部 HTTP 调用。Wings 的备份/恢复/删除都是**异步执行**，HTTP 请求只是"触发"，一旦请求发出（哪怕后续网络超时），Wings 侧的动作已开始，无法通过数据库回滚来撤销。

### 2.5 Wings 通信层

**基类**：`app/Repositories/Wings/DaemonRepository.php`

**HTTP 客户端配置**：
- Base URI：`{scheme}://{fqdn}:{daemonListen}`
- 超时：`config('pterodactyl.guzzle.timeout')`
- TLS 验证：生产环境强制开启
- 认证头：`Authorization: Bearer {node.getDecryptedKey()}`

**备份请求**：`app/Repositories/Wings/DaemonBackupRepository.php:35`
```php
POST /api/servers/{server_uuid}/backup
Body: {
    "adapter": "wings|s3",
    "uuid": "backup-uuid",
    "ignore": "file1\nfile2\n"
}
```

**异常传播**：Guzzle `TransferException` → 包装为 `DaemonConnectionException`（HTTP 502/504 等）

### 2.6 典型失败场景（数据库回滚但不回滚远端动作）

| 场景 | 时序 | 结果 | 排障建议 |
|-----|------|------|---------|
| **Wings 已接收但 HTTP 超时** | 1. 事务内 INSERT 备份记录成功<br>2. Wings 收到请求，已开始打包<br>3. HTTP 超时，抛出 `DaemonConnectionException`<br>4. 事务回滚，记录被撤销 | ✅ 数据库回滚<br>❌ Wings 仍在后台打包 → **孤儿备份任务** | 去节点侧 `daemon.log` 查 `X-Request-Id`，看 Wings 是否实际执行了备份 |
| **Wings 已创建任务但返回非 200** | 1. 事务内 INSERT 成功<br>2. Wings 收到请求，创建任务<br>3. Wings 返回 400/500 错误<br>4. 事务回滚 | ✅ 数据库回滚<br>❌ Wings 任务已创建，可能仍在执行 | 检查 Wings 侧是否已有该备份 UUID 的任务记录 |
| **TCP 连接已建立但包丢失** | 1. 事务内 INSERT 成功<br>2. HTTP 请求已发出（Wings 已收到）<br>3. 响应包丢失，Panel 侧读超时<br>4. 事务回滚 | ✅ 数据库回滚<br>❌ Wings 已收到并执行 | 最隐蔽的不一致场景，需依赖孤儿备份清理 |

> **排障总原则**：用户反馈"创建备份失败但节点上有备份进程在跑"时，首先排查网络超时问题，不要盲目重试。`p:maintenance:prune-backups` 命令可以兜底清理这类孤儿备份的面板记录，但无法终止 Wings 侧已启动的进程。

### 2.7 加密与安全机制

#### 节点密钥加密存储
- 数据库存储：`Node.daemon_token` 使用 Laravel 加密器加密
- 解密使用：`Container::getInstance()->make(Encrypter::class)->decrypt($token)`
- 密钥格式：`{daemon_token_id}.{daemon_token}`

#### 传输加密
- 协议：HTTPS（通过 `Node.scheme` 配置 `http`/`https`）
- 生产环境强制证书验证：`verify => $this->app->environment('production')`

#### 回调认证（Wings → Panel）
**中间件**：`app/Http/Middleware/Api/Daemon/DaemonAuthenticate.php`

```php
// Bearer Token 格式：{token_id}.{token}
$parts = explode('.', $bearer);
$node = $this->repository->findFirstWhere(['daemon_token_id' => $parts[0]]);
if (hash_equals($this->encrypter->decrypt($node->daemon_token), $parts[1])) {
    $request->attributes->set('node', $node);
    return $next($request);
}
```

### 2.8 S3 分片上传完整时序

**⚠️ 时序修正**：S3 分片上传发生在 Wings 打包完成后、完成回调之前。

```
1. Panel 创建备份记录 → 通知 Wings 开始备份
2. Wings 本地打包文件
3. Wings → Panel: GET /api/remote/backups/{backup}?size={bytes}
   └─ 控制器：app/Http/Controllers/Api/Remote/Backups/BackupRemoteUploadController.php:34
4. Panel 调用 S3 CreateMultipartUpload 初始化
5. Panel 生成多个 UploadPart 预签名 URL
6. Panel 保存 upload_id 到备份记录
7. Panel 返回 { parts: [url1, url2, ...], part_size: 5GB } 给 Wings
8. Wings 使用预签名 URL 逐个上传分片到 S3
9. Wings → Panel: POST /api/remote/backups/{backup} (完成回调)
   └─ 控制器：app/Http/Controllers/Api/Remote/Backups/BackupStatusController.php:32
10. Panel 更新备份状态 → CompleteMultipartUpload (成功) / AbortMultipartUpload (失败)
```

**接口**：`GET /api/remote/backups/{backup}?size={bytes}`
**控制器**：`app/Http/Controllers/Api/Remote/Backups/BackupRemoteUploadController.php:34`

**预签名 URL 有效期**：`config('backups.presigned_url_lifespan', 60)` 分钟
**默认分片大小**：`BACKUP_MAX_PART_SIZE = 5GB`（AWS S3 单分片上限）

### 2.9 备份完成回调

**接口**：`POST /api/remote/backups/{backup}`
**控制器**：`app/Http/Controllers/Api/Remote/Backups/BackupStatusController.php:32`

**请求校验**：`app/Http/Requests/Api/Remote/ReportBackupCompleteRequest.php`
```php
[
    'successful' => 'required|boolean',
    'checksum' => 'nullable|string|required_if:successful,true',
    'checksum_type' => 'nullable|string|required_if:successful,true',
    'size' => 'nullable|numeric|required_if:successful,true',
    'parts' => 'nullable|array',
    'parts.*.etag' => 'required|string',
    'parts.*.part_number' => 'required|numeric',
]
```

**业务逻辑**：
1. 节点归属校验：`$server->node_id !== $node->id` 抛出 403
2. 幂等校验：已成功的备份不可重复更新
3. 成功分支：
   - `is_successful = true`
   - `checksum = {type}:{hash}`
   - `bytes = {size}`
   - `completed_at = now()`
   - S3 适配器：`CompleteMultipartUpload` 合并分片
4. 失败分支：
   - `is_successful = false`
   - `is_locked = false`（自动解锁，便于删除）
   - `checksum = null`
   - `bytes = 0`
   - `completed_at = now()`
   - S3 适配器：`AbortMultipartUpload` 中止分片

### 2.10 S3 分片合并异常分支

**代码位置**：`app/Http/Controllers/Api/Remote/Backups/BackupStatusController.php:120`

```php
protected function completeMultipartUpload(Backup $backup, S3Filesystem $adapter, bool $successful, ?array $parts): void
{
    if (empty($backup->upload_id)) {
        if (!$successful) {
            return; // ⚠️ 上传开始前就失败，静默返回，不抛错
        }
        throw new DisplayException('Cannot complete backup request: no upload_id present on model.');
    }

    if (!$successful) {
        $client->execute($client->getCommand('AbortMultipartUpload', $params));
        return;
    }

    if (is_null($parts)) {
        // ⚠️ Wings 未传 parts 时，自动调用 ListParts 查询已上传分片
        $params['MultipartUpload']['Parts'] = $client->execute($client->getCommand('ListParts', $params))['Parts'];
    } else {
        // 使用 Wings 传入的 parts 列表
        foreach ($parts as $part) {
            $params['MultipartUpload']['Parts'][] = [
                'ETag' => $part['etag'],
                'PartNumber' => $part['part_number'],
            ];
        }
    }

    $client->execute($client->getCommand('CompleteMultipartUpload', $params));
}
```

**异常分支矩阵**：

| 场景 | upload_id | successful | parts | 行为 |
|-----|-----------|------------|-------|------|
| 上传前失败 | 空 | false | - | 静默 return |
| 上传中失败 | 非空 | false | - | AbortMultipartUpload |
| 上传成功但 upload_id 丢失 | 空 | true | - | 抛 DisplayException |
| 上传成功，Wings 未传 parts | 非空 | true | null | 自动 ListParts 查询 |
| 上传成功，Wings 传 parts | 非空 | true | 数组 | 使用传入 parts 合并 |

---

## 三、恢复任务完整链路

### 3.1 API 入口

**路由**：`POST /api/client/servers/{server}/backups/{backup}/restore`
**控制器**：`app/Http/Controllers/Api/Client/Servers/BackupController.php:198`

### 3.2 四层限流/校验机制

#### 第零层：api.client 全局限流（与创建共用）

同备份创建，见 2.2 节。所有 `/api/client` 路由共用此限流。

#### 第一层：ResourceLimit 路由限流（仅恢复）

> ⚠️ **限流落点说明**：恢复备份的限流与创建备份完全不同。
>
> - **路由层面**：有 `ResourceLimit::Backup->middleware()`（见 `routes/api-client.php:138`）
>   - 限流规则：每 15 分钟最多 3 次，按服务器维度（`app/Enum/ResourceLimit.php:46`）
> - **业务层面**：无 throttles 检查

**限流 Key**：`app/Enum/ResourceLimit.php:30`
```php
public function throttleKey(): string
{
    return mb_strtolower("api.client:server-resource:{$this->name}");
}
// 实际按 server.uuid 限流：return $case->limit()->by($server->uuid);
```

**排障口径**：HTTP 429，响应头 `X-RateLimit-Limit: 3`，限流 Key 为服务器 UUID。

#### 第二层：服务器状态校验（可发起恢复边界）

**代码位置**：`app/Http/Controllers/Api/Client/Servers/BackupController.php:200-208`

```php
// 校验1：服务器状态必须为 null（空闲状态）
if (!is_null($server->status)) {
    throw new BadRequestHttpException('This server is not currently in a state that allows for a backup to be restored.');
}

// 校验2：备份状态校验 ⚠️ 注意此处逻辑
if (!$backup->is_successful && is_null($backup->completed_at)) {
    throw new BadRequestHttpException('This backup cannot be restored at this time: not completed or failed.');
}
```

**⚠️ 状态流转边界 1："可发起恢复"的完整条件**

"可发起恢复"需同时通过三层检查：

| 检查层级 | 代码位置 | 检查内容 | 通过条件 |
|---------|---------|---------|---------|
| 路由中间件 | `app/Http/Middleware/Api/Client/Server/AuthenticateServerAccess.php:50` | `validateCurrentState()` | 未暂停、节点未维护、已安装、非恢复中、无转移任务 |
| 控制器校验1 | `app/Http/Controllers/Api/Client/Servers/BackupController.php:202` | 服务器状态 | `$server->status === null` |
| 控制器校验2 | `app/Http/Controllers/Api/Client/Servers/BackupController.php:206` | 备份状态 | 非"进行中且未成功"状态 |

**备份状态真值表**：

| is_successful | completed_at | 条件结果 | 是否允许恢复 | 说明 |
|---------------|--------------|----------|-------------|------|
| true | 非空 | false | ✅ 允许 | 成功完成的备份 |
| false | 非空 | false | ✅ 允许 | 已明确失败的备份（代码未禁止） |
| false | null | true | ❌ 禁止 | 进行中且未成功的备份 |

> 重要结论：**已失败的备份（is_successful=false 但 completed_at 有值）并未被禁止恢复**。只有"进行中且未成功"的备份被禁止。

**其他校验**：
- 权限校验：`Permission::ACTION_BACKUP_RESTORE`
- 参数校验：`truncate` 布尔值（是否清空目录后恢复）

### 3.3 恢复执行流程与事务边界

**代码位置**：`app/Http/Controllers/Api/Client/Servers/BackupController.php:214`

```php
$log->transaction(function () use ($backup, $server, $request) {
    // If the backup is for an S3 file we need to generate a unique Download link for
    // it that will allow Wings to actually access the file.
    if ($backup->disk === Backup::ADAPTER_AWS_S3) {
        $url = $this->downloadLinkService->handle($backup, $request->user()); // 外部调用1：S3 预签名
    }

    // Update the status right away for the server so that we know not to allow certain
    // actions against it via the Panel API.
    $server->update(['status' => Server::STATUS_RESTORING_BACKUP]); // 数据库操作

    $this->daemonRepository->setServer($server)->restore($backup, $url ?? null, $request->input('truncate')); // 外部调用2：Wings HTTP
});
```

**⚠️ 恢复流程的事务边界分析**：

| 步骤 | 操作类型 | 是否受事务保护 | 失败时行为 |
|-----|---------|---------------|-----------|
| 1 | S3 预签名 URL 生成 | ❌ 外部调用 | 直接抛异常，事务尚未执行任何数据库操作，无不一致 |
| 2 | `$server->update(['status' => ...])` | ✅ 数据库操作 | 事务回滚，status 还原为 null |
| 3 | `$this->daemonRepository->restore()` | ❌ 外部调用 | **Wings 动作不会回滚** |

**恢复场景下的不一致风险**：

| 场景 | 结果 | 用户可见现象 |
|-----|------|-------------|
| Wings 已开始恢复但 HTTP 超时，事务回滚 | ✅ status 还原为 null<br>❌ Wings 仍在后台恢复 | 服务器状态显示"正常"，但实际在恢复中，可能出现数据不一致 |
| S3 预签名 URL 已生成但后续步骤失败 | ✅ 数据库回滚<br>❌ S3 URL 已签发（5 分钟后自动过期） | 无业务影响，仅浪费一个预签名 URL |

> **排障建议**：用户反馈"恢复失败但服务器数据好像被改了"时，检查 Wings 侧是否有正在进行的恢复任务。由于状态已回滚，用户可能重复发起恢复，导致多次解压覆盖。

### 3.4 Wings 恢复请求

**代码位置**：`app/Repositories/Wings/DaemonBackupRepository.php:60`
```php
POST /api/servers/{server_uuid}/backup/{backup_uuid}/restore
Body: {
    "adapter": "wings|s3",
    "truncate_directory": true|false,
    "download_url": "..."  // S3 备份时提供
}
```

### 3.5 下载链接生成

**服务**：`app/Services/Backups/DownloadLinkService.php:24`

- **Wings 本地备份**：生成 JWT 签名的下载链接，有效期 15 分钟
  ```php
  $token = $this->jwtService
      ->setExpiresAt(CarbonImmutable::now()->addMinutes(15))
      ->setUser($user)
      ->setClaims([
          'backup_uuid' => $backup->uuid,
          'server_uuid' => $backup->server->uuid,
      ])
      ->handle($backup->server->node, $user->id . $backup->server->uuid);
  ```

- **S3 备份**：生成 S3 预签名 URL，有效期 5 分钟

### 3.6 JWT 签名机制

**服务**：`app/Services/Nodes/NodeJWTService.php:63`

- 算法：HS256（HMAC-SHA256）
- 密钥：节点解密后的 `daemon_token`
- 声明（Claims）：
  - `iss`：面板 URL
  - `aud`：节点连接地址
  - `jti`：唯一标识（md5 哈希）
  - `iat`：签发时间
  - `nbf`：5 分钟前（防止时钟偏差）
  - `exp`：过期时间
  - 自定义：`backup_uuid`、`server_uuid`、`user_uuid`、`unique_id`

### 3.7 恢复完成回调

**接口**：`POST /api/remote/backups/{backup}/restore`
**控制器**：`app/Http/Controllers/Api/Remote/Backups/BackupStatusController.php:93`

**⚠️ 状态流转边界 2："恢复成功"的处理逻辑**

```php
public function restore(Request $request, string $backup): JsonResponse
{
    // 节点归属校验...
    
    // ⚠️ 无论成功失败，都清除服务器状态
    $model->server->update(['status' => null]);

    // ⚠️ successful 字段仅用于审计事件名称，无业务分支
    Activity::event($request->boolean('successful') 
        ? 'server:backup.restore-complete' 
        : 'server:backup.restore-failed')
        ->subject($model, $model->server)
        ->property('name', $model->name)
        ->log();

    return new JsonResponse([], JsonResponse::HTTP_NO_CONTENT);
}
```

**⚠️ successful 字段语义修正**：

| 用途 | 是否使用 successful | 说明 |
|-----|---------------------|------|
| 服务器状态更新 | ❌ 不使用 | 无论成功失败，都 `update(['status' => null])` |
| 业务逻辑分支 | ❌ 不使用 | 无任何 `if ($successful)` 业务分支 |
| 审计事件名称 | ✅ 使用 | 仅决定事件名是 `restore-complete` 还是 `restore-failed` |

> 设计要点：无论恢复成功或失败，都清除服务器状态，避免陷入不可用状态。用户可重试恢复或使用重装功能。成功/失败仅记录在审计日志中。

---

## 四、限流架构全景

### 4.1 三层限流并存关系（排障核心口径）

> ⚠️ **关键结论**：三层限流**并存生效**，独立计数，互不影响。任一限流触发都会拦截请求。

| 限流类型 | 技术实现 | 计数 Key | 创建备份 | 恢复备份 | 响应特征 |
|---------|---------|----------|---------|---------|---------|
| api.client 全局 | `RateLimiter::for('api.client')` | 用户 UUID / IP | ✅ | ✅ | `X-RateLimit-Limit: 256` |
| throttles 业务 | 业务代码内 `getBackupsGeneratedDuringTimespan()` | 服务器 ID | ✅ | ❌ | 429 + `Retry-After` 秒数 |
| ResourceLimit 路由 | `RateLimiter::for('api.client:server-resource:backup')` | 服务器 UUID | ❌ | ✅ | `X-RateLimit-Limit: 3` |

### 4.2 限流技术实现细节

**api.client 与 ResourceLimit 是两个独立的 Laravel RateLimiter 实例**：

```php
// api.client - 按用户限流
RateLimiter::for('api.client', function (Request $request) {
    $key = optional($request->user())->uuid ?: $request->ip();
    return Limit::perMinutes(1, 256)->by($key);
});

// ResourceLimit::Backup - 按服务器限流
RateLimiter::for('api.client:server-resource:backup', function (Request $request) {
    $server = $request->route()->parameter('server');
    return Limit::perMinutes(15, 3)->by($server->uuid);
});
```

> **排障指导**：用户报"限流"时，先看响应头。
> - 只有 `X-RateLimit-Limit: 256` → 触发了用户级全局限流
> - 有 `X-RateLimit-Limit: 3` → 触发了服务器级恢复限流
> - 没有 `X-RateLimit` 头但有 `Retry-After` → 触发了备份创建的 throttles 业务限流

### 4.3 限流执行顺序

**创建备份**：
```
1. api.client 全局限流 (256/分钟)   ← 独立计数1
   ↓
2. 权限/参数校验
   ↓
3. throttles 业务限流 (2次/10分钟)   ← 独立计数2
   ↓
4. backup_limit 配额校验
```

**恢复备份**：
```
1. api.client 全局限流 (256/分钟)   ← 独立计数1
   ↓
2. ResourceLimit 路由限流 (3次/15分钟)  ← 独立计数2
   ↓
3. 权限/参数校验
   ↓
4. 服务器状态校验 (可发起恢复边界)
```

---

## 五、计费与配额管理

### 5.1 配额维度

| 配额项 | 存储位置 | 作用 | 语义 |
|--------|---------|------|------|
| `server.backup_limit` | servers 表 | 单服务器备份数量上限 | **0=禁止，>0=上限数量** |
| `backup.bytes` | backups 表 | 单个备份大小 | 用于统计总存储占用 |
| throttles 配置 | `config/backups.php` | 创建频率限制 | 10分钟2次（仅创建） |
| ResourceLimit 限流 | `app/Enum/ResourceLimit.php` | 恢复频率限制 | 15分钟3次（仅恢复） |

### 5.2 计量字段

备份模型 `app/Models/Backup.php` 中用于计费的关键字段：
- `bytes`：备份文件大小（字节）
- `disk`：存储适配器（`wings` 或 `s3`）
- `is_successful`：是否成功（成功才占用配额）
- `is_locked`：是否锁定（锁定的备份不能被自动清理）
- `completed_at`：完成时间

---

## 六、失败补偿机制

### 6.1 孤儿备份清理

**命令**：`p:maintenance:prune-backups`
**类**：`app/Console/Commands/Maintenance/PruneOrphanedBackupsCommand.php`

**触发时机**：定时任务（建议 cron 配置）

**逻辑**：
```php
$query = $this->backupRepository->getBuilder()
    ->whereNull('completed_at')
    ->where('created_at', '<=', CarbonImmutable::now()->subMinutes($since)->toDateTimeString());

$query->update([
    'is_successful' => false,
    'completed_at' => CarbonImmutable::now(),
]);
```

**默认配置**：`prune_age = 360` 分钟（6 小时），0 表示禁用

### 6.2 删除容错处理

**代码位置**：`app/Services/Backups/DeleteBackupService.php:47`

```php
try {
    $this->daemonBackupRepository->setServer($backup->server)->delete($backup);
} catch (DaemonConnectionException $exception) {
    $previous = $exception->getPrevious();
    // Wings 返回 404 时忽略，继续删除面板记录
    if (!$previous instanceof ClientException || 
        $previous->getResponse()->getStatusCode() !== Response::HTTP_NOT_FOUND) {
        throw $exception;
    }
}
```

### 6.3 失败自动解锁

**代码位置**：`app/Http/Controllers/Api/Remote/Backups/BackupStatusController.php:66`

```php
'is_locked' => $successful ? $model->is_locked : false,
```

失败的备份自动解锁，便于用户清理，避免占用配额。

### 6.4 S3 分片上传中止

备份失败时，若存在未完成的 S3 分片上传，自动调用 `AbortMultipartUpload`，避免产生不必要的存储费用。

---

## 七、核心代码索引

| 功能模块 | 文件路径 | 关键行号 |
|---------|---------|---------|
| 备份模型 | `app/Models/Backup.php` | - |
| 服务器模型（备份限制） | `app/Models/Server.php` | 45, 126, 215, 390 |
| 备份初始化服务 | `app/Services/Backups/InitiateBackupService.php` | 76, 92, 94, 109 |
| Wings 备份仓库 | `app/Repositories/Wings/DaemonBackupRepository.php` | 35, 60, 85 |
| 通信基类 | `app/Repositories/Wings/DaemonRepository.php` | 49 |
| 备份状态回调 | `app/Http/Controllers/Api/Remote/Backups/BackupStatusController.php` | 32, 66, 93, 103, 120 |
| S3 分片上传 | `app/Http/Controllers/Api/Remote/Backups/BackupRemoteUploadController.php` | 34 |
| 客户端备份控制器 | `app/Http/Controllers/Api/Client/Servers/BackupController.php` | 67, 198, 200, 206, 214 |
| 备份删除服务 | `app/Services/Backups/DeleteBackupService.php` | 29, 47 |
| 下载链接服务 | `app/Services/Backups/DownloadLinkService.php` | 24 |
| 节点 JWT 服务 | `app/Services/Nodes/NodeJWTService.php` | 63 |
| 守护进程认证中间件 | `app/Http/Middleware/Api/Daemon/DaemonAuthenticate.php` | 34 |
| 孤儿备份清理命令 | `app/Console/Commands/Maintenance/PruneOrphanedBackupsCommand.php` | 23 |
| 资源限流枚举 | `app/Enum/ResourceLimit.php` | 30, 46 |
| 定时任务备份限制 | `app/Http/Controllers/Api/Client/Servers/ScheduleTaskController.php` | 47, 107 |
| 路由服务提供者（全局限流） | `app/Providers/RouteServiceProvider.php` | 56, 93 |
| HTTP 限流配置 | `config/http.php` | 14-16 |
| 备份配置 | `config/backups.php` | - |
| 备份路由 | `routes/api-client.php` | 132-141 |
| 服务器状态异常 | `app/Exceptions/Http/Server/ServerStateConflictException.php` | 23 |
| 服务器访问认证中间件 | `app/Http/Middleware/Api/Client/Server/AuthenticateServerAccess.php` | 50 |
| 守护进程连接异常 | `app/Exceptions/Http/Connection/DaemonConnectionException.php` | 13, 28 |

---

## 八、关键设计决策总结

### 8.1 数据一致性
- 数据库事务确保数据库操作的原子性，但**不保护外部 HTTP 调用**
- 回调接口节点归属校验，防止越权操作
- 幂等性校验，避免重复处理
- 孤儿备份自动清理，作为超时场景的兜底补偿

### 8.2 安全性
- 双向认证：Panel → Wings 使用 Bearer Token，Wings → Panel 使用 Token 对
- 传输加密：HTTPS + JWT 签名链接
- S3 预签名 URL，避免密钥暴露

### 8.3 可靠性
- 孤儿备份自动清理，防止状态不一致
- 删除容错，Wings 404 不阻塞面板清理
- 失败自动解锁，便于用户处理
- S3 分片异常分级处理（静默/中止/抛错）
- 恢复回调无论成败都清除状态，避免死锁

### 8.4 可扩展性
- 适配器模式：Wings 本地 / S3 云存储可切换
- 配置驱动：节流、限流、分片大小等均可配置

---

## 九、重要代码事实修正清单

| 原理解 | 修正后 | 依据 |
|-------|--------|------|
| backup_limit=0 表示无限制 | backup_limit=0 表示禁止备份 | `app/Services/Backups/InitiateBackupService.php:94`、`app/Http/Controllers/Api/Client/Servers/ScheduleTaskController.php:47` |
| 创建和恢复使用相同限流 | 创建用 throttles（10min2次），恢复用 ResourceLimit（15min3次），且共用 api.client 全局限流；三者并存独立计数 | `routes/api-client.php:138`、`app/Services/Backups/InitiateBackupService.php:78`、`app/Providers/RouteServiceProvider.php:56`、`app/Enum/ResourceLimit.php:30` |
| 恢复要求备份必须成功 | 仅禁止"进行中"的备份，已失败的备份也允许恢复 | `app/Http/Controllers/Api/Client/Servers/BackupController.php:206` |
| S3 上传在完成回调之后 | S3 分片上传在完成回调之前完成 | `app/Http/Controllers/Api/Remote/Backups/BackupRemoteUploadController.php` + `BackupStatusController.php` 时序 |
| upload_id 缺失直接报错 | 失败场景下 upload_id 缺失静默返回 | `app/Http/Controllers/Api/Remote/Backups/BackupStatusController.php:124-130` |
| restore 回调根据 successful 做业务分支 | successful 仅用于审计事件名，无论成败都清除状态 | `app/Http/Controllers/Api/Remote/Backups/BackupStatusController.php:103-108` |
| 恢复只需检查 status===null | 还需通过 validateCurrentState()（未暂停/未维护/已安装/无转移） | `app/Http/Middleware/Api/Client/Server/AuthenticateServerAccess.php:50`、`app/Models/Server.php:390` |
| 数据库事务会回滚 Wings 动作 | 事务仅保护数据库操作，Wings 调用失败/超时会导致数据库回滚但远端动作已执行 | `app/Services/Backups/InitiateBackupService.php:109`、`app/Repositories/Wings/DaemonBackupRepository.php:50` |
| 三层限流互斥 | 三层限流并存生效，独立计数，任一触发都会拦截 | `app/Providers/RouteServiceProvider.php:93`、`app/Enum/ResourceLimit.php:61` |
