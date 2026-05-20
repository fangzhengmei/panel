# Panel 与 Wings 衔接关系详解

## 概述

Pterodactyl 面板新建服务并在 Wings 上真正跑起来的过程涉及三个核心阶段：**实例编排**、**Wings 守护进程调用**、**部署状态回流**。本文档从代码层面详细解析这三个阶段的衔接关系。

---

## 一、实例编排：ServerCreationService

### 1.1 核心入口

**文件位置**: `app/Services/Servers/ServerCreationService.php:52`

```php
public function handle(array $data, ?DeploymentObject $deployment = null): Server
```

这是创建服务器的核心编排方法，调用链通常为：

```
API 入口 (ServerController::store)
    → ServerCreationService::handle()
        → 数据库事务创建 Server 记录
        → DaemonServerRepository::create() 通知 Wings
```

### 1.2 编排流程详解

| 步骤 | 操作 | 代码位置 | 说明 |
|------|------|----------|------|
| 1 | 自动部署配置 | `ServerCreationService.php:56-60` | 如果传入 `DeploymentObject`，自动查找可用节点和分配端口 |
| 2 | 数据校验与补全 | `ServerCreationService.php:64-78` | 根据 allocation_id 推导 node_id，根据 egg_id 推导 nest_id，验证环境变量 |
| 3 | 数据库事务 | `ServerCreationService.php:86-94` | 在事务中创建服务器、分配端口、存储环境变量 |
| 4 | 调用 Wings | `ServerCreationService.php:96-104` | 通知 Wings 创建服务器实例 |
| 5 | 失败回滚 | `ServerCreationService.php:100-104` | 如果 Wings 调用失败，删除已创建的数据库记录 |

### 1.3 Server 模型初始状态

**文件位置**: `app/Models/Server.php:122-126`

服务器创建时状态被设置为：
```php
public const STATUS_INSTALLING = 'installing';
```

**关键状态枚举**:
```php
STATUS_INSTALLING      = 'installing';       // 安装中
STATUS_INSTALL_FAILED  = 'install_failed';   // 安装失败
STATUS_REINSTALL_FAILED = 'reinstall_failed'; // 重装失败
STATUS_SUSPENDED       = 'suspended';        // 已暂停
STATUS_RESTORING_BACKUP = 'restoring_backup'; // 恢复备份中
```

---

## 二、Wings 守护进程调用

### 2.1 DaemonServerRepository

**文件位置**: `app/Repositories/Wings/DaemonServerRepository.php:42`

```php
public function create(bool $startOnCompletion = true): void
{
    $this->getHttpClient()->post('/api/servers', [
        'json' => [
            'uuid' => $this->server->uuid,
            'start_on_completion' => $startOnCompletion,
        ],
    ]);
}
```

### 2.2 HTTP 客户端配置

**文件位置**: `app/Repositories/Wings/DaemonRepository.php:49`

```php
public function getHttpClient(array $headers = []): Client
{
    return new Client([
        'base_uri' => $this->node->getConnectionAddress(),
        'headers' => [
            'Authorization' => 'Bearer ' . $this->node->getDecryptedKey(),
            'Accept' => 'application/json',
            'Content-Type' => 'application/json',
        ],
    ]);
}
```

### 2.3 节点认证机制

**文件位置**: `app/Models/Node.php:189`

节点通过 `daemon_token_id` + `daemon_token` 进行认证：
```php
public function getDecryptedKey(): string
{
    return (string) Container::getInstance()->make(Encrypter::class)
        ->decrypt($this->daemon_token);
}
```

节点连接地址格式：
```
{scheme}://{fqdn}:{daemonListen}
```

### 2.4 Panel → Wings API 接口列表

| 方法 | 路径 | 用途 |
|------|------|------|
| POST | `/api/servers` | 创建服务器 |
| GET | `/api/servers/{uuid}` | 获取服务器详情 |
| POST | `/api/servers/{uuid}/sync` | 触发服务器同步 |
| POST | `/api/servers/{uuid}/reinstall` | 重装服务器 |
| DELETE | `/api/servers/{uuid}` | 删除服务器 |
| POST | `/api/servers/{uuid}/archive` | 请求归档 |

---

## 三、部署状态回流：Wings → Panel

Wings 通过调用 Panel 的 `/api/remote` 系列接口来回传状态。这些接口受 `DaemonAuthenticate` 中间件保护。

### 3.1 DaemonAuthenticate 中间件

**文件位置**: `app/Http/Middleware/Api/Daemon/DaemonAuthenticate.php:34`

认证流程：
1. 解析 `Authorization: Bearer {token_id}.{token}` 头部
2. 根据 `token_id` 查找对应节点
3. 解密节点存储的 `daemon_token` 并与请求中的 token 比对
4. 认证通过后将 `node` 对象注入请求属性

