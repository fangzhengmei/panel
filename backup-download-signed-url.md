# Pterodactyl Panel 备份下载签名 URL 安全分析（修订版）

## 1. 总体架构概览

Pterodactyl Panel 的备份下载流程分为两种存储适配器模式：

| 适配器类型 | 下载目标 | URL 类型 | 有效期 |
|-----------|---------|---------|--------|
| Wings（本地） | Wings 节点 `/download/backup` 端点 | JWT 签名 URL | **15 分钟** |
| AWS S3 | S3 存储桶 GetObject | AWS 预签名 URL (Signature v4) | **5 分钟** |

---

## 2. 服务端生成签名 URL 的全部代码路径

`DownloadLinkService::handle()` 在 Panel 仓库中被调用的地方共有 **2 处**，服务于两种不同的业务场景：

### 2.1 路径 A：用户显式下载备份

**触发方式**：用户点击前端 "Download" 按钮

| 环节 | 位置 |
|-----|------|
| 路由 | `GET /api/client/servers/{server}/backups/{backup}/download` [routes/api-client.php:136](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/routes/api-client.php#L136) |
| 控制器 | [BackupController::download()](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Http/Controllers/Api/Client/Servers/BackupController.php#L167-L185) |
| 调用链 | `BackupController::download → DownloadLinkService::handle → 返回 JSON { attributes.url }` |

后端校验（代码可验证）：
1. **备份类型校验**：仅允许 `disk === ADAPTER_AWS_S3 (s3)` 或 `disk === ADAPTER_WINGS (wings)`，其他磁盘驱动抛出 BadRequestHttpException
2. **权限校验**：`Permission::ACTION_BACKUP_DOWNLOAD`（通过 `$user->can()`）
3. **URL 返回格式**：
   ```json
   {
     "object": "signed_url",
     "attributes": { "url": "..." }
   }
   ```

### 2.2 路径 B：恢复备份时 Wings 内部拉取（仅 S3 适配器）

**触发方式**：用户点击前端 "Restore" 按钮（且备份为 S3 存储）

| 环节 | 位置 |
|-----|------|
| 路由 | `POST /api/client/servers/{server}/backups/{backup}/restore` [routes/api-client.php:139](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/routes/api-client.php#L139) |
| 控制器 | [BackupController::restore()](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Http/Controllers/Api/Client/Servers/BackupController.php#L198-L229) |
| 调用链 | `BackupController::restore → (仅当 S3 时) DownloadLinkService::handle → 将 URL 作为参数传给 Wings` |

关键代码（[BackupController.php:215-225](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Http/Controllers/Api/Client/Servers/BackupController.php#L215-L225)）：
```php
if ($backup->disk === Backup::ADAPTER_AWS_S3) {
    $url = $this->downloadLinkService->handle($backup, $request->user());
}
$this->daemonRepository->setServer($server)->restore($backup, $url ?? null, $truncate);
```

**此路径的特殊性**：
- 生成的 URL **不会返回给用户浏览器**，而是作为 `download_url` 参数通过 Panel→Wings 的内部 HTTPS 请求发送
- 实际执行下载的是 Wings 节点（服务端到服务端），不是用户浏览器
- Wings 适配器（本地备份）不需要此步骤，因为文件已在本机磁盘

### 2.3 对比：文件下载（非备份）使用的签名 URL

作为参考，Panel 中还存在一个类似机制——**服务器文件下载**：

- 路由：`GET /api/client/servers/{server}/files/download` [FileController::download()](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Http/Controllers/Api/Client/Servers/FileController.php#L77-L100)
- 同样使用 `NodeJWTService`，相同的 15 分钟有效期
- JWT Claims 差异：包含 `file_path` 而不是 `backup_uuid`
- 端点差异：Wings 端走 `/download/file`（备份走 `/download/backup`）

### 2.4 Application API 中不存在备份下载端点

已搜索 `routes/api-application.php`，Application API 只提供服务器、节点等管理接口，**不提供用户级备份下载或恢复接口**。生成签名 URL 的唯一入口是 Client API 上述两条路径。

---

## 3. 前端按钮限制 vs 后端实际校验（差异对比）

### 3.1 前端 UI 层面的限制逻辑

#### 3.1.1 备份行操作按钮可见性

[BackupRow.tsx:89-99](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/resources/scripts/components/server/backups/BackupRow.tsx#L89-L99)
```tsx
<Can action={['backup.download', 'backup.restore', 'backup.delete']} matchAny>
    {!backup.completedAt ? (invisible placeholder) : (操作菜单)}
</Can>
```
- **限制 1**：仅当用户拥有 `backup.download` / `backup.restore` / `backup.delete` 三者中的至少一个权限时才显示操作菜单
- **限制 2**：`backup.completedAt === null`（备份未完成/进行中）时菜单占位不可见

#### 3.1.2 下载/恢复菜单项的具体限制

[BackupContextMenu.tsx:165-208](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/resources/scripts/components/server/backups/BackupContextMenu.tsx#L165-L208)

```tsx
{backup.isSuccessful ? (
    <DropdownMenu>
        <Can action={'backup.download'}>
            <DropdownButtonRow onClick={doDownload}>Download</DropdownButtonRow>
        </Can>
        <Can action={'backup.restore'}>
            <DropdownButtonRow onClick={() => setModal('restore')}>Restore</DropdownButtonRow>
        </Can>
        {/* ... lock / delete ... */}
    </DropdownMenu>
) : (
    /* only delete button */
)}
```

| 前端限制条件 | 位置 | 说明 |
|-------------|------|------|
| `backup.isSuccessful === true` | [BackupContextMenu.tsx:165](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/resources/scripts/components/server/backups/BackupContextMenu.tsx#L165) | 仅成功备份显示下载/恢复菜单；失败/进行中只显示删除 |
| `<Can action={'backup.download'}>` | [BackupContextMenu.tsx:177](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/resources/scripts/components/server/backups/BackupContextMenu.tsx#L177) | 权限组件，缺少权限时按钮不渲染 |
| `<Can action={'backup.restore'}>` | [BackupContextMenu.tsx:183](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/resources/scripts/components/server/backups/BackupContextMenu.tsx#L183) | 同上 |

#### 3.1.3 前端 API 请求封装

- 下载：[getBackupDownloadUrl.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/resources/scripts/api/server/backups/getBackupDownloadUrl.ts)
  - 直接 GET `/api/client/servers/${uuid}/backups/${backup}/download`
  - 解析返回值中的 `data.attributes.url`
- 恢复：[backups/index.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/resources/scripts/api/server/backups/index.ts)
  - 直接 POST `/api/client/servers/${uuid}/backups/${backup}/restore`
  - Body: `{ truncate }`

### 3.2 后端实际校验（可绕过前端的情况）

以下表格**对比前端限制与后端校验的差异**，差异行标 ★：

| 校验项 | 前端限制 | 后端 download() 校验 | 后端 restore() 校验 |
|-------|---------|---------------------|--------------------|
| 用户认证（Session/Token） | — （隐含） | ✅ `auth:sanctum` | ✅ `auth:sanctum` |
| 服务器访问授权 | —（隐含） | ✅ `AuthenticateServerAccess` | ✅ `AuthenticateServerAccess` |
| 资源归属（backup 属于 server） | —（隐含） | ✅ `ResourceBelongsToServer` | ✅ `ResourceBelongsToServer` |
| **备份完成状态** | ❌ `isSuccessful` 显示条件（仅成功的显示下载/恢复菜单） | ★ **完全未校验**（代码中无 `is_successful` / `completed_at` 检查，任何状态均可生成下载链接） | ★ **仅拒绝"进行中"**（`!is_successful && is_null(completed_at)` → 拒绝；失败但已完成的备份**允许恢复**） |
| **具体权限** | ❌ `<Can action="backup.download">` | ✅ `backup.download`（`$user->can()`） | ✅ `backup.restore`（FormRequest `permission()`） |
| 磁盘驱动类型合法 | — | ✅ 仅 s3 / wings | —（不校验，直接传 adapter 给 Wings） |
| 服务器状态可操作 | — | ✅ `AuthenticateServerAccess.validateCurrentState()` | ✅ `server.status === null`（仅完全空闲状态允许，比 download 更严格） |

### 3.3 差异点说明：备份完成状态校验缺失

**关键发现**：[BackupController::download()](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Http/Controllers/Api/Client/Servers/BackupController.php#L167-L185) 方法中**完全没有**检查 `$backup->is_successful` 或 `$backup->completed_at`。

这意味着：
- 即使备份失败（`is_successful = false`），只要磁盘驱动合法且用户拥有 `backup.download` 权限，后端仍会生成签名 URL
- 即使备份仍在进行中（`completed_at = null`），后端也会生成签名 URL
- 前端 UI 屏蔽了这些情况，但通过直接构造 HTTP 请求（curl、Postman 等）可以绕过前端限制，获取进行中或失败备份的下载链接
- 对比：`restore()` 中正确校验了状态：`if (!$backup->is_successful && is_null($backup->completed_at)) { throw ... }`

---

## 4. 用户身份相关 Claim 用途深度分析

### 4.1 NodeJWTService 中用户身份字段注入逻辑

[NodeJWTService.php:39-97](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Services/Nodes/NodeJWTService.php#L39-L97)

```php
public function setUser(User $user): self {
    $this->user = $user;
    return $this;
}
// ...
if (!is_null($this->user)) {
    $builder = $builder
        ->withClaim('user_uuid', $this->user->uuid)
        ->withClaim('user_id', $this->user->id);  // deprecated
}
```

### 4.2 各身份相关字段用途详解

| Claim / 参数 | 值来源 | 注入位置 | 用途（基于代码的推断） |
|--------------|--------|---------|----------------------|
| `user_uuid` | `User.uuid`（UUID 字符串） | [NodeJWTService.php:90](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Services/Nodes/NodeJWTService.php#L90) | **Wings 审计日志用**。Wings 节点在下载日志中记录"哪个用户触发了下载"，但不以此做访问控制——因为 Wings 没有 Panel 的用户数据库，无法回查此 UUID 的有效性。 |
| `user_id` | `User.id`（自增数字 ID） | [NodeJWTService.php:96](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Services/Nodes/NodeJWTService.php#L96) | **已弃用**。代码注释明确标注为 deprecated，保留仅为向后兼容旧版 Wings。将在 Panel@1.11+ 移除。不能用于安全决策。 |
| `identifiedBy` 参数 | `$user->id . $server->uuid`（字符串拼接） | 备份下载：[DownloadLinkService.php:37](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Services/Backups/DownloadLinkService.php#L37)<br>文件下载：[FileController.php:86](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Http/Controllers/Api/Client/Servers/FileController.php#L86) | 用于生成 **`jti`（JWT ID）**：`md5($userId . $serverUuid)`。**确定性生成**——同一用户对同一服务器重复请求，得到的 jti 完全相同。 |
| `jti` | `md5(identifiedBy)` | [NodeJWTService.php:65-72](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Services/Nodes/NodeJWTService.php#L65-L72) | JWT 标准唯一标识。由于确定性，它**不能用于防止重放攻击**。Wings 若实现防重放，需依赖 `unique_id`。 |
| `unique_id` | `Str::random()`（40 字符随机串） | [NodeJWTService.php:100](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Services/Nodes/NodeJWTService.php#L100) | **真正的随机唯一标识**。每次生成 JWT 都会产生全新值，如 Wings 需要做使用次数限制或防重放，应基于此字段做缓存/黑名单。 |

### 4.3 不同 JWT 场景的 Claims 对比（佐证身份字段用途）

| 场景 | 调用方 | setClaims 内容 | 是否 setUser | 额外说明 |
|-----|--------|---------------|-------------|---------|
| 备份下载 | DownloadLinkService | `backup_uuid`, `server_uuid` | ✅ | **不含 permissions** → Wings 无法做用户级权限校验，只能依赖 JWT 有效 |
| 文件下载 | FileController | `file_path`, `server_uuid` | ✅ | 同上，不含 permissions |
| 文件上传 | FileUploadController | `server_uuid` | ✅ | 同上 |
| Websocket 连接 | WebsocketController | `server_uuid`, `permissions[]` | ✅ | **含完整 permissions 数组**。Wings 基于此做实时控制台操作的权限校验 |

**结论**：仅在 Websocket 场景下，Wings 才有能力进行用户级细粒度权限校验（因为 JWT 包含完整权限列表）。备份/文件的下载上传 JWT 仅携带用户身份标识用于**审计日志**，不携带权限信息。

---

## 5. 浏览器获取签名 URL 后的完整跳转路径

### 5.1 流程时序图（用户主动下载场景）

```
用户浏览器                     Panel API                 Wings / S3
    |                            |                         |
    |  1. 点击 Download 按钮      |                         |
    | -------------------------> |                         |
    |  (BackupContextMenu.doDownload)                     |
    |                            |                         |
    |                            |  2. 全套中间件校验       |
    |                            |  (见第 6 节)            |
    |                            |                         |
    |                            |  3. DownloadLinkService |
    |                            |     生成签名 URL         |
    |                            |                         |
    |  4. HTTP 200 JSON          |                         |
    | <------------------------- |                         |
    |  { object: "signed_url",   |                         |
    |    attributes: { url } }   |                         |
    |                            |                         |
    |  5. window.location = url  |                         |
    |  (BackupContextMenu.tsx:45)|                         |
    | -------------------------------------------------->  |
    |         HTTP GET 请求（携带 token query param）        |
    |                            |                         |
    |                            |  6A. Wings 模式：        |
    |                            |      验证 JWT 签名/声明   |
    |                            |      查找备份文件        |
    |  6B. S3 模式：             |                         |
    |   AWS Signature 验证       |                         |
    |   GetObject 执行           |                         |
    | <--------------------------------------------------  |
    |    HTTP 200 + 文件流       |                         |
    |    (Content-Disposition)   |                         |
```

### 5.2 每一步骤的代码定位

| 步骤 | 代码位置 | 说明 |
|-----|---------|------|
| 1 | [BackupContextMenu.tsx:39-52](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/resources/scripts/components/server/backups/BackupContextMenu.tsx#L39-L52) | `doDownload()` 触发 API 请求，期间显示 `SpinnerOverlay` |
| 4 | [getBackupDownloadUrl.ts:3-8](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/resources/scripts/api/server/backups/getBackupDownloadUrl.ts#L3-L8) | `resolve(data.attributes.url)` — 解析响应中的 URL 字段 |
| 5 | [BackupContextMenu.tsx:45](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/resources/scripts/components/server/backups/BackupContextMenu.tsx#L45) | `window.location = url` — 浏览器整页导航到目标下载地址（**非 AJAX**，非 `window.open`，非 `a[download]`） |

### 5.3 跳转方式的安全含义

使用 `window.location = url` 而非 `fetch()` / `XMLHttpRequest` 下载文件：

**优点**：
- 无论目标 URL 返回什么内容类型（S3 的 `application/x-gzip` 或 Wings 的附件响应），浏览器都会按 Content-Disposition 正确处理
- 无需处理二进制流的 Blob 转换
- 自动携带浏览器 Cookie（Wings 模式下通常不需要，因为 JWT 在 URL 参数中）

**缺点（安全风险点）**：
- **Referer 泄露**：若 Panel 页面包含外部资源或导航到 Wings/S3，浏览器默认会发送 `Referer` 头，其中包含 Panel 的当前页面 URL。但下载签名 URL 本身已包含 token，主要风险在于：
  - Referer 中可能携带敏感信息（如 server UUID）
  - 若 Wings 域配置错误，可能被上游代理记录完整 URL
- **导航即下载**：第 5 步执行后，用户无法在当前页面继续操作（除非浏览器弹出"保存文件"对话框后页面仍停留在原处）。若失败，用户需要后退返回。

### 5.4 Wings 模式下：可直接验证 vs 需推断的下载端点信息

#### ✅ Panel 代码中可直接验证 / 确认的信息：

| 项目 | 证据 |
|-----|------|
| Wings 端点路径：`/download/backup` | [DownloadLinkService.php:39](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Services/Backups/DownloadLinkService.php#L39) 硬编码的 `sprintf('%s/download/backup?token=%s', ...)` |
| Token 传输方式：`?token=` Query 参数 | 同上 |
| JWT 必须包含的业务 Claims：`backup_uuid` + `server_uuid` | [DownloadLinkService.php:33-36](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Services/Backups/DownloadLinkService.php#L33-L36) |
| JWT 的签名密钥：目标节点的 `daemon_token`（HS256） | [NodeJWTService.php:66](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Services/Nodes/NodeJWTService.php#L66) + [Node.php:189-194](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Models/Node.php#L189-L194) |
| JWT 标准 Claims 存在：`iss`, `aud`, `jti`, `iat`, `nbf`, `exp` | [NodeJWTService.php:68-78](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Services/Nodes/NodeJWTService.php#L68-L78) |
| `aud` 值：节点自身的连接地址（scheme+fqdn+port） | [Node.php:134-137](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Models/Node.php#L134-L137) `getConnectionAddress()` |
| 标准 JWT Header：算法 HS256，附加 `jti` 和（可选）`sub` 头部 | [NodeJWTService.php:72+81](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Services/Nodes/NodeJWTService.php#L72) |
| URL 生成时会记录 Activity 日志（Panel 侧） | [BackupController.php:179](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Http/Controllers/Api/Client/Servers/BackupController.php#L179) `server:backup.download` 事件写入 `activity_logs` |

#### ⚠️ 仅能通过 Claim 设计与项目惯例推断的信息（Wings Go 代码不在本仓库）：

| 项目 | 推断依据 | 合理推断结论 | 置信度 |
|-----|---------|-------------|--------|
| Wings 如何验证 JWT 签名 | 使用与 Panel 相同的 `daemon_token` | Wings 使用 JWT 库（Go 生态如 `golang-jwt/jwt`）验证 HS256 签名，确保 Token 未被篡改 | 高 |
| 是否校验 `exp` / `nbf` | 标准 JWT 实现的默认行为 | **会校验**。`exp` 保证 15 分钟过期，`nbf` 允许 5 分钟时钟偏移 | 高 |
| 是否校验 `aud`（防止跨节点使用） | `aud` 设为节点自身地址 | **大概率会校验**。节点只接受 `aud === 自身连接地址` 的 JWT，防止将 Token 转发到其他节点 | 高 |
| 是否校验 `backup_uuid` 存在且属于本机 | Claim 同时带 `backup_uuid` 和 `server_uuid` | **会校验**：Wings 先查本机 `server_uuid` 是否存在，再查其下是否有 `backup_uuid` 对应的备份文件 | 高 |
| 是否校验 `backup_uuid` 与 `server_uuid` 关联一致 | 两层绑定（Claim + 本机磁盘结构） | **会校验**：防止用服务器 A 的备份 UUID 去访问服务器 B 的备份 | 高 |
| 是否验证备份成功状态（`is_successful`） | Panel 的 download() 未做此校验 | **未知**：Wings 可能检查备份元数据文件中的成功标记。若未检查，可能允许下载进行中/失败的半成品备份 | 中 |
| `user_uuid` 是否被使用 | Websocket 场景的 `permissions` 字段对比 | **仅用于 Wings 端的访问日志**（如 "user X downloaded backup Y at Z time"），不参与访问控制决策 | 高 |
| 是否实现 `jti` / `unique_id` 防重放 | 有效期仅 15 分钟 | **大概率不实现**：实践中更常见以 `exp` 做时限控制，不维护 Token 使用状态数据库。即便实现，也基于 `unique_id`（非确定性 jti） | 中 |
| 响应头是否包含 `Content-Disposition: attachment` | Panel 代码无法确认 | **会设置**：作为备份下载，Wings 会设置 `Content-Disposition: attachment; filename="{backup_name}.tar.gz"` 或类似值，确保浏览器触发"保存文件"对话框 | 高 |
| 是否校验 `iss` | `iss` = `config('app.url')` | **大概率不校验**：Wings 通常不知道 Panel 的 app.url（除非在 daemon 配置中显式同步），校验 iss 的概率较低 | 中 |
| Wings 是否对 Token 使用 IP 绑定 | Claim 中无 IP 字段 | **不实现**：JWT Claims 不含客户端 IP，Wings 无法做 IP 绑定。同 JWT 可被任何网络位置使用 | 高 |
| Wings 下载请求后是否有使用标记回源 Panel | `/api/remote/*` 路由无对应端点 | **不实现**：Panel 的远程 API 仅包含上传 URL 分配、状态上报、恢复完成上报，无"Token 已消费"回调 | 高 |

### 5.5 S3 模式下的跳转路径（可完全由 AWS SDK 行为确认）

S3 预签名 URL 的行为由 AWS SDK 与 S3 服务端协议保证：

| 项目 | 说明 |
|-----|------|
| 跳转目标 | S3 服务端 GetObject 预签名 URL（完整 URL 含 `X-Amz-Algorithm`, `X-Amz-Credential`, `X-Amz-Date`, `X-Amz-Expires`, `X-Amz-SignedHeaders`, `X-Amz-Signature` 等参数） |
| 有效期 | 5 分钟（由 `X-Amz-Expires=300` 秒参数指定） |
| 签名覆盖范围 | `host` header + 所有 Query 参数（AWS SigV4 规范），**无法在签名后篡改任何参数** |
| 响应内容类型 | `Content-Type: application/x-gzip`（由 Panel 在生成预签名 URL 时指定） |
| 下载文件名 | 由 S3 对象 Key（`{server_uuid}/{backup_uuid}.tar.gz`）决定，当前代码**未设置** `response-content-disposition` 自定义文件名 |
| 访问日志 | 可在 AWS CloudTrail / S3 Access Logs 中看到 GetObject 调用，但日志中的请求者身份是预签名 URL 中 `X-Amz-Credential` 对应的 AWS IAM 用户，**不包含 Panel 的用户身份** |

---

## 6. Panel 端访问控制（URL 生成前）

### 6.1 路由与中间件链

路由定义：[routes/api-client.php:132-141](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/routes/api-client.php#L132-L141)

备份相关路由完整列表：

```
GET    /api/client/servers/{server}/backups                   (index)
POST   /api/client/servers/{server}/backups                   (store)
GET    /api/client/servers/{server}/backups/{backup}          (view)
GET    /api/client/servers/{server}/backups/{backup}/download (download) ← 生成签名 URL
POST   /api/client/servers/{server}/backups/{backup}/lock     (toggleLock)
POST   /api/client/servers/{server}/backups/{backup}/restore  (restore) ← S3 时内部生成 URL
DELETE /api/client/servers/{server}/backups/{backup}          (delete)
```

所有 `{server}/{backup}` 路由共享相同的中间件栈（由 [RouteServiceProvider.php:56-59](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Providers/RouteServiceProvider.php#L56-L59) 和 [Kernel.php](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Http/Kernel.php) 定义）：

| 层级 | 中间件 | 作用 | 所在文件 |
|-----|--------|------|---------|
| `api` group | `EnsureStatefulRequests` | 处理有状态请求（Session Cookie 等） | Laravel 内置 |
| `api` group | `auth:sanctum` | Sanctum Token / Laravel Session 认证 | Laravel Sanctum |
| `api` group | `IsValidJson` | 验证请求为合法 JSON | [IsValidJson.php] |
| `api` group | `TrackAPIKey` | 活动日志中关联 API Key（若使用） | [TrackAPIKey.php] |
| `api` group | `RequireTwoFactorAuthentication` | 强制双因素认证（若用户启用） | [RequireTwoFactorAuthentication.php] |
| `api` group | `AuthenticateIPAccess` | IP 白名单校验（若用户设置） | [AuthenticateIPAccess.php] |
| `client-api` group | `SubstituteClientBindings` | 路由模型绑定（支持 uuidShort/identifier） | [SubstituteClientBindings.php](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Http/Middleware/Api/Client/SubstituteClientBindings.php) |
| `client-api` group | `RequireClientApiKey` | 禁止将 Application API Key 用于 Client 端点 | [RequireClientApiKey.php](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Http/Middleware/Api/Client/RequireClientApiKey.php) |
| 路由组 | `ServerSubject` | 活动日志的服务器主体绑定 | [ServerSubject.php] |
| 路由组 | `AuthenticateServerAccess` | 用户对服务器的访问授权校验 | [AuthenticateServerAccess.php](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Http/Middleware/Api/Client/Server/AuthenticateServerAccess.php) |
| 路由组 | `ResourceBelongsToServer` | 资源归属：backup.server_id === server.id | [ResourceBelongsToServer.php](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Http/Middleware/Api/Client/Server/ResourceBelongsToServer.php) |
| 路由组 | `throttle:api.client` | 速率限制：256 req/min/用户（或未认证按 IP） | [RouteServiceProvider.php:93-100](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Providers/RouteServiceProvider.php#L93-L100) |
| restore 额外 | `ResourceLimit::Backup->middleware()` | 备份资源限额（并发等） | [routes/api-client.php:138](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/routes/api-client.php#L138) |

### 6.2 AuthenticateServerAccess：用户身份 + 服务器状态校验

[AuthenticateServerAccess.php:29-67](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Http/Middleware/Api/Client/Server/AuthenticateServerAccess.php#L29-L67)

**用户身份准入（三者满足其一即可）**：
- `user.id === server.owner_id`（服务器所有者）
- `user.root_admin === true`（根管理员，全局 bypass）
- `server.subusers` 集合中存在匹配的 `user_id`（子用户）

若三者均不满足 → 返回 404（故意不返回 403，避免暴露服务器存在性）。

**服务器状态校验**（[Server.php:390-401](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Models/Server.php#L390-L401)）：
调用 `Server::validateCurrentState()`，任一为真则抛出 `ServerStateConflictException`：
- 服务器已暂停（`status === suspended`）
- 节点处于维护模式（`node.maintenance_mode === true`）
- 服务器未安装完成（`installing` 或 `install_failed`）
- 正在恢复备份（`status === restoring_backup`）
- 正在进行服务器迁移（`transfer` 关系非空）

### 6.3 ResourceBelongsToServer：跨服务器访问防护

[ResourceBelongsToServer.php:27-85](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Http/Middleware/Api/Client/Server/ResourceBelongsToServer.php#L27-L85)

对 Backup 模型的具体判断（[第 49-57 行](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Http/Middleware/Api/Client/Server/ResourceBelongsToServer.php#L49-L57)）：
```php
case Backup::class:
    if ($model->server_id !== $server->id) {
        throw new NotFoundHttpException('The requested resource was not found for this server.');
    }
    break;
```

**攻击场景防御**：请求 `/api/client/servers/{serverA_uuid}/backups/{backupB_uuid}`（backupB 物理上属于 serverB），即使通过了 AuthenticateServerAccess（用户确实有 serverA 的权限），此中间件也会因 id 不匹配返回 404。

### 6.4 控制器层：权限校验细节

#### download() 方法（[BackupController.php:167-185](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Http/Controllers/Api/Client/Servers/BackupController.php#L167-L185)）

```php
// 权限校验
if (!$request->user()->can(Permission::ACTION_BACKUP_DOWNLOAD, $server)) {
    throw new AuthorizationException();
}
// 磁盘驱动校验
if ($backup->disk !== Backup::ADAPTER_AWS_S3 && $backup->disk !== Backup::ADAPTER_WINGS) {
    throw new BadRequestHttpException('... unknown disk driver ...');
}
// 生成 URL
$url = $this->downloadLinkService->handle($backup, $request->user());
// 活动日志
Activity::event('server:backup.download')->subject($backup)->property('name', $backup->name)->log();
```

**注意**：此处没有 `StoreBackupRequest` / `RestoreBackupRequest` 那样的专门 FormRequest 类，权限是通过控制器内直接调用 `$user->can()` 校验的。

#### restore() 方法（[BackupController.php:198-229](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Http/Controllers/Api/Client/Servers/BackupController.php#L198-L229)）

权限校验委托给 `RestoreBackupRequest`：
- FormRequest 继承 `ClientApiRequest`
- `ClientApiRequest::authorize()` 会自动调用 `permission()` 方法，执行 `$user->can($request->permission(), $server)`
- [RestoreBackupRequest.php:10-13](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Http/Requests/Api/Client/Servers/Backups/RestoreBackupRequest.php#L10-L13) 返回 `Permission::ACTION_BACKUP_RESTORE`

两种路径的权限判定逻辑相同（[ServerPolicy.php](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Policies/ServerPolicy.php)）：
- 服务器所有者或根管理员 → **直接通过**（`before()` hook）
- 子用户 → 检查 `subuser.permissions` 数组中是否包含对应权限字符串

---

## 7. 签名 URL 生成机制（详细）

### 7.1 核心服务：DownloadLinkService

[DownloadLinkService.php](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Services/Backups/DownloadLinkService.php)

```php
public function handle(Backup $backup, User $user): string
{
    if ($backup->disk === Backup::ADAPTER_AWS_S3) {
        return $this->getS3BackupUrl($backup);
    }
    $token = $this->jwtService
        ->setExpiresAt(CarbonImmutable::now()->addMinutes(15))
        ->setUser($user)
        ->setClaims([
            'backup_uuid' => $backup->uuid,
            'server_uuid' => $backup->server->uuid,
        ])
        ->handle($backup->server->node, $user->id . $backup->server->uuid);

    return sprintf('%s/download/backup?token=%s',
        $backup->server->node->getConnectionAddress(),
        $token->toString()
    );
}
```

### 7.2 Wings 适配器：JWT 详细构造

#### 7.2.1 JWT 标准声明（由 NodeJWTService 生成）

[NodeJWTService.php:63-102](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Services/Nodes/NodeJWTService.php#L63-L102)

| 声明 | 生成代码 | 值来源 | 说明 |
|-----|---------|--------|------|
| `iss` | `->issuedBy(config('app.url'))` | Panel 的 `APP_URL` 环境变量 | 签发者标识 |
| `aud` | `->permittedFor($node->getConnectionAddress())` | `scheme://fqdn:daemonListen` | 受众：目标节点地址 |
| `jti` | `->identifiedBy($identifier)` | `md5($user->id . $server->uuid)` | JWT 唯一标识（确定性） |
| Header.`jti` | `->withHeader('jti', $identifier)` | 同上 | 同时写入 Header |
| `iat` | `->issuedAt(CarbonImmutable::now())` | 当前时间 | 签发时间 |
| `nbf` | `->canOnlyBeUsedAfter(now()->subMinutes(5))` | 当前时间 -5 分钟 | 允许 5 分钟时钟偏移 |
| `exp` | `->expiresAt($this->expiresAt)` | 当前时间 +15 分钟 | **过期时间** |
| `sub` / Header.`sub` | （备份下载场景不设置） | — | 未使用 |

#### 7.2.2 业务声明（备份下载场景专属）

| 声明 | 写入位置 | 作用 |
|-----|---------|------|
| `backup_uuid` | [DownloadLinkService.php:34](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Services/Backups/DownloadLinkService.php#L34) | 标识要下载的备份 |
| `server_uuid` | [DownloadLinkService.php:35](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Services/Backups/DownloadLinkService.php#L35) | 标识所属服务器 |
| `user_uuid` | [NodeJWTService.php:90](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Services/Nodes/NodeJWTService.php#L90) | 请求用户 UUID（审计） |
| `user_id` | [NodeJWTService.php:96](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Services/Nodes/NodeJWTService.php#L96) | 请求用户数字 ID（已弃用） |
| `unique_id` | [NodeJWTService.php:100](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Services/Nodes/NodeJWTService.php#L100) | `Str::random()` 随机唯一值 |

#### 7.2.3 签名算法与密钥

- **算法**：HS256（HMAC-SHA256，对称加密）
  - 由 `Configuration::forSymmetricSigner(new Sha256(), ...)` 指定
- **密钥**：每个 Wings 节点独立的 `daemon_token`（解密后原始值）
  - 存储：`nodes.daemon_token` 列，由 Laravel Encrypter（AES-256-CBC）加密
  - 获取：`$node->getDecryptedKey()` → 使用 `Encrypter::decrypt()` 解密
  - 长度：`Node::DAEMON_TOKEN_LENGTH = 64` 字符（创建时随机生成）

#### 7.2.4 最终 URL 结构示例

```
https://node1.example.com:8080/download/backup?token=
  eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.    ← JOSE Header (typ, alg, + jti)
  eyJpc3MiOiJodHRwczovL3BhbmVsLmV4YW1wbGUu  ← Payload
  Y29tIiwiYXVkIjoiaHR0cHM6Ly9ub2RlMS5leGFt
  cGxlLmNvbTo4MDgwIiwianRpIjoiZTNhZjk3...
  .SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c  ← HMAC-SHA256 Signature
```

### 7.3 S3 适配器：AWS 预签名 URL

生成逻辑：[DownloadLinkService.php:46-61](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Services/Backups/DownloadLinkService.php#L46-L61)

```php
protected function getS3BackupUrl(Backup $backup): string
{
    $adapter = $this->backupManager->adapter(Backup::ADAPTER_AWS_S3);
    $request = $adapter->getClient()->createPresignedRequest(
        $adapter->getClient()->getCommand('GetObject', [
            'Bucket'      => $adapter->getBucket(),
            'Key'         => sprintf('%s/%s.tar.gz', $backup->server->uuid, $backup->uuid),
            'ContentType' => 'application/x-gzip',
        ]),
        CarbonImmutable::now()->addMinutes(5)
    );
    return $request->getUri()->__toString();
}
```

S3 预签名 URL 参数详解（由 AWS SDK 生成，非本仓库代码可修改）：

| 参数 | 说明 |
|-----|------|
| `X-Amz-Algorithm` | 固定为 `AWS4-HMAC-SHA256` |
| `X-Amz-Credential` | `{access_key}/{date}/{region}/s3/aws4_request` |
| `X-Amz-Date` | ISO8601 格式签名时间 |
| `X-Amz-Expires` | 过期秒数，此处为 `300`（5 分钟） |
| `X-Amz-SignedHeaders` | 签名包含的 Header，通常为 `host` |
| `X-Amz-Signature` | HMAC-SHA256 签名（十六进制） |

**对象 Key 结构**：`{server_uuid}/{backup_uuid}.tar.gz`
- 例如：`abc123-def4-5678-90gh-ijklmnopqrst/1234abcd-12ab-34cd-56ef-1234567890ab.tar.gz`

---

## 8. 有效期与使用次数控制

### 8.1 有效期总结（所有备份相关签名 URL）

| URL 类型 | 有效期 | 配置方式 | 代码位置 |
|---------|--------|---------|---------|
| Wings 备份下载 JWT | **15 分钟** | **硬编码** | [DownloadLinkService.php:31](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Services/Backups/DownloadLinkService.php#L31) `addMinutes(15)` |
| S3 备份下载预签名 | **5 分钟** | **硬编码** | [DownloadLinkService.php:57](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Services/Backups/DownloadLinkService.php#L57) `addMinutes(5)` |
| Wings 文件下载 JWT | **15 分钟** | 硬编码 | [FileController.php:80](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Http/Controllers/Api/Client/Servers/FileController.php#L80) |
| Wings 文件上传 JWT | **15 分钟** | 硬编码 | [FileUploadController.php](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Http/Controllers/Api/Client/Servers/FileUploadController.php) |
| S3 分片上传预签名 | **60 分钟**（可配置） | `BACKUP_PRESIGNED_URL_LIFESPAN` 环境变量 | [config/backups.php:13](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/config/backups.php#L13) + [BackupRemoteUploadController.php:72](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Http/Controllers/Api/Remote/Backups/BackupRemoteUploadController.php#L72) |

### 8.2 ⚠️ 使用次数控制：完全无限制

**代码搜索结论**：Panel 仓库中不存在以下任何机制：
- Token 使用记录表（数据库或缓存中记录 `unique_id` / `jti` 是否已被消费）
- DownloadLinkService 返回 URL 前后的使用计数逻辑
- 与下载行为挂钩的 Activity 事件反向校验（记录了 `server:backup.download`，但不会回头阻止再次生成 URL）
- Wings 回源 Panel 校验"此 Token 是否已使用过"的 API（`/api/remote/*` 路由中不存在）

**结论**：
1. 同一用户（或任何获取到 URL 的第三方）在有效期内可以**无限次重复下载**
2. 用户可以每分钟生成 256 个不同的 URL（受 `throttle:api.client` 限制），每个 URL 都可独立使用 15/5 分钟
3. 若需防重放或使用次数限制，必须依赖 Wings 端实现（使用 `unique_id` 做短时效缓存），Panel 端目前完全不提供此能力

### 8.3 节流限制（仅限制 URL 生成频率）

`throttle:api.client` [配置](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/config/http.php#L14-L20)：
- 默认 256 请求 / 分钟 / 用户（按 `user.uuid` 限流）
- 未认证时按 IP 限流
- **仅限制 URL 生成请求数**，不限制已生成 URL 的使用

备份创建另有独立限流 [config/backups.php:29-32](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/config/backups.php#L29-L32)：
- 默认 10 分钟（600 秒）内最多创建 2 个备份（含失败/进行中）
- 与下载 URL 无关

---

## 9. 跨域 / 跨节点 / 跨用户访问防护

### 9.1 跨节点访问防护（可由代码结构确认）

**机制**：JWT 使用**节点专属密钥**签名。

证据链：
1. 每个节点有独立的 `daemon_token`（64 字符随机串），创建于 [NodeCreationService](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Services/Nodes/NodeCreationService.php) 或 [NodeUpdateService.php](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Services/Nodes/NodeUpdateService.php)（重置 token 时）
2. JWT 签名使用请求的 `$backup->server->node` 对应的密钥 [DownloadLinkService.php:37](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Services/Backups/DownloadLinkService.php#L37)
3. 节点 B 的密钥无法验证节点 A 密钥签名的 JWT（HS256 的对称性）
4. 即便攻击者手工构造（或从其他节点泄露）JWT，也会因 `aud` 声明不匹配而被 Wings 拒绝（基于合理推断）

**攻击面评估**：单个节点的 `daemon_token` 泄露 → 仅该节点上的备份面临风险，其他节点不受影响。

### 9.2 跨服务器访问防护（双重保障）

| 保障层 | 机制 | 代码位置 | 状态 |
|-------|------|---------|------|
| Panel 端（生成前） | `ResourceBelongsToServer` 强制 `backup.server_id === server.id` | [ResourceBelongsToServer.php:49-57](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Http/Middleware/Api/Client/Server/ResourceBelongsToServer.php#L49-L57) | ✅ 代码确认 |
| Wings 端（下载时） | JWT Claims 同时包含 `backup_uuid` 和 `server_uuid`，Wings 应根据本机服务器目录结构双重校验 | 推断（由 Claim 设计佐证） | ⚠️ 推断 |

此外，JWT 中绑定了 `server_uuid`，即使备份 UUID 被枚举（理论上 UUIDv4 不可枚举），也无法匹配错误的服务器目录。

### 9.3 跨用户访问防护

#### Panel 端（生成 URL 之前）——代码确认

1. `AuthenticateServerAccess`：仅所有者 / 管理员 / 子用户可到达控制器 [AuthenticateServerAccess.php:42-47](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Http/Middleware/Api/Client/Server/AuthenticateServerAccess.php#L42-L47)
2. `ServerPolicy`：子用户需显式拥有 `backup.download` 权限（所有者/管理员自动通过） [ServerPolicy.php](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Policies/ServerPolicy.php)

#### Wings 端（下载执行时）——推断 + 架构分析

Wings **没有** Panel 的用户数据库、Session 存储、API Key 表，它只能验证：
- ✅ JWT 是否由合法 Panel 签发（签名验证）
- ✅ JWT 是否过期（`exp`）
- ❌ 无法验证 `user_uuid` 对应的用户是否存在
- ❌ 无法验证用户是否仍对该服务器有权限（权限可能在 URL 生成后 15 分钟内被撤销）
- ❌ 无法验证发起下载请求的 IP / 浏览器是否为生成 URL 时的同一用户

**关键风险**：一旦签名 URL 泄露（通过日志、Referer、浏览器历史、中间人攻击[若 HTTPS 配置不当]等途径），任何持有 URL 的匿名第三方均可在有效期内完整下载备份文件。这是**当前设计的固有边界**，而非实现 Bug。

---

## 10. 备份相关远程 API（Wings → Panel 回源）

Wings 节点通过 `/api/remote/*` 回源 Panel，使用独立的认证机制（非 JWT，而是 `token_id.token` 格式）。此部分虽不直接涉及下载 URL，但有助于理解节点-面板信任模型。

### 10.1 DaemonAuthenticate：Wings 身份认证

[DaemonAuthenticate.php:34-66](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Http/Middleware/Api/Daemon/DaemonAuthenticate.php#L34-L66)

认证格式（HTTP Header）：
```
Authorization: Bearer {daemon_token_id}.{daemon_token}
```

验证流程：
1. 用 `daemon_token_id`（16 字符）在 `nodes` 表中查找节点
2. 解密找到的节点的 `daemon_token`（AES-256-CBC）
3. 用 `hash_equals` 时序安全比较解密结果与请求中的 token 值
4. 匹配成功 → 将 `$node` 写入 `$request->attributes`，供后续控制器使用

### 10.2 备份状态回传的节点归属校验

两个备份相关的回源控制器均做了节点归属检查：

[BackupRemoteUploadController.php:48-53](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Http/Controllers/Api/Remote/Backups/BackupRemoteUploadController.php#L48-L53)（S3 分片上传 URL 分配）：
```php
$server = $model->server;
if ($server->node_id !== $node->id) {
    throw new HttpForbiddenException('Requesting node does not have permission to access this server.');
}
```

[BackupStatusController.php](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Http/Controllers/Api/Remote/Backups/BackupStatusController.php)（备份完成/失败上报、恢复完成/失败上报）也使用了相同逻辑。

**防止的攻击场景**：已攻陷的恶意节点 A，使用合法的自身 `daemon_token` 认证后，尝试上报或操作属于节点 B 的服务器备份状态 → 被 `node_id` 比较拒绝。

### 10.3 回源路由与备份的对应关系

| 路由 | 方法 | 控制器 | 作用 |
|-----|------|--------|------|
| `/api/remote/backups/{backup}` | GET | BackupRemoteUploadController | Wings 请求 Panel 分配 S3 分片上传预签名 URL（仅 Wings→S3 上传场景） |
| `/api/remote/backups/{backup}` | POST | BackupStatusController::index | Wings 上报备份成功/失败状态 |
| `/api/remote/backups/{backup}/restore` | POST | BackupStatusController::restore | Wings 上报备份恢复完成/失败状态 |

---

## 11. 安全评审结论（修订版）

### 11.1 已实现的防护机制（代码确认 ✅）

| 防护 | 位置 | 说明 |
|-----|------|------|
| 用户身份认证 | `auth:sanctum` 中间件 | Session 或 Client API Key（不可用 Application Key） |
| 双因素认证强制 | `RequireTwoFactorAuthentication` | 用户若启用 2FA，API 请求也需 2FA 已通过 |
| IP 白名单 | `AuthenticateIPAccess` | 若用户配置，限制请求来源 IP |
| 服务器访问授权 | `AuthenticateServerAccess` | 所有者/管理员/子用户三级准入 + 服务器状态校验 |
| 资源归属绑定 | `ResourceBelongsToServer` | `backup.server_id === server.id`，防跨服务器枚举 |
| 细粒度下载权限 | `Permission::ACTION_BACKUP_DOWNLOAD` | 子用户需显式被授予此权限（所有者/管理员自动通过） |
| 细粒度恢复权限 | `Permission::ACTION_BACKUP_RESTORE` | FormRequest 自动校验 |
| URL 有效期（Wings） | JWT `exp` 声明 | 15 分钟过期 |
| URL 有效期（S3） | AWS `X-Amz-Expires` | 5 分钟过期 |
| 节点专属密钥 | HS256 每节点独立签名 | 跨节点 Token 无法验证 |
| Wings 回源认证 | `DaemonAuthenticate` | `token_id.token` 双字段，防冒充节点 |
| 回源节点归属 | 备份回源控制器 | 校验 `server.node_id === authenticated node.id` |
| URL 生成频率限制 | `throttle:api.client` | 256 次/分钟/用户 |
| Activity 审计日志 | 所有备份相关操作 | `server:backup.download` / `.restore` / `.start` 等事件记录至 `activity_logs` |

### 11.2 风险点与设计边界 ⚠️

| 风险 | 严重程度 | 说明 |
|-----|---------|------|
| **有效期内重复下载无限制** | 中高 | 无一次性 Token / 使用计数 / 撤销机制。15 分钟（Wings）或 5 分钟（S3）内可无限次使用 |
| **URL 泄露即匿名下载** | 高 | JWT 中 `user_uuid` 仅用于审计，不参与访问控制。任何持有者即可下载 |
| **权限撤销后 URL 仍有效** | 中 | URL 生成后，若用户被移除子用户或服务器被暂停，已发出的 URL 在过期前仍可使用（Wings 无法实时校验权限） |
| **备份状态未校验（download）** | 中低 | `BackupController::download()` 未校验 `is_successful` / `completed_at`，失败或进行中的备份也可生成下载链接（Wings 侧是否检查未知） |
| **Wings 下载端点无用户级校验** | 中 | 架构性限制：Wings 无用户数据库，无法做用户身份再校验 |
| **S3 预签名 URL 完全无身份上下文** | 中高 | S3 访问日志无法关联到 Panel 用户身份，无法审计具体谁执行了下载 |
| **`jti` 确定性生成** | 低 | `md5(userId+serverUuid)` 可预测，但 `unique_id` 提供了不可预测性 |
| **硬编码有效期** | 低 | 15 分钟和 5 分钟不可通过配置调整（仅 S3 上传 URL 可配） |
| **`window.location` 跳转 Referer 泄露** | 低 | 若 Panel 页面含外部资源或跳转链，可能泄露完整 URL（含 Token）。可通过配置 Referrer-Policy Header 缓解 |

### 11.3 签名 URL 使用边界声明

签名 URL 的安全边界由以下 5 个维度共同定义，边界之内的访问由系统保障，边界之外不提供保护：

| 维度 | 边界值 | 说明 |
|-----|--------|------|
| **时间边界** | Wings: 15 min / S3: 5 min | 过期后自动失效，无法续期。过期时间硬编码，不可按用户/服务器差异化配置 |
| **节点边界** | JWT `aud` + 节点独立密钥 | 仅目标 Wings 节点可验证。跨节点 Token 无法通过 HS256 签名校验 |
| **服务器边界** | JWT Claims 绑定 `backup_uuid` + `server_uuid` | 不可跨服务器使用（Wings 端校验） + Panel 端 `ResourceBelongsToServer` 双重保障 |
| **次数边界** | 无限制 | 有效期内可重复使用。无一次性 Token / 使用计数 / 撤销机制 |
| **用户身份边界** | 仅在生成时校验 | URL 生成后不承载用户级访问控制。`user_uuid` 仅用于审计日志，泄露后任何第三方均可使用 |

---

## 12. 文件索引

| 组件 | 文件路径 |
|-----|---------|
| 备份控制器（下载/恢复核心逻辑） | [BackupController.php](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Http/Controllers/Api/Client/Servers/BackupController.php) |
| 签名 URL 生成服务 | [DownloadLinkService.php](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Services/Backups/DownloadLinkService.php) |
| 节点 JWT 签发服务 | [NodeJWTService.php](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Services/Nodes/NodeJWTService.php) |
| 恢复备份权限 FormRequest | [RestoreBackupRequest.php](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Http/Requests/Api/Client/Servers/Backups/RestoreBackupRequest.php) |
| Client API Request 基类（权限自动校验） | [ClientApiRequest.php](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Http/Requests/Api/Client/ClientApiRequest.php) |
| 服务器访问授权中间件 | [AuthenticateServerAccess.php](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Http/Middleware/Api/Client/Server/AuthenticateServerAccess.php) |
| 资源归属校验中间件 | [ResourceBelongsToServer.php](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Http/Middleware/Api/Client/Server/ResourceBelongsToServer.php) |
| 路由模型绑定（Client） | [SubstituteClientBindings.php](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Http/Middleware/Api/Client/SubstituteClientBindings.php) |
| 禁止 Application Key 中间件 | [RequireClientApiKey.php](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Http/Middleware/Api/Client/RequireClientApiKey.php) |
| 服务器权限策略 | [ServerPolicy.php](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Policies/ServerPolicy.php) |
| 权限常量定义（含 backup.download） | [Permission.php](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Models/Permission.php) |
| 节点模型（密钥/连接地址） | [Node.php](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Models/Node.php) |
| 备份模型（disk / adapter 等） | [Backup.php](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Models/Backup.php) |
| 服务器模型（状态校验等） | [Server.php](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Models/Server.php) |
| 备份配置（throttle、S3 等） | [backups.php](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/config/backups.php) |
| HTTP 限流配置 | [http.php](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/config/http.php) |
| Wings → Panel 认证中间件 | [DaemonAuthenticate.php](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Http/Middleware/Api/Daemon/DaemonAuthenticate.php) |
| 备份 S3 上传预签名分配 | [BackupRemoteUploadController.php](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Http/Controllers/Api/Remote/Backups/BackupRemoteUploadController.php) |
| 备份状态回传（含节点归属校验） | [BackupStatusController.php](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Http/Controllers/Api/Remote/Backups/BackupStatusController.php) |
| 客户端 API 路由（备份部分） | [routes/api-client.php](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/routes/api-client.php#L132-L141) |
| 路由服务提供（中间件挂载） | [RouteServiceProvider.php](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Providers/RouteServiceProvider.php) |
| 文件下载控制器（对比参考） | [FileController.php](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Http/Controllers/Api/Client/Servers/FileController.php) |
| Websocket 控制器（JWT permissions 对比） | [WebsocketController.php](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/app/Http/Controllers/Api/Client/Servers/WebsocketController.php) |
| 备份授权测试（含 download） | [BackupAuthorizationTest.php](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/tests/Integration/Api/Client/Server/Backup/BackupAuthorizationTest.php) |
| 前端：备份右键菜单（下载按钮） | [BackupContextMenu.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/resources/scripts/components/server/backups/BackupContextMenu.tsx) |
| 前端：备份行（完成状态判断） | [BackupRow.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/resources/scripts/components/server/backups/BackupRow.tsx) |
| 前端：下载 URL API 封装 | [getBackupDownloadUrl.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/resources/scripts/api/server/backups/getBackupDownloadUrl.ts) |
| 前端：恢复 API 封装 | [backups/index.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/resources/scripts/api/server/backups/index.ts) |
| 前端：权限组件（Can） | [Can.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/209-panel/resources/scripts/components/elements/Can.tsx) |