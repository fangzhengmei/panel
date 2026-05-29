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
    throw new HttpForbiddenException('You do not have permission to connect to this server\'s websocket.');
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
            onmessage: (e) => {
                const { event, args } = JSON.parse(e.data);
                args ? this.emit(event, ...args) : this.emit(event);
            },
            // ... 其他事件处理
        });
        return this;
    }
}
```

### 3.2 鉴权握手

连接建立后，前端立即发送 `auth` 事件进行鉴权：

**文件**: `Websocket.ts:72-76`
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
    if (updatingToken) return;
    updatingToken = true;
    getWebsocketToken(uuid)
        .then((data) => socket.setToken(data.token, true))  // isUpdate=true 会重新认证
        .catch((error) => console.error(error))
        .then(() => { updatingToken = false; });
};
```

### 3.4 认证状态事件

| 事件 | 触发时机 | 参数 |
|------|----------|------|
| `auth success` | JWT 验证通过 | 无 |
| `token expiring` | Token 剩余 < 3分钟 | 无 |
| `token expired` | Token 已过期 | 无 |
| `jwt error` | JWT 验证失败 | error message |
| `daemon error` | 守护进程错误 | error message |

### 3.5 内部连接状态事件

**文件**: `Websocket.ts:33-55`

| 事件 | 触发时机 |
|------|----------|
| `SOCKET_OPEN` | WebSocket 连接建立 |
| `SOCKET_CLOSE` | WebSocket 连接关闭 |
| `SOCKET_ERROR` | WebSocket 连接错误 |
| `SOCKET_RECONNECT` | WebSocket 重连中 |
| `SOCKET_CONNECT_ERROR` | 重连次数耗尽 (20次) |

### 3.6 特殊关闭码

**文件**: `Websocket.ts:40-50`

- `4400`: 预留错误码
- `4409`: 服务器已暂停，停止重连

## 4. WebSocket 事件协议全集

### 4.1 统一消息格式

**文件**: `Websocket.ts:92-94`

所有消息统一使用 `{event, args}` 格式：

```typescript
send(event: string, payload?: string | string[]) {
    this.socket?.json({ 
        event, 
        args: Array.isArray(payload) ? payload : [payload] 
    });
}
```

### 4.2 客户端请求事件 (Browser -> Wings)

**文件**: `resources/scripts/components/server/events.ts:16-20`

| 事件常量 | 事件名 | 参数 | 用途 | 代码位置 |
|---------|--------|------|------|----------|
| `SocketRequest.SEND_LOGS` | `send logs` | 无 | 请求历史日志 | `Console.tsx:189` |
| `SocketRequest.SEND_STATS` | `send stats` | 无 | 请求统计数据 | `ServerDetailsBlock.tsx:70` |
| `SocketRequest.SET_STATE` | `set state` | `start`\|`stop`\|`restart`\|`kill` | 电源操作 | `PowerButtons.tsx:29` |
| - | `send command` | command string | 发送控制台命令 | `Console.tsx:121` |
| - | `auth` | JWT token | 鉴权握手 | `Websocket.ts:74` |

### 4.3 服务端推送事件 (Wings -> Browser)

**文件**: `resources/scripts/components/server/events.ts:1-14`

| 事件常量 | 事件名 | 参数 | 用途 | 代码位置 |
|---------|--------|------|------|----------|
| `SocketEvent.CONSOLE_OUTPUT` | `console output` | log line | 控制台输出日志 | `Console.tsx:172` |
| `SocketEvent.INSTALL_OUTPUT` | `install output` | log line | 安装过程输出 | `Console.tsx:173` |
| `SocketEvent.INSTALL_STARTED` | `install started` | 无 | 安装开始 | `InstallListener.tsx:26` |
| `SocketEvent.INSTALL_COMPLETED` | `install completed` | 无 | 安装完成 | `InstallListener.tsx:20` |
| `SocketEvent.STATUS` | `status` | `offline`\|`starting`\|`stopping`\|`running` | 服务器状态变更 | `WebsocketHandler.tsx:44` |
| `SocketEvent.STATS` | `stats` | JSON string | 资源统计数据 | `StatGraphs.tsx:52` |
| `SocketEvent.TRANSFER_LOGS` | `transfer logs` | log line | 迁移日志 | `Console.tsx:174` |
| `SocketEvent.TRANSFER_STATUS` | `transfer status` | `pending`\|`processing`\|`failed`\|`completed` | 迁移状态 | `TransferListener.tsx:11` |
| `SocketEvent.DAEMON_MESSAGE` | `daemon message` | message | 守护进程消息 | `Console.tsx:176` |
| `SocketEvent.DAEMON_ERROR` | `daemon error` | error message | 守护进程错误 | `Console.tsx:177`, `WebsocketHandler.tsx:46` |
| `SocketEvent.BACKUP_COMPLETED` | `backup completed` | 无 | 备份完成 | - |
| `SocketEvent.BACKUP_RESTORE_COMPLETED` | `backup restore completed` | 无 | 备份恢复完成 | `InstallListener.tsx:12` |