**路由组配置** (`app/Providers/RouteServiceProvider.php:62`):
```php
Route::middleware('daemon')
    ->prefix('/api/remote')
    ->group(base_path('routes/api-remote.php'));
```

### 3.2 核心回流接口

#### 3.2.1 获取安装脚本

**路径**: `GET /api/remote/servers/{uuid}/install`

**文件位置**: `app/Http/Controllers/Api/Remote/Servers/ServerInstallController.php:31`

Wings 在开始安装前调用此接口获取：
- `container_image`: 安装脚本运行容器
- `entrypoint`: 入口命令
- `script`: 安装脚本内容

#### 3.2.2 上报安装结果

**路径**: `POST /api/remote/servers/{uuid}/install`

**文件位置**: `app/Http/Controllers/Api/Remote/Servers/ServerInstallController.php:53`

这是最关键的状态回流接口。Wings 安装完成后调用此接口：

```php
public function store(InstallationDataRequest $request, string $uuid): JsonResponse
{
    $server = $this->repository->getByUuid($uuid);
    $status = null;

    // 根据安装结果设置状态
    if (!$request->boolean('successful')) {
        $status = $request->boolean('reinstall') 
            ? Server::STATUS_REINSTALL_FAILED 
            : Server::STATUS_INSTALL_FAILED;
    }

    // 如果服务器原本是暂停状态，保持暂停
    if ($server->status === Server::STATUS_SUSPENDED) {
        $status = Server::STATUS_SUSPENDED;
    }

    // 更新状态和安装时间
    $this->repository->update($server->id, [
        'status' => $status, 
        'installed_at' => CarbonImmutable::now()
    ], true, true);

    // 触发安装完成事件（发送邮件等）
    if ($request->boolean('successful')) {
        $this->eventDispatcher->dispatch(new ServerInstalled($server));
    }

    return new JsonResponse([], Response::HTTP_NO_CONTENT);
}
```

**请求参数**:
```json
{
    "successful": true,
    "reinstall": false
}
```

#### 3.2.3 获取服务器配置

**路径**: `GET /api/remote/servers/{uuid}`

**文件位置**: `app/Http/Controllers/Api/Remote/Servers/ServerDetailsController.php:38`

Wings 定期或启动时调用此接口同步服务器配置，返回：
- `settings`: 服务器基础配置（构建参数、环境变量、启动命令等）
- `process_configuration`: Egg 进程配置

#### 3.2.4 重置服务器状态

**路径**: `POST /api/remote/servers/reset`

**文件位置**: `app/Http/Controllers/Api/Remote/Servers/ServerDetailsController.php:89`

Wings 重启时调用，将所有处于 `installing` 或 `restoring_backup` 状态的服务器重置为正常状态（null），避免因 Wings 重启导致状态卡住。

---

## 四、完整时序图

```
┌─────────┐          ┌──────────────┐          ┌──────────────┐          ┌───────┐
│ 用户/API │          │ Panel 服务器 │          │ 数据库        │          │ Wings │
└────┬────┘          └──────┬───────┘          └──────┬───────┘          └───┬───┘
     │                      │                         │                        │
     │ 1. 请求创建服务器    │                         │                        │
     │─────────────────────>│                         │                        │
     │                      │                         │                        │
     │                      │ 2. 开启事务            │                        │
     │                      │────────────────────────>│                        │
     │                      │                         │                        │
     │                      │ 3. 插入 Server 记录    │                        │
     │                      │   (status=installing)  │                        │
     │                      │────────────────────────>│                        │
     │                      │                         │                        │
     │                      │ 4. 提交事务            │                        │
     │                      │<────────────────────────│                        │
     │                      │                         │                        │
     │                      │ 5. POST /api/servers   │                        │
     │                      │   {uuid, start_on_completion} │                  │
     │                      │────────────────────────────────────────────────>│
     │                      │                         │                        │
     │                      │ 6. 返回 204            │                        │
     │                      │<────────────────────────────────────────────────│
     │                      │                         │                        │
     │ 7. 返回 Server 对象  │                         │                        │
     │<─────────────────────│                         │                        │
     │                      │                         │                        │
     │                      │ 8. GET /api/remote/servers/{uuid}/install       │
     │                      │<────────────────────────────────────────────────│
     │                      │                         │                        │
     │                      │ 9. 返回安装脚本信息    │                        │
     │                      │────────────────────────────────────────────────>│
     │                      │                         │                        │
     │                      │                         │                        │ 10. 执行安装
     │                      │                         │                        │
     │                      │ 11. POST /api/remote/servers/{uuid}/install     │
     │                      │    {successful: true}  │                        │
     │                      │<────────────────────────────────────────────────│
     │                      │                         │                        │
     │                      │ 12. 更新 status=null,  │                        │
     │                      │     installed_at=now   │                        │
     │                      │────────────────────────>│                        │
     │                      │                         │                        │
     │                      │ 13. 触发 ServerInstalled 事件                   │
     │                      │                         │                        │
     │                      │ 14. 返回 204            │                        │
     │                      │────────────────────────────────────────────────>│
     │                      │                         │                        │
     │                      │                         │                        │ 15. 启动服务器
```

