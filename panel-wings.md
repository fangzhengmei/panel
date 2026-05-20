# Panel 与 Wings 衔接关系详解（可排障级别）

## 概述

Pterodactyl 面板新建服务并在 Wings 上真正跑起来的过程涉及三个核心阶段：**实例编排**、**Wings 守护进程调用**、**部署状态回流**。本文档从代码层面详细解析这三个阶段的衔接关系，重点澄清 `start_on_completion` 机制、Authorization 真实格式、403 排查要点、以及安装状态与运行状态的读取链路差异。

---

## 一、实例编排：ServerCreationService

### 1.1 核心入口

**文件位置**: `app/Services/Servers/ServerCreationService.php:52`

```php
public function handle(array $data, ?DeploymentObject $deployment = null): Server
```

这是创建服务器的核心编排方法，调用链通常为：

```
API 入口 (ServerController::store) 或 管理后台 (CreateServerController::store)
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
STATUS_INSTALLING       = 'installing';        // 安装中
STATUS_INSTALL_FAILED   = 'install_failed';    // 安装失败
STATUS_REINSTALL_FAILED = 'reinstall_failed';  // 重装失败
STATUS_SUSPENDED        = 'suspended';         // 已暂停
STATUS_RESTORING_BACKUP = 'restoring_backup';  // 恢复备份中
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

### 2.2 `start_on_completion` 参数详解

#### 2.2.1 默认值来源

`start_on_completion` 决定了 Wings 在安装完成后是否自动启动服务器。其默认值在不同入口有不同表现：

| 调用入口 | 文件位置 | 默认值 | 说明 |
|----------|----------|--------|------|
| **Application API** | `StoreServerRequest.php:93` | `false` | 可选参数，不传则不自动启动 |
| **管理后台** | `new.blade.php:49` | `true` | 页面 checkbox 默认勾选，提交时值为 `on`/`1` |
| **服务层兜底** | `ServerCreationService.php:98` | `false` | `Arr::get($data, 'start_on_completion', false) ?? false` |
| **方法签名** | `DaemonServerRepository.php:42` | `true` | 方法参数默认值，但 Panel 始终显式传值 |

> **重要**: 虽然 `DaemonServerRepository::create()` 方法签名的默认值是 `true`，但 Panel 所有调用路径都会显式传递该参数，实际生效的是用户传入或请求默认值。

#### 2.2.2 调用链路传递

```
用户请求
    ↓
StoreServerRequest::validated()  [API] 或 CreateServerController::store()  [后台]
    → 提取 start_on_completion（API 默认 false，后台 checkbox 默认 true）
    ↓
ServerCreationService::handle($data)
    → Arr::get($data, 'start_on_completion', false) ?? false
    ↓
DaemonServerRepository::create($startOnCompletion)
    → POST /api/servers { "start_on_completion": <value> }
```

#### 2.2.3 两条运行分支

**分支 A：start_on_completion = true（自动启动）**

```
Wings 接收创建请求
    ↓
创建容器、拉取镜像
    ↓
调用 Panel /api/remote/servers/{uuid}/install 获取安装脚本
    ↓
执行安装脚本
    ↓
安装成功 → 自动调用 power/start 启动服务器
    ↓
回调 Panel /api/remote/servers/{uuid}/install 上报 successful: true
    ↓
Panel 更新 status=null, installed_at=now
    ↓
服务器已启动并运行
```

**分支 B：start_on_completion = false（不自动启动）**

```
Wings 接收创建请求
    ↓
创建容器、拉取镜像
    ↓
调用 Panel /api/remote/servers/{uuid}/install 获取安装脚本
    ↓
执行安装脚本
    ↓
安装成功 → 不启动服务器（容器处于停止状态）
    ↓
回调 Panel /api/remote/servers/{uuid}/install 上报 successful: true
    ↓
Panel 更新 status=null, installed_at=now
    ↓