### 4.4 STATS 事件参数结构

**文件**: `StatGraphs.tsx:52-66`

`stats` 事件的参数是 JSON 字符串，解析后结构：
```typescript
{
    cpu_absolute: number;           // CPU 使用率（绝对值）
    memory_bytes: number;           // 内存使用（字节）
    network: {
        tx_bytes: number;           // 网络发送总量（字节）
        rx_bytes: number;           // 网络接收总量（字节）
    }
}
```

## 5. JWT 验证后的共用连接机制

### 5.1 核心设计思想

JWT 验证通过后，**所有操作复用同一个 WebSocket 连接**，通过不同的 `event` 字段区分消息类型。这种设计的优势：
- 减少连接开销，避免为不同功能建立多个连接
- 统一鉴权上下文，权限信息在 JWT 中一次性验证
- 简化前端状态管理，单一连接实例全局共享

### 5.2 连接生命周期与事件订阅时序

```
Browser                          Wings
   |                               |
   | 1. WebSocket 连接 (已完成JWT验证)
   |<------------------------------>|
   |    auth success               |
   |                               |
   | 2. 日志订阅请求               |
   | {"event":"send logs"}        |
   |------------------------------>|
   |                               |
   | 3. 历史日志推送 (批量)        |
   | {"event":"console output", "args":["[2024-01-01] Server started"]} |
   |<------------------------------|
   | {"event":"console output", "args":["[2024-01-01] Loading plugins..."]} |
   |<------------------------------|
   | ... (多条日志)                |
   |                               |
   | 4. 统计数据请求               |
   | {"event":"send stats"}       |
   |------------------------------>|
   |                               |
   | 5. 统计数据推送               |
   | {"event":"stats", "args":["{\"cpu_absolute\":15.5,\"memory_bytes\":104857600}"]} |
   |<------------------------------|
   |                               |
   | 6. 实时日志推送 (持续)        |
   | {"event":"console output", "args":["Player joined the game"]} |
   |<------------------------------|
   |                               |
   | 7. 用户发送控制台命令         |
   | {"event":"send command", "args":["say Hello World"]} |
   |------------------------------>|
   |                               |
   | 8. 命令执行结果输出           |
   | {"event":"console output", "args":["[Server] Hello World"]} |
   |<------------------------------|
   |                               |
   | 9. 用户点击电源按钮           |
   | {"event":"set state", "args":["restart"]} |
   |------------------------------>|
   |                               |
   | 10. 状态变更推送              |
   | {"event":"status", "args":["stopping"]} |
   |<------------------------------|
   | {"event":"console output", "args":["Stopping server..."]} |
   |<------------------------------|
   | {"event":"status", "args":["starting"]} |
   |<------------------------------|
   | {"event":"console output", "args":["Starting server..."]} |
   |<------------------------------|
   | {"event":"status", "args":["running"]} |
   |<------------------------------|
   |                               |
   | 11. 持续统计数据推送 (定期)   |
   | {"event":"stats", "args":["{...}"]} |
   |<------------------------------|
   | (每几秒一次，持续推送)        |
```

### 5.3 日志订阅机制详解

**文件**: `Console.tsx:169-199`

#### 订阅流程
```typescript
useEffect(() => {
    const listeners: Record<string, (s: string) => void> = {
        [SocketEvent.STATUS]: handlePowerChangeEvent,
        [SocketEvent.CONSOLE_OUTPUT]: handleConsoleOutput,
        [SocketEvent.INSTALL_OUTPUT]: handleConsoleOutput,
        [SocketEvent.TRANSFER_LOGS]: handleConsoleOutput,
        [SocketEvent.DAEMON_MESSAGE]: (line) => handleConsoleOutput(line, true),
        [SocketEvent.DAEMON_ERROR]: handleDaemonErrorOutput,
    };

    if (connected && instance) {
        if (!isTransferring) {
            terminal.clear();
        }

        // 注册所有事件监听器
        Object.keys(listeners).forEach((key: string) => {
            instance.addListener(key, listeners[key]);
        });
        
        // 请求历史日志
        instance.send(SocketRequest.SEND_LOGS);
    }

    // 清理函数：移除所有监听器
    return () => {
        if (instance) {
            Object.keys(listeners).forEach((key: string) => {
                instance.removeListener(key, listeners[key]);
            });
        }
    };
}, [connected, instance]);
```

