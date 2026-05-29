# Pterodactyl Panel 控制台 WebSocket 连接与鉴权流程

## 1. 整体架构概览

Pterodactyl Panel 的控制台 WebSocket 连接采用 **Panel 作为认证代理，Wings 作为实际 WebSocket 服务端** 的架构模式：

```
┌─────────────┐   1. 获取临时Token    ┌─────────────┐
│   Browser   │ ─────────────────────> │    Panel    │
│ (Frontend)  │ <───────────────────── │   (Backend) │
└─────────────┘   2. 返回JWT+WS地址    └─────────────┘
       │
       │ 3. WebSocket连接 + JWT鉴权
       ▼
┌─────────────┐
│    Wings    │
│  (Daemon)   │
└─────────────┘
```

## 2. 临时 Token 获取流程

### 2.1 前端 API 调用

**文件**: `resources/scripts/api/server/getWebsocketToken.ts:8-18`

前端通过 HTTP GET 请求获取 WebSocket 连接凭证：

```typescript
http.get(`/api/client/servers/${server}/websocket`)
    .then(({ data }) => resolve({
        token: data.data.token,    // JWT 令牌
        socket: data.data.socket,  // WebSocket 连接地址
    }))
```

**API 路由**: `routes/api-client.php:66-68`
```php
Route::middleware([ResourceLimit::Websocket->middleware()])
    ->get('/websocket', Client\Servers\WebsocketController::class)
    ->name('api:client:server.ws');
```

### 2.2 后端鉴权流程

**文件**: `app/Http/Controllers/Api/Client/Servers/WebsocketController.php:33-72`

#### 权限检查
```php
// 检查用户是否有 websocket.connect 权限
if ($user->cannot(Permission::ACTION_WEBSOCKET_CONNECT, $server)) {
    throw new HttpForbiddenException('...');
}

// 获取用户在该服务器上的所有权限
$permissions = $this->permissionsService->handle($server, $user);
```

**权限获取服务**: `app/Services/Servers/GetUserPermissionsService.php:15-33`
- 管理员/所有者: 拥有 `*` 全部权限 + 额外 admin 权限
- 子用户: 从 `subusers` 表获取分配的权限
- 管理员额外权限: `admin.websocket.errors`, `admin.websocket.install`, `admin.websocket.transfer`

### 2.3 JWT 令牌生成

**文件**: `app/Services/Nodes/NodeJWTService.php:63-102`

JWT 使用 Node 的密钥进行 HMAC-SHA256 签名：

```php
$token = $this->jwtService
    ->setExpiresAt(CarbonImmutable::now()->addMinutes(10))  // 10分钟有效期
    ->setUser($request->user())
    ->setClaims([
        'server_uuid' => $server->uuid,      // 服务器标识
        'permissions' => $permissions,       // 用户权限列表
    ])
    ->handle($node, $user->id . $server->uuid);
```

#### JWT 标准声明 (Claims)
| 声明 | 说明 |
|------|------|
| `iss` | 签发者，Panel 的 app.url |
| `aud` | 受众，Node 的连接地址 |
| `jti` | 唯一标识，`md5(user_id + server_uuid)` |
| `iat` | 签发时间 |
| `nbf` | 最早可用时间 (签发时间 -5分钟) |
| `exp` | 过期时间 (签发时间 +10分钟) |
| `sub` | 主题，可选 |

#### JWT 自定义声明
| 声明 | 说明 |
|------|------|
| `server_uuid` | 目标服务器 UUID |
| `permissions` | 用户权限数组 |
| `user_uuid` | 用户 UUID |
| `user_id` | 用户 ID (已弃用，向后兼容) |
| `unique_id` | 随机字符串，防重放 |

### 2.4 返回响应

```php
return new JsonResponse([
    'data' => [
        'token' => $token->toString(),  // JWT 字符串
        'socket' => $socket . sprintf('/api/servers/%s/ws', $server->uuid),
    ],
]);
```

WebSocket 地址转换规则:
- `https://` → `wss://`
- `http://` → `ws://`

## 3. Wings WebSocket 接入流程

### 3.1 连接管理

**文件**: `resources/scripts/plugins/Websocket.ts:4-95`

WebSocket 类基于 `sockette` 库实现，支持自动重连：

```typescript
class Websocket extends EventEmitter {
    private socket: Sockette | null = null;
    private url: string | null = null;
    private token = '';  // 认证令牌
    
    connect(url: string): this {
        this.socket = new Sockette(url, {
            timeout: 1000,
            maxAttempts: 20,  // 最多重连20次
            onopen: () => {
                this.emit('SOCKET_OPEN');
                this.authenticate();  // 连接建立后立即认证
            },
            // ... 其他事件处理
        });
        return this;
    }
}
```

### 3.2 鉴权握手

连接建立后，前端立即发送 `auth` 事件进行鉴权：

```typescript
authenticate() {
    if (this.url && this.token) {
        this.send('auth', this.token);  // 发送 JWT 给 Wings
    }
}
```

**消息格式**:
```json
{
    "event": "auth",
    "args": ["<JWT_TOKEN>"]
}
```

### 3.3 Token 刷新机制

**文件**: `resources/scripts/components/server/WebsocketHandler.tsx:50-52`

Token 有效期为 10 分钟，Wings 会提前发送过期通知：

```typescript
// 收到 token 即将过期事件（提前3分钟）
socket.on('token expiring', () => updateToken(uuid, socket));

// 收到 token 已过期事件
socket.on('token expired', () => updateToken(uuid, socket));
```

**Token 更新逻辑**: `WebsocketHandler.tsx:20-30`
```typescript
const updateToken = (uuid: string, socket: Websocket) => {
    getWebsocketToken(uuid)
        .then((data) => socket.setToken(data.token, true))  // isUpdate=true 会重新认证
        .catch(console.error);
};
```

### 3.4 认证状态事件

| 事件 | 触发时机 |
|------|----------|
| `auth success` | JWT 验证通过 |
| `token expiring` | Token 剩余 < 3分钟 |
| `token expired` | Token 已过期 |
| `jwt error` | JWT 验证