服务器已就绪但未运行，需用户手动启动
```

### 2.3 其他场景的 start_on_completion

| 场景 | 文件位置 | 硬编码值 | 说明 |
|------|----------|----------|------|
| **服务器迁移** | `DaemonTransferRepository.php:29` | `false` | 迁移完成后不自动启动，需用户确认 |
| **服务器重装** | `ReinstallServerService.php:30` | N/A | 调用 `reinstall()` 方法，无此参数，Wings 内部决定 |

### 2.4 HTTP 客户端配置与 Authorization 真实格式

#### 2.4.1 Panel → Wings 调用

**文件位置**: `app/Repositories/Wings/DaemonRepository.php:49`

```php
public function getHttpClient(array $headers = []): Client
{
    return new Client([
        'base_uri' => $this->node->getConnectionAddress(),
        'headers' => [
            // ⚠️  真实格式：Bearer + 解密后的 daemon_token（纯 token，无点号分隔）
            'Authorization' => 'Bearer ' . $this->node->getDecryptedKey(),
            'Accept' => 'application/json',
            'Content-Type' => 'application/json',
        ],
    ]);
}
```

**文件位置**: `app/Models/Node.php:189`

```php
public function getDecryptedKey(): string
{
    // 返回解密后的 64 位随机字符串
    return (string) Container::getInstance()->make(Encrypter::class)
        ->decrypt($this->daemon_token);
}
```

**⚠️ 关键纠正：Panel → Wings 的 Authorization 格式**

```
Authorization: Bearer <64位随机字符串>
```

- **不是** `Bearer <token_id>.<token>` 格式
- 仅包含解密后的 `daemon_token` 值（64 位随机字符串）
- 由 `NodeCreationService.php:28` 生成：`Str::random(Node::DAEMON_TOKEN_LENGTH)`，DAEMON_TOKEN_LENGTH = 64

#### 2.4.2 Wings → Panel 回调

**文件位置**: `app/Http/Middleware/Api/Daemon/DaemonAuthenticate.php:40-56`

```php
// Wings 回调的 Authorization 格式必须是：Bearer <token_id>.<decrypted_token>
$parts = explode('.', $bearer);  // 按点号拆分
if (count($parts) !== 2 || empty($parts[0]) || empty($parts[1])) {
    throw new BadRequestHttpException('...');  // 400 Bad Request
}

$node = $this->repository->findFirstWhere([
    'daemon_token_id' => $parts[0],  // 用前半部分查节点
]);

// 比对后半部分的解密 token
if (hash_equals((string) $this->encrypter->decrypt($node->daemon_token), $parts[1])) {
    // 认证通过
}
```

**⚠️ 关键纠正：Wings → Panel 的 Authorization 格式**

```
Authorization: Bearer <16位token_id>.<64位decrypted_token>
```

- 必须包含点号分隔符
- 前半部分：`daemon_token_id`（16 位随机字符串）
- 后半部分：解密后的 `daemon_token`（64 位随机字符串）
- 由 `NodeCreationService.php:28-29` 生成：
  ```php
  $data['daemon_token'] = app(Encrypter::class)->encrypt(Str::random(64));
  $data['daemon_token_id'] = Str::random(16);
  ```

#### 2.4.3 双向认证格式对比

| 方向 | Authorization 格式 | 示例 |
|------|-------------------|------|
| **Panel → Wings** | `Bearer <decrypted_token>` | `Bearer a1b2c3d4...`（64 位） |
| **Wings → Panel** | `Bearer <token_id>.<decrypted_token>` | `Bearer wxyz7890.a1b2c3d4...`（16位 + 点 + 64位） |

> Wings 配置文件中同时存储了 `token_id` 和 `token`，因此回调时使用带点号的格式；而 Panel 调用 Wings 时只需要发送 token 本身。

### 2.5 节点连接地址

**文件位置**: `app/Models/Node.php:134`

```php
public function getConnectionAddress(): string
{
    return sprintf('%s://%s:%s', $this->scheme, $this->fqdn, $this->daemonListen);
}
```

格式：`{scheme}://{fqdn}:{daemonListen}`

### 2.6 Panel → Wings API 接口列表

| 方法 | 路径 | 用途 |
|------|------|------|
| POST | `/api/servers` | 创建服务器 |
| GET | `/api/servers/{uuid}` | 获取服务器详情（含运行状态） |
| POST | `/api/servers/{uuid}/sync` | 触发服务器同步 |
| POST | `/api/servers/{uuid}/reinstall` | 重装服务器 |
| DELETE | `/api/servers/{uuid}` | 删除服务器 |
| POST | `/api/servers/{uuid}/archive` | 请求归档 |
| POST | `/api/servers/{uuid}/power` | 发送电源指令（start/stop/restart/kill） |

---

## 三、部署状态回流：Wings → Panel

Wings 通过调用 Panel 的 `/api/remote` 系列接口来回传状态。这些接口受 `DaemonAuthenticate` 中间件保护。

