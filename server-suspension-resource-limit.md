# Pterodactyl Panel 服务器挂起（Suspension）资源限制说明

> **重要说明**：本文档严格区分 **✅ 代码事实**（Panel 代码中可直接验证的逻辑）与 **⚠️ Wings 推断**（依赖 Wings 守护进程行为，Panel 代码无法直接验证的部分）。客服请特别注意区分。

---

## 一、概述

服务器挂起是 Pterodactyl Panel 中用于临时禁用服务器的机制，通常用于商户欠费、违规操作等场景。

---

## 二、挂起触发条件与入口

### 2.1 触发入口 ✅ 代码事实

挂起操作可通过以下三种方式触发：

| 触发方式 | 位置 | 说明 |
|---------|------|------|
| 管理后台界面 | [manage.blade.php](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/resources/views/admin/servers/view/manage.blade.php#L57-L93) | 管理员在服务器管理页面点击"Suspend Server"按钮 |
| 应用 API（挂起） | [ServerManagementController.php](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/app/Http/Controllers/Api/Application/Servers/ServerManagementController.php#L29-L34) | `POST /api/application/servers/{server}/suspend` |
| 应用 API（解除挂起） | [ServerManagementController.php](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/app/Http/Controllers/Api/Application/Servers/ServerManagementController.php#L41-L46) | `POST /api/application/servers/{server}/unsuspend` |

### 2.2 前置条件检查 ✅ 代码事实

在 [SuspensionService::toggle()](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/app/Services/Servers/SuspensionService.php#L28-L60) 中执行以下检查：

1. **状态去重检查**：如果服务器已经是目标状态（已挂起时再次挂起，或未挂起时解除挂起），直接返回，不执行任何操作
2. **转移状态检查**：如果服务器正在转移中（`$server->transfer` 不为 null），抛出 `ConflictHttpException` 异常

```php
// 核心逻辑片段（SuspensionService.php 第 36-43 行）
if ($isSuspending === $server->isSuspended()) {
    return;
}

if (!is_null($server->transfer)) {
    throw new ConflictHttpException('Cannot toggle suspension status on a server that is currently being transferred.');
}
```

---

## 三、状态变更完整流程

### 3.1 数据库状态字段 ✅ 代码事实

服务器状态存储在 `servers.status` 字段中。挂起状态为常量 `STATUS_SUSPENDED = 'suspended'`。

- 正常运行状态：`status = null`
- 挂起状态：`status = 'suspended'`

相关代码见 [Server.php](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/app/Models/Server.php#L122-L126) 第 125 行。

### 3.2 完整执行流程 ✅ 代码事实

```
1. 更新数据库状态（Panel 代码）
   ↓
2. 向 Wings 发送 sync 触发请求（空 POST，Panel 代码）
   ↓
3. ⚠️ Wings 收到 sync 后主动拉取配置（推断）
   ↓
4. Panel 返回包含 suspended 字段的配置（Panel 代码）
   ↓
5. ⚠️ Wings 执行挂起操作（停止容器等）（推断）
   ↓
6. 成功 → 流程完成
   失败 → 回滚数据库状态 → 抛出异常（Panel 代码）
```

**步骤 1：更新数据库状态** ✅ 代码事实  
[SuspensionService.php 第 46-48 行](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/app/Services/Servers/SuspensionService.php#L46-L48)

```php
$server->update([
    'status' => $isSuspending ? Server::STATUS_SUSPENDED : null,
]);
```

**步骤 2：向 Wings 发送 sync 触发请求** ✅ 代码事实  
[SuspensionService.php 第 52 行](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/app/Services/Servers/SuspensionService.php#L52)

```php
$this->daemonServerRepository->setServer($server)->sync();
```

**关键细节** ✅ 代码事实：  
[DaemonServerRepository::sync()](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/app/Repositories/Wings/DaemonServerRepository.php#L63-L72) 只发送一个**空的 POST 请求**，没有请求体：

```php
public function sync(): void
{
    $this->getHttpClient()->post("/api/servers/{$this->server->uuid}/sync");
}
```

**步骤 3：Wings 主动拉取配置** ⚠️ Wings 推断  
Wings 收到 sync 请求后，会调用 `GET /api/remote/servers/{uuid}` 从 Panel 拉取最新配置。

**步骤 4：Panel 返回配置** ✅ 代码事实  
[ServerDetailsController::__invoke()](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/app/Http/Controllers/Api/Remote/Servers/ServerDetailsController.php#L38-L60) 返回包含 `suspended` 字段的配置：

```php
return new JsonResponse([
    'settings' => $this->configurationStructureService->handle($server),
    'process_configuration' => $this->eggConfigurationService->handle($server),
]);
```

配置结构中包含 `suspended` 字段 ✅ 代码事实：  
[ServerConfigurationStructureService.php 第 51 行](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/app/Services/Servers/ServerConfigurationStructureService.php#L51)

```json
{
  "uuid": "...",
  "suspended": true,
  "environment": { ... },
  "build": { ... },
  ...
}
```

**步骤 5：Wings 执行挂起操作** ⚠️ Wings 推断  
Wings 收到 `suspended: true` 后的具体行为无法在 Panel 代码中验证，根据设计推断会：
- 停止服务器容器的运行
- 禁止服务器启动
- 禁用网络连接
- 保留磁盘上的所有数据

**步骤 6：失败回滚** ✅ 代码事实  
[SuspensionService.php 第 53-59 行](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/app/Services/Servers/SuspensionService.php#L53-L59)

```php
try {
    $this->daemonServerRepository->setServer($server)->sync();
} catch (\Exception $exception) {
    // 回滚数据库状态
    $server->update([
        'status' => $isSuspending ? null : Server::STATUS_SUSPENDED,
    ]);
    throw $exception;
}
```

> **设计要点** ✅ 代码事实：先更新数据库，再同步 Wings。如果 Wings 同步失败，回滚数据库状态，保证 Panel 与 Wings 状态一致性。

---

## 四、挂起期间的资源限制

### 4.1 网络限制

| 项目 | 挂起前 | 挂起后 | 验证类型 | 代码依据 |
|-----|-------|-------|---------|---------|
| 服务器运行 | 可运行 | 停止运行 | ⚠️ 推断 | Wings 停止容器 |
| 网络连接 | 正常 | 断开 | ⚠️ 推断 | Wings 禁用网络 |
| Websocket 连接 | 可连接 | 关闭连接（错误码 4409） | ✅ 事实 | [Websocket.ts 第 40-47 行](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/resources/scripts/plugins/Websocket.ts#L40-L47) |

**Websocket 关闭处理** ✅ 代码事实：
```typescript
// Wings 返回代码 4409 表示服务器已挂起
// 代码 4400 为预留扩展码
if (evt.code === 4409 || evt.code === 4400) {
    this.close(1000);
}
```

> **说明**：错误码 4409 是 Panel 前端代码中硬编码处理的，但具体由谁返回、何时返回，属于 Wings 行为，Panel 代码无法验证。

### 4.2 存储（文件）限制

| 操作 | 挂起前 | 挂起后 | 验证类型 | 代码依据 |
|-----|-------|-------|---------|---------|
| 文件浏览 | 可访问 | 不可访问 | ✅ 事实 | API 中间件拦截 |
| 文件上传/下载 | 可操作 | 不可操作 | ✅ 事实 | API 中间件拦截 |
| SFTP 访问 | 可连接 | 拒绝连接 | ✅ 事实 | [SftpAuthenticationController.php 第 153 行](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/app/Http/Controllers/Api/Remote/SftpAuthenticationController.php#L153) |
| 数据保留 | - | 完整保留 | ✅ 事实 | 挂起逻辑中无删除操作 |

**API 中间件拦截** ✅ 代码事实：  
所有 `/api/client/servers/{server}/files/*` 路由都经过 [AuthenticateServerAccess](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/app/Http/Middleware/Api/Client/Server/AuthenticateServerAccess.php) 中间件，挂起时被拦截。

**重要** ✅ 代码事实：  
[FileController.php](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/app/Http/Controllers/Api/Client/Servers/FileController.php) 本身**没有**额外的挂起检查，完全依赖中间件拦截。

**SFTP 认证校验** ✅ 代码事实：
```php
// validateSftpAccess 方法中调用
$server->validateCurrentState();
```

> **说明**：SFTP 拒绝是在 Panel 端认证时就拒绝的，不是由 Wings 拒绝的。

### 4.3 备份限制

| 操作 | 挂起前 | 挂起后 | 验证类型 | 代码依据 |
|-----|-------|-------|---------|---------|
| 手动创建备份 | 可创建 | 不可创建 | ✅ 事实 | API 中间件拦截 |
| 查看备份列表 | 可查看 | 不可查看 | ✅ 事实 | API 中间件拦截 |
| 下载备份 | 可下载 | 不可下载 | ✅ 事实 | API 中间件拦截 |
| 恢复备份 | 可恢复 | 不可恢复 | ✅ 事实 | 中间件 + 控制器双重检查 |
| 定时备份任务 | 可执行 | 跳过执行 | ✅ 事实 | [RunTaskJob.php 第 53-57 行](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/app/Jobs/Schedule/RunTaskJob.php#L53-L57) |
| 已存在备份 | - | 完整保留 | ✅ 事实 | 挂起逻辑中无删除操作 |

**定时任务跳过逻辑** ✅ 代码事实：
```php
// 如果服务器状态不为 null（包括 suspended），任务直接失败退出
if (!is_null($server->status)) {
    $this->failed();
    return;
}
```

> **注意** ✅ 代码事实：挂起期间定时任务会标记为失败，但不会产生错误日志，也不会重试。调度任务的 `is_processing` 状态会被重置为 false。

**备份恢复双重检查** ✅ 代码事实：  
[BackupController::restore()](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/app/Http/Controllers/Api/Client/Servers/BackupController.php#L198-L229) 中有额外的状态检查：

```php
// 即使绕过中间件，这里也会检查
if (!is_null($server->status)) {
    throw new BadRequestHttpException('This server is not currently in a state that allows for a backup to be restored.');
}
```

### 4.4 计费相关

Panel 代码本身**不包含计费逻辑** ✅ 代码事实。挂起状态（`status = 'suspended'`）可作为外部计费系统的判断依据：

- `status` 字段可通过 API 查询 ✅ 事实
- 可通过挂起时间计算暂停计费时长 ✅ 事实
- 备份、数据库等资源在挂起期间仍占用存储，是否计费由业务系统决定 ⚠️ 业务决策

---

## 五、API 访问限制

### 5.1 客户端 API 中间件 ✅ 代码事实

客户端 API 的所有服务器路由都经过 [AuthenticateServerAccess](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/app/Http/Middleware/Api/Client/Server/AuthenticateServerAccess.php) 中间件。

中间件调用 `$server->validateCurrentState()` 进行状态校验。

### 5.2 状态校验核心方法 ✅ 代码事实

[Server::validateCurrentState()](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/app/Models/Server.php#L390-L401) 是服务器状态校验的核心方法：

```php
public function validateCurrentState()
{
    if (
        $this->isSuspended()
        || $this->node->isUnderMaintenance()
        || !$this->isInstalled()
        || $this->status === self::STATUS_RESTORING_BACKUP
        || !is_null($this->transfer)
    ) {
        throw new ServerStateConflictException($this);
    }
}
```

该方法在以下位置被调用 ✅ 代码事实：
- 客户端 API 中间件
- SFTP 认证控制器
- 备份恢复控制器（额外检查）

### 5.3 允许访问的接口（白名单）✅ 代码事实

挂起状态下，以下客户端 API 接口**可以访问**：

| 路由名称 | 端点 | 说明 |
|---------|------|------|
| `api:client:server.view` | `GET /api/client/servers/{server}` | 查看服务器基本信息 |
| `api:client:server.resources` | `GET /api/client/servers/{server}/resources` | 查看服务器资源使用情况 |

**资源使用接口** ✅ 代码事实：  
[ResourceUtilizationController](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/app/Http/Controllers/Api/Client/Servers/ResourceUtilizationController.php) 从 Wings 获取实时资源数据，缓存 20 秒。

**中间件逻辑** ✅ 代码事实（[AuthenticateServerAccess.php 第 49-62 行](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/app/Http/Middleware/Api/Client/Server/AuthenticateServerAccess.php#L49-L62)）：

```php
try {
    $server->validateCurrentState();
} catch (ServerStateConflictException $exception) {
    // 例外 1：允许查看服务器基本信息（所有异常状态下）
    if (!$request->routeIs('api:client:server.view')) {
        // 例外 2：挂起或节点维护时，还允许访问资源使用接口
        if (($server->isSuspended() || $server->node->isUnderMaintenance()) 
            && !$request->routeIs('api:client:server.resources')) {
            throw $exception;
        }
        // 例外 3：管理员可以访问 websocket 接口
        if (!$user->root_admin || !$request->routeIs($this->except)) {
            throw $exception;
        }
    }
}
```

### 5.4 管理员例外 ✅ 代码事实

管理员（`root_admin = true`）可以访问 Websocket 接口（`api:client:server.ws`），即使服务器挂起。

见 [AuthenticateServerAccess.php 第 15-17 行](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/app/Http/Middleware/Api/Client/Server/AuthenticateServerAccess.php#L15-L17) 和第 58 行。

### 5.5 禁止访问的接口 ✅ 代码事实

除上述白名单外，所有其他客户端 API 接口均被拒绝，返回 `409 Conflict` 错误，错误消息为：

> "This server is currently suspended and the functionality requested is unavailable."

错误定义见 [ServerStateConflictException.php](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/app/Exceptions/Http/Server/ServerStateConflictException.php)。

被禁止的操作包括但不限于 ✅ 代码事实：
- 启动/停止/重启服务器（[PowerController.php](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/app/Http/Controllers/Api/Client/Servers/PowerController.php)）
- 发送控制台命令（[CommandController.php](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/app/Http/Controllers/Api/Client/Servers/CommandController.php)）
- 文件管理（浏览、上传、下载、删除等）
- 数据库管理
- 调度任务管理
- 子用户管理
- 网络分配管理
- 备份管理
- 启动参数修改
- 服务器设置修改

> **重要** ✅ 代码事实：以上控制器本身**都没有**额外的挂起检查，完全依赖中间件拦截。只有备份恢复控制器有双重检查。

---

## 六、客户端界面表现 ✅ 代码事实

### 6.1 服务器列表页

在 [ServerRow.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/resources/scripts/components/dashboard/ServerRow.tsx) 中：

- 显示红色 "Suspended" 标签
- 不请求资源使用数据（避免不必要的 HTTP 请求）
- 服务器仍可点击进入详情页

```typescript
// 挂起时不轮询资源使用数据
if (isSuspended) return;
```

### 6.2 服务器详情页

在 [ConflictStateRenderer.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/resources/scripts/components/server/ConflictStateRenderer.tsx) 中：

- 显示全屏遮罩
- 标题："Server Suspended"
- 提示信息："This server is suspended and cannot be accessed."
- 使用 `ServerErrorSvg` 错误图标

用户无法看到控制台、文件管理等任何功能页面。

---

## 七、解除挂起（Unsuspend）恢复路径

### 7.1 解除挂起流程 ✅ 代码事实

解除挂起的流程与挂起流程完全相同，只是状态变化方向相反：

```
status = 'suspended' → status = null
```

同样通过 [SuspensionService::toggle()](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/app/Services/Servers/SuspensionService.php#L28-L60) 方法，传入 `ACTION_UNSUSPEND` 参数。

### 7.2 恢复后的行为

| 行为 | 状态 | 验证类型 | 说明 |
|-----|------|---------|------|
| 服务器状态恢复为正常 | ✅ 事实 | 代码 | `status = null` |
| Wings 同步后可启动 | ⚠️ 推断 | Wings | Panel 只发 sync 请求，实际行为由 Wings 决定 |
| API 接口恢复访问 | ✅ 事实 | 代码 | 中间件不再拦截 |
| SFTP 恢复正常 | ✅ 事实 | 代码 | 认证不再拒绝 |
| 服务器自动启动 | ❌ 不会 | ✅ 事实 | Panel 代码中没有自动启动逻辑 |
| 调度任务恢复执行 | ✅ 事实 | 代码 | 状态为 null 后任务不再被跳过 |
| 备份功能恢复正常 | ✅ 事实 | 代码 | 中间件不再拦截 |

> **客服要点** ✅ 代码事实：解除挂起后，服务器**不会自动启动**，需要用户手动启动。

### 7.3 安装完成时的挂起状态保持 ✅ 代码事实

如果服务器在安装期间被挂起，安装完成后会**保持挂起状态**，不会自动解除。

见 [ServerInstallController.php 第 71-74 行](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/app/Http/Controllers/Api/Remote/Servers/ServerInstallController.php#L71-L74)：

```php
// Keep the server suspended if it's already suspended
if ($server->status === Server::STATUS_SUSPENDED) {
    $status = Server::STATUS_SUSPENDED;
}
```

### 7.4 回滚路径 ✅ 代码事实

当 Wings sync 失败时，Panel 会回滚数据库状态：

- 挂起失败：`status = 'suspended'` → `status = null`
- 解除挂起失败：`status = null` → `status = 'suspended'`

见 [SuspensionService.php 第 53-59 行](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/app/Services/Servers/SuspensionService.php#L53-L59)。

---

## 八、API 返回字段 ✅ 代码事实

### 8.1 客户端 API

在 [ServerTransformer.php](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/app/Transformers/Api/Client/ServerTransformer.php) 中，挂起相关字段：

| 字段 | 类型 | 说明 |
|-----|------|------|
| `status` | string\|null | 服务器状态，挂起时为 `'suspended'` |
| `is_suspended` | boolean | **已废弃**，请使用 `status` 字段 |
| `is_installing` | boolean | **已废弃**，请使用 `status` 字段 |
| `is_transferring` | boolean | 是否在转移中 |

### 8.2 资源使用 API

在 [StatsTransformer.php](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/app/Transformers/Api/Client/StatsTransformer.php) 中：

| 字段 | 类型 | 说明 |
|-----|------|------|
| `current_state` | string | 服务器电源状态（running/stopped 等） |
| `is_suspended` | boolean | 是否挂起 |
| `resources` | object | 资源使用数据（内存、CPU、磁盘、网络） |

> **注意**：`is_suspended` 字段的值由 Wings 返回，Panel 只是透传。

### 8.3 应用 API（管理员）

在 [Application/ServerTransformer.php](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/app/Transformers/Api/Application/ServerTransformer.php) 中：

| 字段 | 类型 | 说明 |
|-----|------|------|
| `status` | string\|null | 服务器状态 |
| `suspended` | boolean | **已废弃**，请使用 `status` 字段 |

---

## 九、事实与推断汇总表

| 环节 | 代码事实（Panel 可验证） | Wings 推断（Panel 不可验证） |
|-----|-------------------------|-----------------------------|
| **状态变更** | 更新数据库 status 字段、发送 sync 请求、失败回滚 | Wings 停止容器、禁用网络 |
| **网络限制** | Websocket 错误码 4409 处理、前端不请求资源数据 | 实际停止容器、断开网络 |
| **文件限制** | API 中间件拦截、SFTP 认证拒绝 | Wings 端是否有额外限制 |
| **备份限制** | API 中间件拦截、定时任务跳过、恢复接口双重检查 | -（全部可验证） |
| **API 白名单** | 只有 view 和 resources 可访问、管理员 websocket 例外 | -（全部可验证） |
| **解除挂起** | 状态反向变更、不自动启动、安装时保持挂起 | Wings 恢复可启动状态 |

---

## 十、关键代码文件索引 ✅ 代码事实

| 文件 | 作用 |
|-----|------|
| [SuspensionService.php](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/app/Services/Servers/SuspensionService.php) | 挂起/解除挂起核心服务 |
| [Server.php](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/app/Models/Server.php) | 服务器模型，状态常量和校验方法 |
| [DaemonServerRepository.php](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/app/Repositories/Wings/DaemonServerRepository.php) | Wings 通信，sync 方法 |
| [ServerConfigurationStructureService.php](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/app/Services/Servers/ServerConfigurationStructureService.php) | 生成发给 Wings 的配置结构 |
| [ServerDetailsController.php](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/app/Http/Controllers/Api/Remote/Servers/ServerDetailsController.php) | Wings 拉取配置的接口 |
| [AuthenticateServerAccess.php](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/app/Http/Middleware/Api/Client/Server/AuthenticateServerAccess.php) | 客户端 API 访问控制中间件 |
| [ServerStateConflictException.php](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/app/Exceptions/Http/Server/ServerStateConflictException.php) | 状态冲突异常 |
| [RunTaskJob.php](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/app/Jobs/Schedule/RunTaskJob.php) | 定时任务执行（挂起时跳过） |
| [SftpAuthenticationController.php](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/app/Http/Controllers/Api/Remote/SftpAuthenticationController.php) | SFTP 认证（挂起时拒绝） |
| [BackupController.php](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/app/Http/Controllers/Api/Client/Servers/BackupController.php) | 备份管理（恢复时有双重检查） |
| [ConflictStateRenderer.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/resources/scripts/components/server/ConflictStateRenderer.tsx) | 前端挂起状态展示组件 |
| [Websocket.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/resources/scripts/plugins/Websocket.ts) | Websocket 挂起断开处理 |

---

## 十一、客服问答速查 ✅ 代码事实

**Q: 服务器挂起后数据会丢失吗？**  
A: 不会。挂起只是停止服务器运行并禁用访问，所有数据（包括文件、数据库、备份）都会完整保留。Panel 代码中没有任何删除数据的逻辑。

**Q: 挂起期间还计费吗？**  
A: Panel 本身不处理计费逻辑。挂起状态（`status = 'suspended'`）可作为计费系统的参考，具体是否暂停计费由业务系统决定。

**Q: 挂起后用户能做什么？**  
A: 用户只能查看服务器基本信息和资源使用情况，不能进行任何操作（启动、文件管理、备份等均不可用）。这是由 API 中间件严格控制的。

**Q: 解除挂起后服务器会自动启动吗？**  
A: 不会。解除挂起只是恢复服务器的可操作状态，用户需要手动启动服务器。Panel 代码中没有自动启动的逻辑。

**Q: 挂起期间定时备份还会执行吗？**  
A: 不会。挂起期间所有定时任务都会被跳过，代码中明确检查 `!is_null($server->status)` 就直接返回。

**Q: 挂起的服务器还占不占资源？**  
A: 服务器进程会被停止（Wings 行为，推断），不占用 CPU 和内存。但磁盘空间（包括服务器文件和备份）仍然被占用。

**Q: SFTP 还能连接吗？**  
A: 不能。挂起状态下 SFTP 认证会在 Panel 端就被拒绝，调用 `validateCurrentState()` 时抛出异常。

**Q: 挂起操作失败了怎么办？**  
A: Panel 代码有回滚机制，如果 Wings 同步失败，会自动把数据库状态恢复到操作前的状态，并抛出异常。不会出现状态不一致的情况。