**关键点**:
1. `send logs` 请求触发 Wings 推送历史日志缓冲区
2. 多条 `console output` 事件批量推送历史日志
3. 后续实时日志持续通过 `console output` 事件推送
4. 组件卸载时自动移除监听器，避免内存泄漏

### 5.4 控制台命令发送机制

**文件**: `Console.tsx:117-123`

```typescript
if (e.key === 'Enter' && command.length > 0) {
    // 保存命令历史
    setHistory((prevHistory) => [command, ...prevHistory!].slice(0, 32));
    setHistoryIndex(-1);

    // 通过同一连接发送命令
    instance && instance.send('send command', command);
    e.currentTarget.value = '';
}
```

**权限检查**: `Console.tsx:66`
```typescript
const [canSendCommands] = usePermissions(['control.console']);
// 若无权限则不渲染命令输入框
```

### 5.5 Power Action 机制

**文件**: `PowerButtons.tsx:17-31`

```typescript
const onButtonClick = (
    action: PowerAction | 'kill-confirmed',
    e: React.MouseEvent<HTMLButtonElement, MouseEvent>
): void => {
    e.preventDefault();
    if (action === 'kill') {
        return setOpen(true);  // 强制停止需要确认
    }

    if (instance) {
        setOpen(false);
        // 通过同一连接发送电源操作
        instance.send(
            'set state', 
            action === 'kill-confirmed' ? 'kill' : action
        );
    }
};
```

**权限检查**: 通过 `<Can>` 组件控制按钮可见性
- `control.start` - 启动权限
- `control.stop` - 停止权限  
- `control.restart` - 重启权限

### 5.6 状态推送机制

**文件**: `WebsocketHandler.tsx:44`

```typescript
socket.on('status', (status) => setServerStatus(status));
```

状态值: `offline` | `starting` | `stopping` | `running` | `null`

**状态更新全局共享**: 通过 `ServerContext` store 管理，所有组件可订阅
```typescript
const status = ServerContext.useStoreState((state) => state.status.value);
```

### 5.7 统计数据推送机制

**文件**: `StatGraphs.tsx:52-66`

```typescript
useWebsocketEvent(SocketEvent.STATS, (data: string) => {
    let values: any = {};
    try {
        values = JSON.parse(data);
    } catch (e) {
        return;
    }
    // 更新 CPU、内存、网络图表数据
    cpu.push(values.cpu_absolute);
    memory.push(Math.floor(values.memory_bytes / 1024 / 1024));
    network.push([
        previous.current.tx < 0 ? 0 : Math.max(0, values.network.tx_bytes - previous.current.tx),
        previous.current.rx < 0 ? 0 : Math.max(0, values.network.rx_bytes - previous.current.rx),
    ]);
    previous.current = { tx: values.network.tx_bytes, rx: values.network.rx_bytes };
});
```

**主动请求**: `ServerDetailsBlock.tsx:70`
```typescript
useEffect(() => {
    if (connected && instance) {
        instance.send(SocketRequest.SEND_STATS);
    }
}, [instance, connected]);
```

### 5.8 多组件事件订阅机制

**文件**: `resources/scripts/plugins/useWebsocketEvent.ts`

多个组件可通过 `useWebsocketEvent` Hook 独立订阅同一连接的事件：

```typescript
// Console 组件订阅日志和状态事件
// StatGraphs 组件订阅 stats 事件
// InstallListener 组件订阅 install 相关事件
// TransferListener 组件订阅 transfer 相关事件
// Feature 组件订阅 console output 进行特殊匹配
```

**事件分发机制**: `Websocket.ts:25-31`
```typescript
onmessage: (e) => {
    try {
        const { event, args } = JSON.parse(e.data);
        // 通过 EventEmitter 广播给所有订阅者
        args ? this.emit(event, ...args) : this.emit(event);
    } catch (ex) {
        console.warn('Failed to parse incoming websocket message.', ex);
    }
}
```

## 6. 完整端到端时序图