### 3.1 DaemonAuthenticate 中间件与 403 排障指南

**文件位置**: `app/Http/Middleware/Api/Daemon/DaemonAuthenticate.php:34`

#### 3.1.1 完整校验流程

```php
public function handle(Request $request, \Closure $next): mixed
{
    // 步骤 1：检查路由是否在例外列表
    if (in_array($request->route()->getName(), $this->except)) {
        return $next($request);
    }

    // 步骤 2：检查 Bearer Token 是否存在
    if (is_null($bearer = $request->bearerToken())) {
        throw new HttpException(401, '...');  // ⚠️  返回 401，不是 403
    }

    // 步骤 3：检查格式是否为 xxx.yyy
    $parts = explode('.', $bearer);
    if (count($parts) !== 2 || empty($parts[0]) || empty($parts[1])) {
        throw new BadRequestHttpException('...');  // ⚠️  返回 400 Bad Request
    }

    // 步骤 4：用 token_id 查找节点
    try {
        $node = $this->repository->findFirstWhere([
            'daemon_token_id' => $parts[0],
        ]);

        // 步骤 5：比对解密后的 token
        if (hash_equals((string) $this->encrypter->decrypt($node->daemon_token), $parts[1])) {
            $request->attributes->set('node', $node);
            return $next($request);  // ✅ 认证通过
        }
    } catch (RecordNotFoundException $exception) {
        // ⚠️  静默捕获，不暴露节点不存在
        // 直接进入下面的 403
    }

    // 步骤 6：认证失败，返回 403
    throw new AccessDeniedHttpException('You are not authorized to access this resource.');
}
```

#### 3.1.2 ⚠️ 403 排障要点（重要！）

| 错误码 | 触发场景 | 排查优先级 | 排查要点 |
|--------|----------|------------|----------|
| **401** | Bearer Token 不存在 | 低 | 检查 Authorization 请求头是否存在 |
| **400** | Token 格式错误（没有点号或部分为空） | 高 | 检查 Token 是否为 `xxx.yyy` 格式，点号前后是否都有值 |
| **403** | token_id 找不到对应节点 | **最高** | 1. 检查 `daemon_token_id` 是否正确<br>2. 确认节点是否存在于 Panel 数据库<br>3. 检查节点是否已被删除 |
| **403** | token 比对失败 | 高 | 1. 检查 Wings 配置中的 `token` 是否正确<br>2. 尝试重置节点 token（节点管理页面→重新生成）<br>3. 确认 APP_KEY 未变更（加密密钥改变会导致旧 token 解密失败） |

> **⚠️ 重要澄清：没有 IP 校验！**
>
> `DaemonAuthenticate` 中间件**完全没有**检查请求来源 IP。遇到 403 时请优先排查 Token 格式、token_id 匹配和 token 解密问题，而不是浪费时间查 IP 白名单。

#### 3.1.3 快速排障命令

