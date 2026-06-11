# Pterodactyl Panel 服务器挂起（Suspension）资源限制说明

## 一、概述

服务器挂起是 Pterodactyl Panel 中用于临时禁用服务器的机制，通常用于商户欠费、违规操作等场景。挂起期间服务器停止运行，用户无法通过面板或 API 进行大部分操作，但数据保留在节点上。

---

## 二、挂起触发条件与入口

### 2.1 触发入口

挂起操作可通过以下三种方式触发：

| 触发方式 | 位置 | 说明 |
|---------|------|------|
| 管理后台界面 | [manage.blade.php](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/resources/views/admin/servers/view/manage.blade.php#L57-L93) | 管理员在服务器管理页面点击"Suspend Server"按钮 |
| 应用 API | [ServerManagementController.php](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/app/Http/Controllers/Api/Application/Servers/ServerManagementController.php#L29-L34) | `POST /api/application/servers/{server}/suspend` |
| 解除挂起 API | [ServerManagementController.php](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/app/Http/Controllers/Api/Application/Servers/ServerManagementController.php#L41-L46) | `POST /api/application/servers/{server}/unsuspend` |

### 2.2 前置条件检查

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

## 三、状态变更流程

### 3.1 数据库状态字段

服务器状态存储在 `servers.status` 字段中。挂起状态为常量 `STATUS_SUSPENDED = 'suspended'`。

- 正常运行状态：`status = null`
- 挂起状态：`status = 'suspended'`

相关代码见 [Server.php](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/app/Models/Server.php#L122-L126) 第 125 行。

### 3.2 状态变更流程

```
1. 更新数据库状态
   ↓
2. 向 Wings 发送同步指令
   ↓
3. 成功 → 完成
   失败 → 回滚数据库状态 → 抛出异常
```

**步骤 1：更新数据库状态**（[SuspensionService.php 第 46-48 行](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/app/Services/Servers/SuspensionService.php#L46-L48)）

```php
$server->update([
    'status' => $isSuspending ? Server::STATUS_SUSPENDED : null,
]);
```

**步骤 2：向 Wings 发送同步指令**（[SuspensionService.php 第 52 行](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/app/Services/Servers/SuspensionService.php#L52)）

```php
$this->daemonServerRepository->setServer($server)->sync();
```

**步骤 3：失败回滚**（[SuspensionService.php 第 53-59 行](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/app/Services/Servers/SuspensionService.php#L53-L59)）

```php
try {
    $this->daemonServerRepository->setServer($server)->sync();
} catch (\Exception $exception) {
    $server->update([
        'status' => $isSuspending ? null : Server::STATUS_SUSPENDED,
    ]);
    throw $exception;
}
```

> **设计要点**：先更新数据库，再同步 Wings。如果 Wings 同步失败，回滚数据库状态，保证 Panel 与 Wings 状态一致性。

---

## 四、Wings 守护进程交互

### 4.1 Sync API 接口

Wings 通过 `POST /api/servers/{uuid}/sync` 接口接收同步指令。

代码见 [DaemonServerRepository::sync()](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/app/Repositories/Wings/DaemonServerRepository.php#L63-L72)。

### 4.2 同步时发送的配置结构

Wings 同步时，Panel 会发送完整的服务器配置，其中包含 `suspended` 字段。

当前格式（[ServerConfigurationStructureService.php 第 51 行](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/app/Services/Servers/ServerConfigurationStructureService.php#L51)）：
```json
{
  "uuid": "...",
  "suspended": true,
  "environment": { ... },
  "build": { ... },
  "allocations": { ... },
  ...
}
```

旧版兼容格式（[ServerConfigurationStructureService.php 第 127 行](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/app/Services/Servers/ServerConfigurationStructureService.php#L127)）：
```json
{
  "suspended": 1,
  "build": { ... },
  ...
}
```

### 4.3 Wings 端挂起行为（推断）

根据代码推断，Wings 收到 `suspended: true` 后会：
- 停止服务器容器的运行
- 禁止服务器启动
- 禁用网络连接
- 保留磁盘上的所有数据

---

## 五、挂起期间的资源限制

### 5.1 网络限制

| 项目 | 挂起前 | 挂起后 | 代码依据 |
|-----|-------|-------|---------|
| 服务器运行 | 可运行 | 停止运行 | Wings 停止容器 |
| 网络连接 | 正常 | 断开 | Wings 禁用网络 |
| Websocket 连接 | 可连接 | 关闭连接（错误码 4409） | [Websocket.ts 第 40-47 行](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/resources/scripts/plugins/Websocket.ts#L40-L47) |

**Websocket 关闭处理**：
```typescript
// Wings 返回代码 4409 表示服务器已挂起
if (evt.code === 4409 || evt.code === 4400) {
    this.close(1000);
}
```

### 5.2 存储（文件）限制

| 操作 | 挂起前 | 挂起后 | 说明 |
|-----|-------|-------|------|
| 文件浏览 | 可访问 | 不可访问 | API 中间件拦截 |
| 文件上传/下载 | 可操作 | 不可操作 | API 中间件拦截 |
| SFTP 访问 | 可连接 | 拒绝连接 | [SftpAuthenticationController.php 第 153 行](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/app/Http/Controllers/Api/Remote/SftpAuthenticationController.php#L153) |
| 数据保留 | - | 完整保留 | 挂起不删除数据 |

**SFTP 认证校验**：
```php
// validateSftpAccess 方法中调用
$server->validateCurrentState();
```

### 5.3 备份限制

| 操作 | 挂起前 | 挂起后 | 代码依据 |
|-----|-------|-------|---------|
| 手动创建备份 | 可创建 | 不可创建 | API 中间件拦截 |
| 查看备份列表 | 可查看 | 不可查看 | API 中间件拦截 |
| 下载备份 | 可下载 | 不可下载 | API 中间件拦截 |
| 恢复备份 | 可恢复 | 不可恢复 | API 中间件拦截 |
| 定时备份任务 | 可执行 | 跳过执行 | [RunTaskJob.php 第 53-57 行](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/app/Jobs/Schedule/RunTaskJob.php#L53-L57) |
| 已存在备份 | - | 完整保留 | 挂起不删除备份 |

**定时任务跳过逻辑**：
```php
// 如果服务器状态不为 null（包括 suspended），任务直接失败退出
if (!is_null($server->status)) {
    $this->failed();
    return;
}
```

> **注意**：挂起期间定时任务会标记为失败，但不会产生错误日志，也不会重试。调度任务的 `is_processing` 状态会被重置为 false。

### 5.4 计费相关

Panel 代码本身**不包含计费逻辑**。挂起状态（`status = 'suspended'`）可作为外部计费系统的判断依据：

- `status` 字段可通过 API 查询
- 可通过挂起时间计算暂停计费时长
- 备份、数据库等资源在挂起期间仍占用存储，是否计费由业务系统决定

---

## 六、API 访问限制

### 6.1 客户端 API 中间件

客户端 API 的所有服务器路由都经过 [AuthenticateServerAccess](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/app/Http/Middleware/Api/Client/Server/AuthenticateServerAccess.php) 中间件。

中间件调用 `$server->validateCurrentState()` 进行状态校验。

### 6.2 允许访问的接口（白名单）

挂起状态下，以下客户端 API 接口**可以访问**：

| 路由名称 | 端点 | 说明 |
|---------|------|------|
| `api:client:server.view` | `GET /api/client/servers/{server}` | 查看服务器基本信息 |
| `api:client:server.resources` | `GET /api/client/servers/{server}/resources` | 查看服务器资源使用情况 |

中间件逻辑（[AuthenticateServerAccess.php 第 49-62 行](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/app/Http/Middleware/Api/Client/Server/AuthenticateServerAccess.php#L49-L62)）：

```php
try {
    $server->validateCurrentState();
} catch (ServerStateConflictException $exception) {
    // 仍然允许用户查看服务器基本信息（安装中或转移中时）
    if (!$request->routeIs('api:client:server.view')) {
        // 挂起或节点维护时，还允许访问资源使用接口
        if (($server->isSuspended() || $server->node->isUnderMaintenance()) 
            && !$request->routeIs('api:client:server.resources')) {
            throw $exception;
        }
        // 管理员例外：可以访问 websocket
        if (!$user->root_admin || !$request->routeIs($this->except)) {
            throw $exception;
        }
    }
}
```

### 6.3 禁止访问的接口

除上述白名单外，所有其他客户端 API 接口均被拒绝，返回 `409 Conflict` 错误，错误消息为：

> "This server is currently suspended and the functionality requested is unavailable."

错误定义见 [ServerStateConflictException.php](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/app/Exceptions/Htp/Server/ServerStateConflictException.php)。

被禁止的操作包括但不限于：
- 启动/停止/重启服务器
- 发送控制台命令
- 文件管理（浏览、上传、下载、删除等）
- 数据库管理
- 调度任务管理
- 子用户管理
- 网络分配管理
- 备份管理
- 启动参数修改
- 服务器设置修改

### 6.4 管理员例外

管理员（`root_admin = true`）可以访问 Websocket 接口（`api:client:server.ws`），即使服务器挂起。

见 [AuthenticateServerAccess.php 第 15-17 行](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/app/Http/Middleware/Api/Client/Server/AuthenticateServerAccess.php#L15-L17) 和第 58 行。

---

## 七、客户端界面表现

### 7.1 服务器列表页

在 [ServerRow.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/resources/scripts/components/dashboard/ServerRow.tsx) 中：

- 显示红色 "Suspended" 标签
- 不请求资源使用数据（避免不必要的 HTTP 请求）
- 服务器仍可点击进入详情页

```typescript
// 挂起时不轮询资源使用数据
if (isSuspended) return;
```

### 7.2 服务器详情页

在 [ConflictStateRenderer.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/resources/scripts/components/server/ConflictStateRenderer.tsx) 中：

- 显示全屏遮罩
- 标题："Server Suspended"
- 提示信息："This server is suspended and cannot be accessed."
- 使用 `ServerErrorSvg` 错误图标

用户无法看到控制台、文件管理等任何功能页面。

---

## 八、状态校验的核心方法

### 8.1 validateCurrentState()

[Server::validateCurrentState()](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/app/Models/Server.php#L390-L401) 是服务器状态校验的核心方法，在多处被调用：

- SFTP 认证
- 客户端 API 中间件
- 其他需要确认服务器状态正常的场景

校验的异常状态包括：
1. 服务器已挂起（`isSuspended()`）
2. 节点维护中（`node->isUnderMaintenance()`）
3. 服务器未安装完成（`!isInstalled()`）
4. 正在从备份恢复（`status === STATUS_RESTORING_BACKUP`）
5. 正在转移中（`!is_null($this->transfer)`）

### 8.2 isSuspended() 辅助方法

[Server::isSuspended()](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/app/Models/Server.php#L218-L221)：

```php
public function isSuspended(): bool
{
    return $this->status === self::STATUS_SUSPENDED;
}
```

---

## 九、解除挂起（Unsuspend）恢复路径

### 9.1 解除挂起流程

解除挂起的流程与挂起流程完全相同，只是状态变化方向相反：

```
status = 'suspended' → status = null
```

同样通过 [SuspensionService::toggle()](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/app/Services/Servers/SuspensionService.php#L28-L60) 方法，传入 `ACTION_UNSUSPEND` 参数。

### 9.2 恢复后的行为

解除挂起后：
- ✅ 服务器状态恢复为正常（`status = null`）
- ✅ Wings 同步后，服务器可以被启动
- ✅ 所有 API 接口恢复正常访问
- ✅ SFTP 恢复正常
- ⚠️ 服务器**不会自动启动**，需要用户手动启动（Wings 只恢复可启动状态）
- ✅ 调度任务恢复正常执行
- ✅ 备份功能恢复正常

### 9.3 安装完成时的挂起状态保持

如果服务器在安装期间被挂起，安装完成后会**保持挂起状态**，不会自动解除。

见 [ServerInstallController.php 第 71-74 行](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/app/Http/Controllers/Api/Remote/Servers/ServerInstallController.php#L71-L74)：

```php
// Keep the server suspended if it's already suspended
if ($server->status === Server::STATUS_SUSPENDED) {
    $status = Server::STATUS_SUSPENDED;
}
```

---

## 十、API 返回字段

### 10.1 客户端 API

在 [ServerTransformer.php](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/app/Transformers/Api/Client/ServerTransformer.php) 中，挂起相关字段：

| 字段 | 类型 | 说明 |
|-----|------|------|
| `status` | string\|null | 服务器状态，挂起时为 `'suspended'` |
| `is_suspended` | boolean | **已废弃**，请使用 `status` 字段 |
| `is_installing` | boolean | **已废弃**，请使用 `status` 字段 |
| `is_transferring` | boolean | 是否在转移中 |

### 10.2 资源使用 API

在 [StatsTransformer.php](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/app/Transformers/Api/Client/StatsTransformer.php) 中：

| 字段 | 类型 | 说明 |
|-----|------|------|
| `current_state` | string | 服务器电源状态（running/stopped 等） |
| `is_suspended` | boolean | 是否挂起 |
| `resources` | object | 资源使用数据（内存、CPU、磁盘、网络） |

### 10.3 应用 API（管理员）

在 [Application/ServerTransformer.php](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/app/Transformers/Api/Application/ServerTransformer.php) 中：

| 字段 | 类型 | 说明 |
|-----|------|------|
| `status` | string\|null | 服务器状态 |
| `suspended` | boolean | **已废弃**，请使用 `status` 字段 |

---

## 十一、关键代码文件索引

| 文件 | 作用 |
|-----|------|
| [SuspensionService.php](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/app/Services/Servers/SuspensionService.php) | 挂起/解除挂起核心服务 |
| [Server.php](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/app/Models/Server.php) | 服务器模型，包含状态常量和校验方法 |
| [DaemonServerRepository.php](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/app/Repositories/Wings/DaemonServerRepository.php) | Wings 通信，sync 方法 |
| [ServerConfigurationStructureService.php](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/app/Services/Servers/ServerConfigurationStructureService.php) | 生成发给 Wings 的配置结构 |
| [AuthenticateServerAccess.php](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/app/Http/Middleware/Api/Client/Server/AuthenticateServerAccess.php) | 客户端 API 访问控制中间件 |
| [ServerStateConflictException.php](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/app/Exceptions/Http/Server/ServerStateConflictException.php) | 状态冲突异常 |
| [RunTaskJob.php](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/app/Jobs/Schedule/RunTaskJob.php) | 定时任务执行（挂起时跳过） |
| [SftpAuthenticationController.php](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/app/Http/Controllers/Api/Remote/SftpAuthenticationController.php) | SFTP 认证（挂起时拒绝） |
| [ConflictStateRenderer.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/resources/scripts/components/server/ConflictStateRenderer.tsx) | 前端挂起状态展示组件 |
| [Websocket.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/207-panel/resources/scripts/plugins/Websocket.ts) | Websocket 挂起断开处理 |

---

## 十二、客服问答速查

**Q: 服务器挂起后数据会丢失吗？**
A: 不会。挂起只是停止服务器运行并禁用访问，所有数据（包括文件、数据库、备份）都会完整保留。

**Q: 挂起期间还计费吗？**
A: Panel 本身不处理计费逻辑。挂起状态可作为计费系统的参考，具体是否暂停计费由业务系统决定。

**Q: 挂起后用户能做什么？**
A: 用户只能查看服务器基本信息和资源使用情况，不能进行任何操作（启动、文件管理、备份等均不可用）。

**Q: 解除挂起后服务器会自动启动吗？**
A: 不会。解除挂起只是恢复服务器的可操作状态，用户需要手动启动服务器。

**Q: 挂起期间定时备份还会执行吗？**
A: 不会。挂起期间所有定时任务都会被跳过，不会执行备份、定时命令等操作。

**Q: 挂起的服务器还占不占资源？**
A: 服务器进程会被停止，不占用 CPU 和内存。但磁盘空间（包括服务器文件和备份）仍然被占用。

**Q: SFTP 还能连接吗？**
A: 不能。挂起状态下 SFTP 认证会被拒绝。