```
Browser                          Panel                          Wings
   |                               |                               |
   | 1. GET /api/client/servers/{uuid}/websocket                  |
   |------------------------------>|                               |
   |                               | 2. 权限检查                  |
   |                               |    - websocket.connect       |
   |                               |    - 获取用户权限            |
   |                               |                               |
   |                               | 3. 生成 JWT                  |
   |                               |    - exp: +10min             |
   |                               |    - 包含 server_uuid, permissions |
   |                               |    - Node 密钥签名           |
   |<------------------------------|                               |
   |    {token, socket: wss://node/api/servers/{uuid}/ws}         |
   |                                                               |
   | 4. WebSocket 连接                                             |
   |-------------------------------------------------------------->|
   |                                                               |
   | 5. 发送 auth 事件                                             |
   | {"event":"auth","args":["<JWT>"]}                            |
   |-------------------------------------------------------------->|
   |                                                               |
   | 6. Wings 验证 JWT                                             |
   |    - 签名验证 (Node 密钥)                                     |
   |    - 过期检查 (exp, nbf)                                     |
   |    - 权限验证 (permissions)                                  |
   |<--------------------------------------------------------------|
   |    {"event":"auth success"}                                   |
   |                                                               |
   | 7. 请求历史日志                                               |
   | {"event":"send logs"}                                        |
   |-------------------------------------------------------------->|
   |<--------------------------------------------------------------|
   |    {"event":"console output","args":["..."]}                  |
   |<--------------------------------------------------------------|
   |    {"event":"console output","args":["..."]}                  |
   |    ... (批量历史日志)                                         |
   |                                                               |
   | 8. 请求统计数据                                               |
   | {"event":"send stats"}                                       |
   |-------------------------------------------------------------->|
   |<--------------------------------------------------------------|
   |    {"event":"stats","args":["{\"cpu_absolute\":...}"]}        |
   |                                                               |
   | 9. 实时日志持续推送 (自动)                                    |
   |<--------------------------------------------------------------|
   |    {"event":"console output","args":["..."]}                  |
   |<--------------------------------------------------------------|
   |    {"event":"console output","args":["..."]}                  |
   |                                                               |
   | 10. 用户发送命令                                              |
   | {"event":"send command","args":["say Hello"]}                |
   |-------------------------------------------------------------->|
   |<--------------------------------------------------------------|
   |    {"event":"console output","args":["[Server] Hello"]}       |
   |                                                               |
   | 11. 电源操作 (同一连接)                                      |
   | {"event":"set state","args":["restart"]}                     |
   |-------------------------------------------------------------->|
   |<--------------------------------------------------------------|
   |    {"event":"status","args":["stopping"]}                     |
   |<--------------------------------------------------------------|
   |    {"event":"console output","args":["Stopping..."]}          |
   |<--------------------------------------------------------------|
   |    {"event":"status","args":["starting"]}                     |
   |<--------------------------------------------------------------|
   |    {"event":"status","args":["running"]}                      |
   |                                                               |
   | 12. Token 即将过期 (剩余<3分钟)                               |
   |<--------------------------------------------------------------|
   |    {"event":"token expiring"}                                 |
   |                                                               |
   | 13. 刷新 Token (回到 Panel)                                  |
   |------------------------------>|                               |
   |<------------------------------|                               |
   | 14. 重新认证 (同一连接无需重建)                               |
   | {"event":"auth","args":["<NEW_JWT>"]}                        |
   |-------------------------------------------------------------->|
   |<--------------------------------------------------------------|
   |    {"event":"auth success"}                                   |
```

## 7. 关键安全设计

### 7.1 权限嵌入 JWT
- JWT 中包含 `permissions` 声明，Wings 可直接验证操作权限
- 无需每次操作都回调 Panel 鉴权，降低延迟

### 7.2 短有效期 Token
- Token 有效期仅 10 分钟
- 支持自动续期，降低 Token 泄露风险
- Wings 提前 3 分钟通知即将过期

### 7.3 防重放机制
- JWT 包含 `unique_id` 随机字符串
- `nbf` (not before) 防止时间偏移攻击
- `jti` 唯一标识可用于黑名单机制

### 7.4 Node 密钥签名
- 使用 Node 独立的密钥签名 JWT
- 每个 Node 有不同的密钥，隔离性好
- 即使一个 Node 密钥泄露，不影响其他 Node

### 7.5 细粒度权限控制
JWT `permissions` 声明包含具体权限，Wings 可精确控制：
- `control.console` - 能否发送命令
- `control.start`/`control.stop`/`control.restart` - 电源操作权限
- `admin.websocket.errors` - 能否查看详细错误
- `admin.websocket.install` - 能否查看安装日志
- `admin.websocket.transfer` - 能否查看迁移日志