```bash
# 1. 检查请求头是否正确
curl -v -H "Authorization: Bearer <token_id>.<token>" https://panel.example.com/api/remote/servers

# 2. 常见错误示例
# ❌ 错误：没有点号
Authorization: Bearer abcdef1234567890

# ❌ 错误：点号前为空
Authorization: Bearer .abcdef1234567890...

# ✅ 正确
Authorization: Bearer wxyz7890abcdef12.a1b2c3d4e5f6...
```

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

    // ⚠️  仅更新状态和安装时间，不执行任何启动操作
    $this->repository->update($server->id, [
        'status' => $status, 
        'installed_at' => CarbonImmutable::now()
    ], true, true);

    // 触发安装完成事件（仅发送邮件通知）
    $isInitialInstall = is_null($server->installed_at);
    if ($isInitialInstall && config()->get('pterodactyl.email.send_install_notification', true)) {
        $this->eventDispatcher->dispatch(new ServerInstalled($server));
    } elseif (!$isInitialInstall && config()->get('pterodactyl.email.send_reinstall_notification', true)) {
        $this->eventDispatcher->dispatch(new ServerInstalled($server));
    }

    return new JsonResponse([], Response::HTTP_NO_CONTENT);
}
```

**⚠️ 关键点：安装成功回调只更新状态，不保证已启动**

- 此接口**不会**调用 `DaemonPowerRepository` 发送启动指令
- 服务器是否已启动完全取决于 Wings 侧是否根据 `start_on_completion` 自动启动
- 如果 `start_on_completion = false`，即使回调返回 successful: true，服务器也处于停止状态
- Panel 仅信任 Wings 上报的状态，不主动校验服务器实际运行状态

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

## 四、安装状态 vs 运行状态：读取链路差异

### 4.1 安装状态（Installation Status）

**来源**: Panel 数据库 `servers.status` 字段

**读取方式**: 本地数据库查询，无网络开销

**代码位置**: `app/Models/Server.php:213`

```php
public function isInstalled(): bool
{
    // 仅根据 status 字段判断
    return $this->status !== self::STATUS_INSTALLING 
        && $this->status !== self::STATUS_INSTALL_FAILED;
}
```

**返回值**:
- `false`: 安装中或安装失败
- `true`: 安装完成（无论服务器是否在运行）

**使用场景**:
- 管理后台显示安装进度 (`resources/views/admin/servers/view/index.blade.php:140`)
- API 返回 `is_installing` 字段 (`ServerTransformer.php:80`)
- 权限控制，安装中的服务器禁止某些操作

### 4.2 运行状态（Running Status）

**来源**: Wings 实时接口 `/api/servers/{uuid}`

**读取方式**: 远程 HTTP 调用，有 20 秒缓存

**代码位置**: `app/Http/Controllers/Api/Client/Servers/ResourceUtilizationController.php:30`

```php
public function __invoke(GetServerRequest $request, Server $server): array
{
    $key = "resources:$server->uuid";
    // 缓存 20 秒
    $stats = $this->cache->remember($key, Carbon::now()->addSeconds(20), function () use ($server) {
        // 调用 Wings 获取实时状态
        return $this->repository->setServer($server)->getDetails();
    });

    return $this->fractal->item($stats)
        ->transformWith($this->getTransformer(StatsTransformer::class))
        ->toArray();
}
```

**代码位置**: `app/Transformers/Api/Client/StatsTransformer.php:18`

```php
public function transform(array $data): array
{
    return [
        // Wings 返回的实时运行状态：running / stopped / starting / stopping
        'current_state' => Arr::get($data, 'state', 'stopped'),
        'is_suspended' => Arr::get($data, 'is_suspended', false),
        'resources' => [
            'memory_bytes' => Arr::get($data, 'utilization.memory_bytes', 0),
            'cpu_absolute' => Arr::get($data, 'utilization.cpu_absolute', 0),
            // ...
        ],
    ];
}
```

### 4.3 两者差异对比

| 维度 | 安装状态（is_installed） | 运行状态（current_state） |
|------|-------------------------|---------------------------|
| **数据来源** | Panel 数据库 | Wings 远程 API |
| **读取方式** | 本地 SQL 查询 | HTTP 请求 + 20 秒缓存 |
| **响应速度** | 毫秒级 | 百毫秒级（依赖网络） |
| **可能值** | `true` / `false` | `running` / `stopped` / `starting` / `stopping` |
| **更新时机** | Wings 回调 `/api/remote/servers/{uuid}/install` 时更新 | 每次调用 `/api/client/servers/{uuid}/resources` 时更新 |
| **代表含义** | 安装流程是否完成 | 服务器进程是否在运行 |

### 4.4 状态组合场景

| 安装状态 | 运行状态 | 含义 |
|---------|----------|------|
| `false`（installing） | N/A | 正在安装中，无法查询运行状态 |
| `true` | `running` | ✅ 安装完成且正在运行 |
| `true` | `stopped` | ⏸️ 安装完成但已停止（可能 start_on_completion=false，或用户手动停止） |
| `true` | `starting` | 🔄 安装完成，正在启动中 |
| `false`（install_failed） | N/A | ❌ 安装失败 |

> **排障提示**: 如果看到"安装完成但服务器未启动"，不要怀疑回调逻辑有问题——这是正常现象。检查 `start_on_completion` 参数是否为 `true`，或手动调用电源接口启动。

---

## 五、完整时序图

### 5.1 自动启动分支（start_on_completion = true）

```
┌─────────┐          ┌──────────────┐          ┌──────────────┐          ┌───────┐
│ 用户/API │          │ Panel 服务器 │          │ 数据库        │          │ Wings │
└────┬────┘          └──────┬───────┘          └──────┬───────┘          └───┬───┘
     │                      │                         │                        │
     │ 1. 创建服务器         │                         │                        │
     │    start_on_completion=true │                  │                        │
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
     │                      │   Authorization: Bearer <token> │                │
     │                      │   {uuid, start_on_completion: true} │            │
     │                      │────────────────────────────────────────────────>│
     │                      │                         │                        │
     │                      │ 6. 返回 204            │                        │
     │                      │<────────────────────────────────────────────────│
     │                      │                         │                        │
     │ 7. 返回 Server 对象  │                         │                        │
     │<─────────────────────│                         │                        │
     │                      │                         │                        │
     │                      │ 8. GET /api/remote/servers/{uuid}/install       │
     │                      │   Authorization: Bearer <id>.<token> │           │
     │                      │<────────────────────────────────────────────────│
     │                      │                         │                        │
     │                      │ 9. 返回安装脚本信息    │                        │
     │                      │────────────────────────────────────────────────>│
     │                      │                         │                        │
     │                      │                         │                        │ 10. 执行安装
     │                      │                         │                        │
     │                      │                         │                        │ 11. 安装成功
     │                      │                         │                        │ 12. 自动启动服务器
     │                      │                         │                        │
     │                      │ 13. POST /api/remote/servers/{uuid}/install     │
     │                      │    {successful: true}  │                        │
     │                      │<────────────────────────────────────────────────│
     │                      │                         │                        │
     │                      │ 14. 更新 status=null,  │                        │
     │                      │     installed_at=now   │                        │
     │                      │────────────────────────>│                        │
     │                      │                         │                        │
     │                      │ 15. 触发 ServerInstalled 事件（发邮件）         │
     │                      │                         │                        │
     │                      │ 16. 返回 204            │                        │
     │                      │────────────────────────────────────────────────>│
     │                      │                         │                        │
     │                      │                         │                        │ ✅ 服务器已启动运行