---

## 五、配置数据结构详解

### 5.1 ServerConfigurationStructureService

**文件位置**: `app/Services/Servers/ServerConfigurationStructureService.php:43`

Wings 拉取的服务器配置结构：

```php
return [
    'uuid' => $server->uuid,
    'meta' => [
        'name' => $server->name,
        'description' => $server->description,
    ],
    'suspended' => $server->isSuspended(),
    'environment' => $this->environment->handle($server),  // 环境变量
    'invocation' => $server->startup,                       // 启动命令
    'skip_egg_scripts' => $server->skip_scripts,
    'build' => [
        'memory_limit' => $server->memory,
        'swap' => $server->swap,
        'io_weight' => $server->io,
        'cpu_limit' => $server->cpu,
        'threads' => $server->threads,
        'disk_space' => $server->disk,
        'oom_disabled' => $server->oom_disabled,
    ],
    'container' => [
        'image' => $server->image,
        'requires_rebuild' => false,
    ],
    'allocations' => [
        'default' => ['ip' => ..., 'port' => ...],
        'mappings' => [...],
    ],
    'mounts' => [...],
    'egg' => [...],
];
```

### 5.2 环境变量生成

**文件位置**: `app/Services/Servers/EnvironmentService.php:33`

环境变量来源包括：
1. Egg 变量（用户配置）
2. Panel 内置变量（STARTUP, P_SERVER_UUID 等）
3. 配置文件定义的变量
4. 动态注入的变量

---

## 六、关键设计要点

### 6.1 先落库后调用

代码采用"先持久化到数据库，再调用 Wings"的模式：

```php
// ServerCreationService.php:85-104
$server = $this->connection->transaction(function () use ($data, $eggVariableData) {
    $server = $this->createModel($data);  // 先写入 DB
    $this->storeAssignedAllocations($server, $data);
    $this->storeEggVariables($server, $eggVariableData);
    return $server;
}, 5);

try {
    $this->daemonServerRepository->setServer($server)->create(...);  // 再调用 Wings
} catch (DaemonConnectionException $exception) {
    $this->serverDeletionService->withForce()->handle($server);  // 失败则回滚 DB
    throw $exception;
}
```

**设计原因**: Wings 需要通过 UUID 从 Panel 拉取服务器详细配置，因此必须先在 Panel 端创建记录。

### 6.2 异步安装

Panel 调用 Wings 创建接口后立即返回，安装过程是异步的：
- Panel 不等待安装完成
- Wings 安装完成后通过回调通知 Panel
- 期间服务器状态保持为 `installing`

### 6.3 幂等性与状态重置

Wings 重启时会调用 `/api/remote/servers/reset` 接口，将卡住的中间状态重置为正常状态，确保系统不会因进程重启而永久处于不一致状态。

### 6.4 安全认证

双向认证机制：
- **Panel → Wings**: 使用节点的 Bearer Token（`daemon_token_id.daemon_token`）
- **Wings → Panel**: 使用相同的 Token 机制，通过 `DaemonAuthenticate` 中间件验证

---

## 七、常见问题排查点

1. **服务器一直显示安装中**:
   - 检查 Wings 是否能正常访问 Panel 的 `/api/remote` 接口
   - 检查 Wings 日志中是否有安装脚本执行错误
   - 确认节点 token 配置正确

2. **Wings 回调返回 403**:
   - 检查 `DaemonAuthenticate` 中间件日志
   - 确认节点的 `daemon_token_id` 和 `daemon_token` 配置正确
   - 确认请求来源 IP 是否正确

3. **创建服务器时 Panel 报错但 Wings 已创建**:
   - 这是回滚机制的边界情况，需要手动在 Wings 上清理残留容器
   - 检查网络连接稳定性

---

## 八、核心文件速查表

| 功能 | 文件路径 |
|------|----------|
| 服务器创建编排 | `app/Services/Servers/ServerCreationService.php` |
| Wings API 调用 | `app/Repositories/Wings/DaemonServerRepository.php` |
| Wings 基类 | `app/Repositories/Wings/DaemonRepository.php` |
| 服务器模型 | `app/Models/Server.php` |
| 节点模型 | `app/Models/Node.php` |
| 安装结果回调 | `app/Http/Controllers/Api/Remote/Servers/ServerInstallController.php` |
| 配置拉取接口 | `app/Http/Controllers/Api/Remote/Servers/ServerDetailsController.php` |
| Wings 认证中间件 | `app/Http/Middleware/Api/Daemon/DaemonAuthenticate.php` |
| 配置结构生成 | `app/Services/Servers/ServerConfigurationStructureService.php` |
| 远程 API 路由 | `routes/api-remote.php` |
| 路由服务提供者 | `app/Providers/RouteServiceProvider.php` |
