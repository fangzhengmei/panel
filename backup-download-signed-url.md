# Pterodactyl Panel 备份下载签名 URL 安全分析

## 1. 总体架构概览

Pterodactyl Panel 的备份下载流程分为两种存储适配器模式：

| 适配器类型 | 下载目标 | URL 类型 | 有效期 |
|-----------|---------|---------|--------|
| Wings（本地） | Wings 节点 `/download/backup` 端点 | JWT 签名 URL | **15 分钟** |
| AWS S3 | S3 存储桶 GetObject | AWS 预签名 URL (Signature v4) | **5 分钟** |

核心入口：[BackupController::download()](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Http/Controllers/Api/Client/Servers/BackupController.php#L167-L185)

---

## 2. Panel 端访问控制（URL 生成前）

### 2.1 路由与中间件链

路由定义：[routes/api-client.php:136](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/routes/api-client.php#L136)

```
GET /api/client/servers/{server}/backups/{backup}/download
```

中间件执行顺序（由 [RouteServiceProvider.php:56-59](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Providers/RouteServiceProvider.php#L56-L59) 和 [Kernel.php](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Http/Kernel.php) 定义）：

| 层级 | 中间件 | 作用 | 所在文件 |
|-----|--------|------|---------|
| `api` group | `EnsureStatefulRequests` | 处理有状态请求 | Laravel 内置 |
| `api` group | `auth:sanctum` | Sanctum Token / Session 认证 | Laravel Sanctum |
| `api` group | `IsValidJson` | 验证 JSON 请求格式 | [IsValidJson.php] |
| `api` group | `TrackAPIKey` | 追踪 API Key 使用 | [TrackAPIKey.php] |
| `api` group | `RequireTwoFactorAuthentication` | 强制双因素认证 | [RequireTwoFactorAuthentication.php] |
| `api` group | `AuthenticateIPAccess` | IP 白名单校验 | [AuthenticateIPAccess.php] |
| `client-api` group | `SubstituteClientBindings` | 路由模型绑定（支持 uuidShort/identifier） | [SubstituteClientBindings.php](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Http/Middleware/Api/Client/SubstituteClientBindings.php) |
| `client-api` group | `RequireClientApiKey` | 禁止使用 Application API Key | [RequireClientApiKey.php](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Http/Middleware/Api/Client/RequireClientApiKey.php) |
| 路由组 | `ServerSubject` | 活动日志服务器主体 | [ServerSubject.php] |
| 路由组 | `AuthenticateServerAccess` | 服务器访问授权 | [AuthenticateServerAccess.php](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Http/Middleware/Api/Client/Server/AuthenticateServerAccess.php) |
| 路由组 | `ResourceBelongsToServer` | 资源归属校验 | [ResourceBelongsToServer.php](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Http/Middleware/Api/Client/Server/ResourceBelongsToServer.php) |
| 路由组 | `throttle:api.client` | 速率限制：256 req/min/用户（或 IP） | [RouteServiceProvider.php:93-100](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Providers/RouteServiceProvider.php#L93-L100) |

### 2.2 AuthenticateServerAccess：用户身份校验

[AuthenticateServerAccess.php:29-67](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Http/Middleware/Api/Client/Server/AuthenticateServerAccess.php#L29-L67)

用户必须满足以下任一条件：
- `user.id === server.owner_id`（服务器所有者）
- `user.root_admin === true`（根管理员）
- 在 `server.subusers` 中存在匹配的 `user_id`（子用户）

同时校验服务器状态（[Server.php:390-401](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Models/Server.php#L390-L401)）：
- 非 suspended（已暂停）
- 节点非 maintenance_mode（维护中）
- 已安装完成（非 installing / install_failed）
- 非 restoring_backup（正在恢复备份）
- 无进行中的 transfer（服务器迁移）

### 2.3 ResourceBelongsToServer：跨服务器访问防护

[ResourceBelongsToServer.php:27-85](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Http/Middleware/Api/Client/Server/ResourceBelongsToServer.php#L27-L85)

对 Backup 模型强制执行：
```php
case Backup::class:
    if ($model->server_id !== $server->id) {
        throw new NotFoundHttpException(...);
    }
    break;
```

**关键防护**：即使通过 URL 参数篡改尝试访问 `{serverA}/backups/{backupB}`（backupB 属于 serverB），也会返回 404。

### 2.4 控制器层：下载权限校验

[BackupController.php:167-171](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Http/Controllers/Api/Client/Servers/BackupController.php#L167-L171)

```php
if (!$request->user()->can(Permission::ACTION_BACKUP_DOWNLOAD, $server)) {
    throw new AuthorizationException();
}
```

权限判定逻辑（[ServerPolicy.php](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Policies/ServerPolicy.php)）：
- 服务器所有者或根管理员 → **直接通过**
- 子用户 → 检查 `subuser.permissions` 中是否包含 `backup.download`

---

## 3. 签名 URL 生成机制

### 3.1 核心服务：DownloadLinkService

[DownloadLinkService.php](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Services/Backups/DownloadLinkService.php)

根据 `backup.disk` 字段分支处理：

```php
if ($backup->disk === Backup::ADAPTER_AWS_S3) {
    return $this->getS3BackupUrl($backup);   // S3 预签名 URL
}
// 否则：Wings JWT URL
```

### 3.2 Wings 适配器：JWT 签名 URL

生成逻辑：[DownloadLinkService.php:30-39](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Services/Backups/DownloadLinkService.php#L30-L39)

#### 3.2.1 JWT 标准声明（由 NodeJWTService 生成）

[NodeJWTService.php:63-102](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Services/Nodes/NodeJWTService.php#L63-L102)

| 声明 (Claim) | 值来源 | 作用 |
|-------------|--------|------|
| `iss` | `config('app.url')` | 签发者：Panel 自身 URL |
| `aud` | `$node->getConnectionAddress()` | 受众：目标 Wings 节点地址 |
| `jti` | `md5($user->id . $server->uuid)` | JWT 唯一标识（**确定性生成**） |
| `iat` | `CarbonImmutable::now()` | 签发时间 |
| `nbf` | `CarbonImmutable::now()->subMinutes(5)` | 生效时间（允许 5 分钟时钟偏移） |
| `exp` | `CarbonImmutable::now()->addMinutes(15)` | **过期时间：15 分钟** |
| `sub` | 未设置 | 无 |

#### 3.2.2 自定义业务声明

| 声明 | 值来源 | 作用 |
|-----|--------|------|
| `backup_uuid` | `$backup->uuid` | 目标备份的 UUID |
| `server_uuid` | `$backup->server->uuid` | 目标服务器的 UUID |
| `user_uuid` | `$user->uuid` | 请求用户 UUID（审计用） |
| `user_id` | `$user->id` | 请求用户数字 ID（已弃用，向后兼容） |
| `unique_id` | `Str::random()` | 随机唯一字符串 |

#### 3.2.3 签名算法与密钥

- **算法**：HS256（HMAC-SHA256，对称加密）
- **密钥**：每个 Wings 节点独立的 `daemon_token`（解密后）
  - 密钥存储：[Node.php:189-194](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Models/Node.php#L189-L194)
  - 加密方式：Laravel Encrypter（AES-256-CBC）
  - 获取方式：`$node->getDecryptedKey()`

#### 3.2.4 最终 URL 格式

```
{scheme}://{fqdn}:{daemonListen}/download/backup?token={jwt_string}
```

示例：
```
https://node1.example.com:8080/download/backup?token=eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9...
```

### 3.3 S3 适配器：AWS 预签名 URL

生成逻辑：[DownloadLinkService.php:46-61](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Services/Backups/DownloadLinkService.php#L46-L61)

使用 AWS SDK `createPresignedRequest` 生成 GetObject 预签名 URL：

| 参数 | 值 |
|-----|----|
| Bucket | 配置的 `AWS_BACKUPS_BUCKET` |
| Key | `{server_uuid}/{backup_uuid}.tar.gz` |
| ContentType | `application/x-gzip` |
| 有效期 | **5 分钟** |
| 签名算法 | AWS Signature v4（由 SDK 处理） |

---

## 4. 有效期与使用次数控制

### 4.1 有效期总结

| 适配器 | URL 有效期 | 配置位置 |
|-------|-----------|---------|
| Wings JWT | **15 分钟**（硬编码） | [DownloadLinkService.php:31](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Services/Backups/DownloadLinkService.php#L31) |
| S3 预签名 | **5 分钟**（硬编码） | [DownloadLinkService.php:57](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Services/Backups/DownloadLinkService.php#L57) |
| S3 分片上传 URL | 默认 60 分钟（可配置） | [config/backups.php:13](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/config/backups.php#L13) `BACKUP_PRESIGNED_URL_LIFESPAN` |

### 4.2 ⚠️ 单次/多次下载控制——无限制

**关键发现**：Panel 代码中**不存在**任何机制来：
- 限制签名 URL 的使用次数
- 记录 URL 是否已被使用
- 在首次下载后撤销 URL
- 实现一次性 token

**结论**：在有效期内，获取到 URL 的任何人可以**无限次重复下载**。

### 4.3 URL 生成频率限制

通过 `throttle:api.client` 限制：
- 每用户（或未认证时按 IP）每分钟最多 256 次请求
- 配置位置：[config/http.php:14-20](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/config/http.php#L14-L20)

这仅限制 URL **生成**频率，不限制已生成 URL 的使用。

---

## 5. Wings 节点回源验证与跨域访问防护

> 注：Wings 节点的 Go 代码不在本仓库中。以下分析基于 Panel 端的 JWT 声明设计推断其验证逻辑。

### 5.1 JWT 验证预期流程（Wings 端）

Wings 收到 `/download/backup?token=...` 请求后，应执行：

1. **签名验证**：使用自身的 `daemon_token`（与 Panel 共享）验证 HMAC-SHA256 签名
2. **标准声明验证**：
   - `exp`：JWT 未过期
   - `nbf`：JWT 已生效
   - `aud`：与节点自身连接地址匹配（防止 Token 被转发到其他节点）
3. **业务声明验证**：
   - `backup_uuid`：对应备份存在且属于本机
   - `server_uuid`：对应服务器存在且运行在本机
   - 校验备份的 `server_id` 与声明中服务器匹配

### 5.2 跨节点访问防护

**机制**：JWT 使用**节点专属密钥**签名。

- 节点 A 的 `daemon_token` ≠ 节点 B 的 `daemon_token`
- 为节点 A 生成的 JWT 无法被节点 B 验证通过
- 攻击面：Panel 上某个节点的 `daemon_token` 泄露仅影响该节点

### 5.3 跨服务器访问防护

**Panel 端**（生成前）：
- [ResourceBelongsToServer.php:49-57](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Http/Middleware/Api/Client/Server/ResourceBelongsToServer.php#L49-L57)：`$backup->server_id === $server->id`

**Wings 端**（通过声明）：
- JWT 中同时包含 `backup_uuid` 和 `server_uuid`
- Wings 应验证该备份确实属于该服务器，且服务器运行在本节点

### 5.4 跨用户访问防护

**Panel 端**（生成前）：
- [AuthenticateServerAccess.php:42-47](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Http/Middleware/Api/Client/Server/AuthenticateServerAccess.php#L42-L47)：仅所有者/管理员/子用户可访问
- [ServerPolicy.php](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Policies/ServerPolicy.php)：子用户需显式拥有 `backup.download` 权限

**⚠️ URL 泄露风险**：
- JWT 中包含 `user_uuid` 和 `user_id`，但这些仅用于审计日志
- **一旦 URL 泄露**，任何持有 URL 的第三方在 15 分钟内均可匿名下载备份文件
- Wings 下载端点不会（也无法）再次校验用户身份——它只有 JWT，没有 Panel 的 Session/Token

---

## 6. S3 适配器的特殊考量

S3 模式下，下载直接发生在用户浏览器与 AWS S3 之间，完全绕过 Wings 节点。

### 6.1 安全边界

| 层面 | 防护措施 |
|-----|---------|
| Panel 生成前 | 同 2.1-2.4 全套中间件 + 权限校验 |
| 传输中 | HTTPS（TLS） |
| S3 侧 | 预签名 URL 仅签名 GetObject，不可用于其他操作 |
| 有效期 | 5 分钟 |

### 6.2 ⚠️ S3 模式下的匿名下载风险

S3 预签名 URL **不包含**任何用户或服务器身份标识：
- URL 中仅包含 AWS Signature v4 参数（`X-Amz-Signature`、`X-Amz-Date`、`X-Amz-Expires` 等）
- 任何持有 URL 的人在 5 分钟内均可下载
- AWS 侧无额外访问控制

---

## 7. 备份相关远程 API（Wings → Panel 回源）

Wings 节点通过 `/api/remote/*` 回源 Panel，使用独立的认证机制。

### 7.1 DaemonAuthenticate：Wings 身份认证

[DaemonAuthenticate.php:34-66](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Http/Middleware/Api/Daemon/DaemonAuthenticate.php#L34-L66)

认证方式：
- Header: `Authorization: Bearer {token_id}.{token}`
- 用 `token_id` 在数据库中查找 Node
- 使用 Laravel Encrypter 解密 `node.daemon_token`，与请求中的 `token` 做 `hash_equals` 比较

### 7.2 备份状态回传的节点归属校验

[BackupRemoteUploadController.php:48-53](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Http/Controllers/Api/Remote/Backups/BackupRemoteUploadController.php#L48-L53)
[BackupStatusController.php:44-49](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Http/Controllers/Api/Remote/Backups/BackupStatusController.php#L44-L49)

```php
$server = $model->server;
if ($server->node_id !== $node->id) {
    throw new HttpForbiddenException('Requesting node does not have permission to access this server.');
}
```

防止节点 A 冒充节点 B 上报备份状态。

---

## 8. 安全评审结论与风险点汇总

### 8.1 已实现的防护机制 ✅

| 防护 | 实现位置 | 说明 |
|-----|---------|------|
| 用户身份认证 | `auth:sanctum` | Session 或 API Token |
| 服务器访问授权 | `AuthenticateServerAccess` | 所有者/管理员/子用户校验 |
| 资源归属校验 | `ResourceBelongsToServer` | `backup.server_id === server.id` |
| 下载权限控制 | `backup.download` 权限 | 子用户需显式授权 |
| URL 有效期限制 | JWT `exp` / AWS 预签名 | 15 分钟 / 5 分钟 |
| 节点间隔离 | 节点专属密钥签名 JWT | 跨节点 Token 不可用 |
| Wings 回源认证 | `DaemonAuthenticate` | `token_id.token` 双字段校验 |
| 节点归属校验 | 备份远程 API | 校验 `server.node_id === request.node.id` |

### 8.2 风险点与设计边界 ⚠️

| 风险 | 严重程度 | 说明 |
|-----|---------|------|
| **有效期内可重复下载** | 中高 | 无一次性 Token 机制，15 分钟（Wings）/ 5 分钟（S3）内可多次下载 |
| **URL 泄露即可匿名下载** | 高 | 一旦 URL 通过日志、浏览器历史、Referer 等泄露，第三方无需认证即可下载 |
| **Wings 下载端点无用户级校验** | 中 | Wings 仅验证 JWT，不校验请求发起者是否为 JWT 中声明的 user |
| **S3 预签名 URL 无身份上下文** | 中高 | AWS 侧完全无法追溯下载者身份 |
| **JWT `jti` 确定性生成** | 低 | `md5($userId . $serverUuid)` 可预测，但 `unique_id` 随机字段提供了熵 |
| **硬编码有效期** | 低 | 15 分钟和 5 分钟不可配置（仅 S3 上传 URL 可配置） |

### 8.3 安全边界声明

签名 URL 的使用边界定义为：

1. **时间边界**：Wings 模式 15 分钟，S3 模式 5 分钟，过期自动失效
2. **节点边界**：JWT 由节点专属密钥签发，仅目标节点可验证
3. **服务器边界**：JWT Claims 绑定 `backup_uuid` + `server_uuid`，不可跨服务器使用
4. ****：生成前需通过 Panel 全套认证与权限校验，但生成后 URL 本身不承载用户级访问控制
5. **次数边界**：无限制——有效期内可重复使用

---

## 9. 文件索引

| 组件 | 文件路径 |
|-----|---------|
| 备份下载控制器 | [BackupController.php](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Http/Controllers/Api/Client/Servers/BackupController.php) |
| 下载链接服务 | [DownloadLinkService.php](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Services/Backups/DownloadLinkService.php) |
| 节点 JWT 服务 | [NodeJWTService.php](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Services/Nodes/NodeJWTService.php) |
| 服务器访问中间件 | [AuthenticateServerAccess.php](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Http/Middleware/Api/Client/Server/AuthenticateServerAccess.php) |
| 资源归属中间件 | [ResourceBelongsToServer.php](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Http/Middleware/Api/Client/Server/ResourceBelongsToServer.php) |
| 服务器权限策略 | [ServerPolicy.php](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Policies/ServerPolicy.php) |
| 权限常量定义 | [Permission.php](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Models/Permission.php) |
| 节点模型（密钥） | [Node.php](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Models/Node.php) |
| 备份模型 | [Backup.php](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Models/Backup.php) |
| 备份配置 | [backups.php](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/config/backups.php) |
| Wings 认证中间件 | [DaemonAuthenticate.php](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Http/Middleware/Api/Daemon/DaemonAuthenticate.php) |
| 备份 S3 上传回源 | [BackupRemoteUploadController.php](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Http/Controllers/Api/Remote/Backups/BackupRemoteUploadController.php) |
| 备份状态回源 | [BackupStatusController.php](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Http/Controllers/Api/Remote/Backups/BackupStatusController.php) |
| 备份授权测试 | [BackupAuthorizationTest.php](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/tests/Integration/Api/Client/Server/Backup/BackupAuthorizationTest.php) |