```

### 5.2 不自动启动分支（start_on_completion = false）

```
┌─────────┐          ┌──────────────┐          ┌──────────────┐          ┌───────┐
│ 用户/API │          │ Panel 服务器 │          │ 数据库        │          │ Wings │
└────┬────┘          └──────┬───────┘          └──────┬───────┘          └───┬───┘
     │                      │                         │                        │
     │ 1. 创建服务器         │                         │                        │
     │    start_on_completion=false │                 │                        │
     │─────────────────────>│                         │                        │
     │                      │                         │                        │
     │  ... 步骤 2-10 同上 ...                      │                        │
     │                      │                         │                        │
     │                      │                         │                        │ 11. 安装成功
     │                      │                         │                        │ 12. 不启动服务器（停止状态）
     │                      │                         │                        │
     │                      │ 13. POST /api/remote/servers/{uuid}/install     │
     │                      │    {successful: true}  │                        │
     │                      │<────────────────────────────────────────────────│
     │                      │                         │                        │
     │                      │ 14. 更新 status=null,  │                        │
     │                      │     installed_at=now   │                        │
     │                      │────────────────────────>│                        │
     │                      │                         │                        │
     │                      │ 15. 触发 ServerInstalled 事件（发邮件）         │
     │                      │                         │                        │
     │                      │ 16. 返回 204            │                        │
     │                      │────────────────────────────────────────────────>│
     │                      │                         │                        │
     │                      │                         │                        │ ⏸️ 服务器就绪但未运行
     │                      │                         │                        │    需用户手动启动