## 8. 错误处理机制

### 8.1 JWT 错误处理

**文件**: `WebsocketHandler.tsx:52-63`

```typescript
socket.on('jwt error', (error: string) => {
    setConnectionState(false);
    console.warn('JWT validation error from wings:', error);

    if (reconnectErrors.find((v) => error.toLowerCase().indexOf(v) >= 0)) {
        updateToken(uuid, socket);  // 可恢复错误，自动刷新 Token
    } else {
        setError(
            'There was an error validating the credentials provided for the websocket. Please refresh the page.'
        );
    }
});
```

可自动恢复的错误:
- `jwt: exp claim is invalid` (Token 过期)
- `jwt: created too far in past (denylist)` (Token 被拉黑)

### 8.2 连接错误
- `SOCKET_ERROR`: 连接错误，显示 "connecting" 状态，自动重连
- `SOCKET_CONNECT_ERROR`: 重连次数耗尽 (20次)，提示用户刷新页面
- `4409` 关闭码: 服务器已暂停，停止重连

### 8.3 守护进程错误

**文件**: `Console.tsx:89-92`
```typescript
const handleDaemonErrorOutput = (line: string) =>
    terminal.writeln(
        TERMINAL_PRELUDE + '\u001b[1m\u001b[41m' + line.replace(/(?:\r\n|\r|\n)$/im, '') + '\u001b[0m'
    );
```

## 9. 连接复用架构总结

```
┌──────────────────────────────────────────────────────────────────┐
│              Single WebSocket Connection (复用)                 │
└─────────┬───────────────────────────┬───────────────────────────┘
          │                           │
          ▼                           ▼
┌───────────────────┐     ┌───────────────────────────┐
│   服务端推送      │     │      客户端请求           │
│   (Read-Only)     │     │      (Write)              │
│                   │     │                           │
│ - console output  │     │ - auth (鉴权)            │
│ - install output  │     │ - send logs (历史日志)    │
│ - transfer logs   │     │ - send stats (统计)       │
│ - daemon message  │     │ - send command (命令)     │
│ - daemon error    │     │ - set state (电源)        │
│ - status          │     │                           │
│ - stats           │     └───────────────────────────┘
│ - install started │
│ - install completed│
│ - transfer status │
│ - backup completed│
│ - backup restore  │
│   completed       │
└───────────────────┘
          │
          ▼
┌───────────────────────────────────────────────────┐
│  多组件事件分发 (EventEmitter)                    │
│                                                   │
│  Console 组件 ──>  日志输出、状态显示             │
│  StatGraphs 组件 ──>  资源统计图表               │
│  PowerButtons 组件 ──>  电源操作按钮             │
│  InstallListener ──>  安装状态监听               │
│  TransferListener ──>  迁移状态监听              │
│  Features (EULA/GSL/etc) ──>  特殊日志匹配       │
└───────────────────────────────────────────────────┘
```

## 10. 核心文件索引

| 文件路径 | 功能 |
|---------|------|
| `app/Http/Controllers/Api/Client/Servers/WebsocketController.php` | Token 签发控制器 |
| `app/Services/Nodes/NodeJWTService.php` | JWT 生成服务 |
| `app/Services/Servers/GetUserPermissionsService.php` | 用户权限获取 |
| `app/Models/Permission.php` | 权限常量定义 |
| `resources/scripts/plugins/Websocket.ts` | WebSocket 客户端封装 |
| `resources/scripts/components/server/WebsocketHandler.tsx` | WebSocket 连接管理 |
| `resources/scripts/api/server/getWebsocketToken.ts` | Token 获取 API |
| `resources/scripts/components/server/console/Console.tsx` | 控制台组件 (日志订阅、命令发送) |
| `resources/scripts/components/server/console/PowerButtons.tsx` | 电源按钮组件 |
| `resources/scripts/components/server/console/StatGraphs.tsx` | 统计图表组件 |
| `resources/scripts/components/server/events.ts` | Socket 事件常量 |
| `resources/scripts/plugins/useWebsocketEvent.ts` | 事件订阅 Hook |
| `resources/scripts/state/server/socket.ts` | Socket 状态管理 |
| `resources/scripts/components/server/InstallListener.tsx` | 安装事件监听 |
| `resources/scripts/components/server/TransferListener.tsx` | 迁移事件监听 |
| `routes/api-client.php` | API 路由定义 |