```

---

## 六、配置数据结构详解

### 6.1 ServerConfigurationStructureService

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

### 6.2 环境变量生成

**文件位置**: `app/Services/Servers/EnvironmentService.php:33`

环境变量来源包括：
1. Egg 变量（用户配置）
2. Panel 内置变量（STARTUP, P_SERVER_UUID 等）
3. 配置文件定义的变量
4. 动态注入的变量

---

## 七、关键设计要点

### 7.1 先落库后调用

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

### 7.2 异步安装 + 启动决策在 Wings 侧

Panel 调用 Wings 创建接口后立即返回，安装和启动逻辑完全在 Wings 侧：
- Panel 不等待安装完成
- Wings 安装完成后通过回调通知 Panel
- **是否自动启动由 Wings 根据 `start_on_completion` 参数决定**
- Panel 的安装回调仅更新状态，不控制服务器启停

### 7.3 安装成功 ≠ 服务器已启动

这是最容易产生误解的设计：
- `POST /api/remote/servers/{uuid}/install` 回调中的 `successful: true` 仅表示**安装过程成功完成**
- 服务器是否实际运行取决于 `start_on_completion` 参数
- API 调用者需要区分"安装完成"和"服务器正在运行"两个不同状态

### 7.4 幂等性与状态重置

Wings 重启时会调用 `/api/remote/servers/reset` 接口，将卡住的中间状态重置为正常状态，确保系统不会因进程重启而永久处于不一致状态。

### 7.5 安全认证

双向认证机制：
- **Panel → Wings**: `Bearer <decrypted_token>`（仅 64 位 token）
- **Wings → Panel**: `Bearer <token_id>.<decrypted_token>`（16 位 ID + 点 + 64 位 token）
- Token 存储时加密，使用时解密
- **无 IP 校验**，完全基于 Token 认证

---

## 八、排障指南

### 8.1 服务器一直显示安装中

**排查步骤**:
1. 检查 Wings 是否能正常访问 Panel 的 `/api/remote` 接口
2. 检查 Wings 日志中是否有安装脚本执行错误
3. 确认节点 token 配置正确
4. 查看 Wings 是否成功回调了 `/api/remote/servers/{uuid}/install`
5. 检查 Panel 日志中是否有 403/400 错误

### 8.2 安装成功但服务器未启动

**排查步骤**:
1. 检查创建服务器时 `start_on_completion` 参数是否为 `true`
2. 通过 Application API 创建时默认是 `false`，需要显式指定
3. 检查 Wings 日志中是否有启动失败的错误
4. 可通过电源管理接口手动启动服务器

### 8.3 Wings 回调返回 403

**⚠️ 优先排查（按顺序）**:
1. ✅ **检查 Token 格式**: 是否为 `xxx.yyy` 格式？点号前后是否都有值？→ 400
2. ✅ **检查 token_id**: 前半部分是否与 Panel 节点的 `daemon_token_id` 一致？
3. ✅ **检查 token**: 后半部分是否与节点解密后的 `daemon_token` 一致？
4. ✅ **重置 Token**: 节点管理页面重新生成 token，更新 Wings 配置
5. ✅ **检查 APP_KEY**: 是否有变更？（APP_KEY 变更会导致旧 token 解密失败）
6. ❌ **无需检查 IP**: 代码中没有 IP 校验逻辑

### 8.4 创建服务器时 Panel 报错但 Wings 已创建

- 这是回滚机制的边界情况，需要手动在 Wings 上清理残留容器
- 检查网络连接稳定性

### 8.5 安装状态显示正常但无法连接服务器

**排查步骤**:
1. 调用 `/api/client/servers/{uuid}/resources` 接口查看 `current_state`
2. 如果是 `stopped`，手动调用电源接口启动
3. 如果是 `running`，检查端口映射和防火墙配置
4. 检查 Wings 日志中是否有容器网络错误

---

## 九、核心文件速查表

| 功能 | 文件路径 |
|------|----------|
| 服务器创建编排 | `app/Services/Servers/ServerCreationService.php` |
| Wings API 调用 | `app/Repositories/Wings/DaemonServerRepository.php` |
| Wings 基类（含 Authorization 配置） | `app/Repositories/Wings/DaemonRepository.php` |
| 电源操作 | `app/Repositories/Wings/DaemonPowerRepository.php` |
| 服务器模型（含 is_installed 判断） | `app/Models/Server.php` |
| 节点模型（含 token 解密） | `app/Models/Node.php` |
| 节点创建（token 生成） | `app/Services/Nodes/NodeCreationService.php` |
| 安装结果回调 | `app/Http/Controllers/Api/Remote/Servers/ServerInstallController.php` |
| 配置拉取接口 | `app/Http/Controllers/Api/Remote/Servers/ServerDetailsController.php` |
| Wings 认证中间件（403 排查） | `app/Http/Middleware/Api/Daemon/DaemonAuthenticate.php` |
| 运行状态查询接口 | `app/Http/Controllers/Api/Client/Servers/ResourceUtilizationController.php` |
| 运行状态转换器 | `app/Transformers/Api/Client/StatsTransformer.php` |
| 配置结构生成 | `app/Services/Servers/ServerConfigurationStructureService.php` |
| API 创建请求 | `app/Http/Requests/Api/Application/Servers/StoreServerRequest.php` |
| 管理后台创建控制器 | `app/Http/Controllers/Admin/Servers/CreateServerController.php` |
| 远程 API 路由 | `routes/api-remote.php` |
| 路由服务提供者 | `app/Providers/RouteServiceProvider.php` |
