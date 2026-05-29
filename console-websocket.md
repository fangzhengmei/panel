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
| `SocketEvent.TRANSFER_STATUS` | `transfer status` | `starting`\|`pending`\|`processing`\|`success`\|`completed`\|`failed`\|`failure` | 迁移状态 | `WebsocketHandler.tsx:65`, `TransferListener.tsx:11`, `Console.tsx:175` |
| `SocketEvent.DAEMON_MESSAGE` | `daemon message` | message | 守护进程消息 | `Console.tsx:176` |
| `SocketEvent.DAEMON_ERROR` | `daemon error` | error message | 守护进程错误 | `Console.tsx:177`, `WebsocketHandler.tsx:46` |
| `SocketEvent.BACKUP_COMPLETED` | `backup completed` | 无 | 备份完成 | - |
| `SocketEvent.BACKUP_RESTORE_COMPLETED` | `backup restore completed` | 无 | 备份恢复完成 | `InstallListener.tsx:12` |

> **transfer status 发送端说明**: 所有 `transfer status` 事件均由 **Wings 端通过 WebSocket 直接推送到前端**，Panel 不参与事件发送。「代码可直接证明」（Panel 代码全面搜索未发现任何发送 "transfer status" 的逻辑，且 WebSocket 连接为前端与 Wings 的直连，无 Panel 中间转发）

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

### 4.5 TRANSFER STATUS 事件状态全集与证据链

`transfer status` 事件的参数是一个字符串，在服务器迁移流程的不同阶段推送不同状态。前端有三个组件分别监听该事件，各自处理不同状态。

#### 证据等级说明

每个结论标注以下等级之一：
- **「代码可直接证明」**：Panel 代码中有明确的代码逻辑可直接验证
- **「基于跨服务推断」**：Panel 代码中无直接证据，需结合 Wings 职责和流程逻辑推断

---

#### 状态枚举与发送端归属

| 状态值 | 发送端归属 | 触发时机 | WebsocketHandler | TransferListener | Console |
|--------|-----------|----------|-----------------|------------------|---------|
| `starting` | 源节点 Wings「基于跨服务推断」 | 源节点开始将服务器归档时推送 | **忽略** (return)「代码可直接证明」 | 未匹配 | 未匹配 |
| `pending` | 源节点 Wings「基于跨服务推断」 | 迁移请求已创建，等待源节点处理 | 触发重连「代码可直接证明」 | `isTransferring = true`「代码可直接证明」 | 未匹配 |
| `processing` | 源节点 Wings「基于跨服务推断」 | 源节点正在执行归档操作 | 触发重连「代码可直接证明」 | `isTransferring = true`「代码可直接证明」 | 未匹配 |
| `success` | 目标节点 Wings「基于跨服务推断」 | 目标节点接收完成数据后推送 | **忽略** (return)「代码可直接证明」 | 未匹配 | 未匹配 |
| `completed` | 目标节点 Wings「基于跨服务推断」 | Panel 数据库更新完成后推送 | 触发重连「代码可直接证明」 | 刷新服务器信息 (getServer)「代码可直接证明」 | 未匹配 |
| `failed` | 源节点或目标节点 Wings「基于跨服务推断」 | 迁移失败（Panel 收到回调后确认） | 触发重连「代码可直接证明」 | `isTransferring = false`「代码可直接证明」 | 未匹配 |
| `failure` | 源节点或目标节点 Wings「基于跨服务推断」 | 迁移失败（节点端直接推送） | 触发重连「代码可直接证明」 | 未匹配 | 终端显示 "Transfer has failed."「代码可直接证明」 |

---

#### 核心结论与证据链

**所有 transfer status 事件均由 Wings 端推送，Panel 不参与事件发送**
- 「代码可直接证明」：Panel 代码全面搜索（`app/` 目录）未发现任何发送 "transfer status" 的逻辑
- 「代码可直接证明」：WebSocket 连接为前端与 Wings 的直连，无 Panel 中间转发
- 补充：Panel 仅通过 HTTP 回调 `/api/remote/servers/{uuid}/transfer/success` 和 `/failure` 接收 Wings 的迁移结果通知

---

**starting 发送端证据边界**
- 「基于跨服务推断」：归档（archive）是源节点的职责，只有源节点知道何时开始归档
- 「代码可直接证明」：WebsocketHandler 对 `starting` 直接 `return`，说明此时连接仍有效（仍指向源节点）
- 「代码可直接证明」：`WebsocketHandler.tsx:65-71` 明确将 `starting` 和 `success` 列为不触发重连的状态

---

**success 发送端证据边界**
- 「基于跨服务推断」：只有目标节点知道何时完成数据接收
- 「代码可直接证明」：`Remote/ServerTransferController.php:74-78` 注释明确 "Only the new node communicates a successful state to the panel"
- 「代码可直接证明」：WebsocketHandler 对 `success` 直接 `return`，不触发重连
- 设计意图：`success` 仅表示 Wings 层面数据传输完成，此时 Panel 数据库可能尚未更新 `node_id`，等 `completed` 再重连可确保 Token 正确指向目标节点

---

**completed 发送端证据边界**
- 「基于跨服务推断」：应是目标节点在确认 Panel 数据库更新完成后推送
- 「代码可直接证明」：`TransferListener.tsx:24-26` 收到 `completed` 后调用 `getServer(uuid)` 刷新，说明此时 Panel 数据已更新
- 「代码可直接证明」：`ServerTransformer.php:81` `is_transferring = !is_null($server->transfer)`，刷新后服务器信息已更新到新节点

---

#### starting 与 success 的完整语义

**`starting` — 源节点 Wings 推送**
- 触发时机：源节点开始对服务器进行归档（archive）操作时
- 语义标志：迁移流程从"准备阶段"进入"执行阶段"
- 处理行为：WebsocketHandler 直接 `return` 忽略，因为此时连接仍然指向源节点，无需重连
- 后续动作：归档完成后，源节点会将 `server_transfer.archived` 标记为 `true`「基于跨服务推断」

**`success` — 目标节点 Wings 推送**
- 触发时机：目标节点成功接收并解压服务器数据后
- 语义标志：迁移在 Wings 层面已成功，但 Panel 端的数据库更新可能尚未完成
- 处理行为：WebsocketHandler 直接 `return` 忽略，因为后续会有 `completed` 事件触发最终重连
- 后续动作：目标节点回调 Panel 的 `/api/remote/servers/{uuid}/transfer/success`，Panel 更新数据库（`successful = true`，切换 `node_id`）「代码可直接证明」

---

#### 多组件监听分工

```
Wings 推送 transfer status 事件
        │
        ▼
   EventEmitter 广播
        │
        ├─→ WebsocketHandler.tsx:65-77 「代码可直接证明」
        │     - starting/success → return（忽略，不重连）
        │     - pending/processing/completed/failed/failure → 关闭连接 + 重新连接
        │
        ├─→ TransferListener.tsx:11-28 「代码可直接证明」
        │     - pending/processing → isTransferring = true（前端状态）
        │     - failed → isTransferring = false（前端状态）
        │     - completed → 刷新服务器信息 (getServer(uuid))
        │
        └─→ Console.tsx:80-87, 175 「代码可直接证明」
              - failure → 终端显示 "Transfer has failed."
              - 其他状态 → 无处理 (switch 无匹配 case)
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

## 9. 服务器迁移与 WebSocket 重连机制

### 9.1 三个核心状态字段定义与证据链

迁移流程由三个关键状态字段串联，每个字段的定义和设置时机如下：

| 字段 | 位置 | 定义 | 设置时机 | 证据等级 |
|------|------|------|----------|----------|
| **`is_transferring`** | 后端 API 返回 (`ServerTransformer.php:81`) | `!is_null($server->transfer)` | 只要 transfer 记录存在即为 `true` | 「代码可直接证明」 |
| **`archived`** | `server_transfer` 表 | 源节点是否已完成归档 | 源节点归档完成后设置 | 「基于跨服务推断」（Panel 代码只读不写） |
| **`successful`** | `server_transfer` 表 | 迁移是否成功 | Panel 收到目标节点 success 回调后设置 `true` | 「代码可直接证明」 |

> **重要区分**:
> - **后端 `is_transferring`**（API 返回）: 由 `ServerTransformer.php:81` 计算，`= !is_null($server->transfer)`「代码可直接证明」
> - **前端 `isTransferring`**（React State）: 由 `TransferListener.tsx:13` 根据 `transfer status` 事件设置，与后端字段是两个独立概念「代码可直接证明」

---

### 9.2 迁移阶段与字段状态流转图（含证据等级）

```
┌──────────────────────────────────────────────────────────────────────────────────────┐
│                      迁移阶段与字段状态流转图（含证据等级）                            │
└──────────────────────────────────────────────────────────────────────────────────────┘

  阶段 0: 正常状态
    ├─ transfer 记录: 不存在
    ├─ is_transferring: false  「代码可直接证明」
    ├─ archived: N/A
    └─ successful: N/A

  阶段 1: 管理员发起迁移
    ├─ Panel 创建 server_transfer 记录
    ├─ transfer 记录: 存在
    ├─ is_transferring: true   「代码可直接证明」
    ├─ archived: false
    └─ successful: null

  阶段 2: 源节点开始归档（starting 事件）
    ├─ transfer status: starting  「基于跨服务推断：源节点 Wings 推送」
    ├─ WebsocketHandler: return（不重连） 「代码可直接证明」
    ├─ 连接仍指向源节点，有效
    ├─ archived: false
    └─ successful: null

  阶段 3: 源节点归档中（processing 事件）
    ├─ transfer status: processing  「基于跨服务推断：源节点 Wings 推送」
    ├─ WebsocketHandler: 触发重连  「代码可直接证明」
    ├─ 重连请求 GET /websocket
    ├─ Panel: archived=false → Token 指向源节点  「代码可直接证明」
    ├─ 重连后仍连源节点
    ├─ archived: false
    └─ successful: null

  阶段 4: 源节点归档完成
    ├─ archived: true  「基于跨服务推断：源节点 Wings 设置」
    ├─ 后续重连 Token 将指向目标节点  「代码可直接证明」
    └─ successful: null

  阶段 5: 源节点传输数据 → 目标节点
    ├─ 期间可能推送 processing / transfer logs 事件
    ├─ archived: true
    └─ successful: null

  阶段 6: 目标节点接收完成（success 事件）
    ├─ transfer status: success  「基于跨服务推断：目标节点 Wings 推送」
    ├─ 目标节点回调 Panel: POST /api/remote/servers/{uuid}/transfer/success
    ├─ WebsocketHandler: return（不重连） 「代码可直接证明」
    │   设计意图：此时 Panel 数据库可能尚未更新 node_id，等 completed 再重连
    ├─ archived: true
    └─ successful: null

  阶段 7: Panel 更新数据库
    ├─ Panel 收到目标节点 success 回调  「代码可直接证明」
    ├─ 更新: node_id → 目标节点 ID
    ├─ 更新: successful = true  「代码可直接证明：Remote/ServerTransferController.php:93」
    ├─ 后端 is_transferring: 仍为 true（transfer 记录还在）
    ├─ archived: true
    └─ successful: true

  阶段 8: 迁移完成（completed 事件）
    ├─ transfer status: completed  「基于跨服务推断：目标节点 Wings 推送」
    ├─ 注意：**不是 Panel 推送**，Panel 无发送 transfer status 代码
    ├─ WebsocketHandler: 触发重连  「代码可直接证明」
    ├─ 重连请求 GET /websocket
    ├─ Panel: node_id 已更新 → Token 指向目标节点  「代码可直接证明」
    ├─ 重连后连接到目标节点
    ├─ TransferListener: getServer(uuid) 刷新服务器信息  「代码可直接证明」
    ├─ archived: true
    └─ successful: true

  阶段 9: 最终状态（transfer 记录被清理后）
    ├─ transfer 记录: 不存在（或被软删除）
    ├─ is_transferring: false  「代码可直接证明」
    └─ 服务器已在目标节点正常运行
```

---

### 9.3 WebsocketController 中的 Node 路由逻辑

**文件**: `WebsocketController.php:42-53`

```php
$node = $server->node;
if (!is_null($server->transfer)) {
    // 需要管理员权限才能在迁移期间获取 WebSocket Token
    if (!in_array('admin.websocket.transfer', $permissions)) {
        throw new HttpForbiddenException('You do not have permission to view server transfer logs.');
    }

    // 关键：归档完成后，Token 指向目标节点
    // 「代码可直接证明」：读 archived 字段决定 Node
    if ($server->transfer->archived) {
        $node = $server->transfer->newNode;
    }
}
```

**路由逻辑证据链**:
| 条件 | Token 指向 | 证据等级 |
|------|-----------|----------|
| `transfer` 为 `null` | 服务器当前 Node | 「代码可直接证明」 |
| `transfer` 存在 + `archived == false` | 源节点 | 「代码可直接证明」 |
| `transfer` 存在 + `archived == true` | 目标节点 | 「代码可直接证明」 |

> **`archived` 字段设置证据边界**:
> - Panel 代码中**只读不写** `archived` 字段（全面搜索 `app/` 目录，未找到任何 `->archived = true` 或 `update(['archived' => ...])` 的代码）
> - 因此：`archived = true` 必然由**源节点 Wings** 回调或直接设置 「基于跨服务推断」

---

### 9.4 迁移流程与 WebSocket 重连时序（修正版）

**重要修正**: 之前的时序图错误地将 `completed` 事件标注为"Panel 通知客户端"，实际所有 `transfer status` 事件均由 Wings 推送。

```
Source Node Wings              Panel              Target Node Wings       Browser (WebSocket)
     |                           |                       |                       |
     | 1. 管理员发起迁移请求     |                       |                       |
     |                           |  创建 ServerTransfer  |                       |
     |                           |  archived = false     |                       |
     |                           |  successful = null    |                       |
     |                           |  is_transferring = true [代码可直接证明]     |
     |<-- notify(transfer) ------|                       |                       |
     |                           |                       |                       |
     | 2. 源节点开始归档         |                       |                       |
     | push: transfer status     |                       |                       |
     |   args: ["starting"]      |                       |                       |
     | [基于跨服务推断: 源节点发送]|                       |                       |
     |-------------------------------------------------->|                       |
     |                           |                       |  收到 starting        |
     |                           |                       |  WebsocketHandler:    |
     |                           |                       |  return (忽略)        |
     |                           |                       |  [代码可直接证明]     |
     |                           |                       |  仍在源节点，连接有效 |
     |                           |                       |                       |
     | 3. 源节点归档进行中       |                       |                       |
     | push: transfer status     |                       |                       |
     |   args: ["processing"]    |                       |                       |
     |-------------------------------------------------->|                       |
     |                           |                       |  触发重连！           |
     |                           |                       |  socket.close()       |
     |                           |                       |  setInstance(null)    |
     |                           |                       |  connect(uuid)        |
     |                           |                       |    │                  |
     |                           |                       |    │ GET /websocket   |
     |                           |                       |    │ Panel 此时       |
     |                           |                       |    │ archived=false   |
     |                           |                       |    │ → Token指向源节点 |
     |                           |                       |    │ [代码可直接证明]  |
     |                           |                       |    │                  |
     |                           |                       |    ├── 重新连接源节点 |
     |                           |                       |    └── 恢复日志接收  |
     |                           |                       |                       |
     | 4. 源节点完成归档         |                       |                       |
     |    设置 archived = true   |                       |                       |
     |    [基于跨服务推断]        |                       |                       |
     |                           |                       |                       |
     | 5. 源节点传输数据到目标   |                       |                       |
     |---------------------------|---------------------->|                       |
     |                           |                       |                       |
     | 6. 目标节点接收完成       |                       |                       |
     |    push: transfer status  |                       |                       |
     |      args: ["success"]    |                       |                       |
     |    [基于跨服务推断: 目标节点发送]                |                       |
     |                           |                       |---------------------->|
     |                           |                       |  收到 success         |
     |                           |                       |  WebsocketHandler:    |
     |                           |                       |  return (忽略)        |
     |                           |                       |  [代码可直接证明]     |
     |                           |                       |  等待 completed       |
     |                           |                       |                       |
     | 7. 目标节点通知 Panel     |                       |                       |
     |                           |<----------------------|                       |
     |                           | POST /api/remote/     |                       |
     |                           |   servers/{uuid}/     |                       |
     |                           |   transfer/success    |                       |
     |                           | [代码可直接证明]       |                       |
     |                           |                       |                       |
     |                           | 更新数据库：          |                       |
     |                           | - node_id → new_node  |                       |
     |                           | - successful = true   |                       |
     |                           |   [代码可直接证明]     |                       |
     |                           |                       |                       |
     | 8. 目标节点推送 completed |                       |                       |
     |                           |                       | push: transfer status |
     |                           |                       |   args: ["completed"] |
     |                           |                       | [基于跨服务推断: 目标节点发送]|
     |                           |                       |---------------------->|
     |                           |                       |  触发重连！           |
     |                           |                       |  socket.close()       |
     |                           |                       |  setInstance(null)    |
     |                           |                       |  connect(uuid)        |
     |                           |                       |    │                  |
     |                           |                       |    │ GET /websocket   |
     |                           |                       |    │ Panel 此时       |
     |                           |                       |    │ node_id已更新    |
     |                           |                       |    │ → Token指向目标  |
     |                           |                       |    │ [代码可直接证明]  |
     |                           |                       |    │                  |
     |                           |                       |    ├── 连接目标节点   |
     |                           |                       |    └── 接收目标节点日志|

   注意：步骤 8 的 completed 事件**不是 Panel 推送**，是目标节点 Wings 推送。
   Panel 代码中无任何发送 transfer status 的逻辑。「代码可直接证明」
```

---

### 9.5 WebsocketHandler 重连逻辑详解

**文件**: `WebsocketHandler.tsx:65-77`

```typescript
socket.on('transfer status', (status: string) => {
    // starting 和 success 不会触发重连
    // - starting: 源节点刚开始归档，当前连接仍指向源节点，连接有效
    // - success: 目标节点完成接收，但 Panel 可能尚未更新 node_id
    //           等 completed 事件时再重连，确保 Panel 已切换节点
    // 「代码可直接证明」
    if (status === 'starting' || status === 'success') {
        return;
    }

    // 其他状态（pending/processing/completed/failed/failure）触发重连
    socket.close();                    // 关闭当前连接
    setError('connecting');            // 显示连接中提示
    setConnectionState(false);         // 标记连接断开
    setInstance(null);                 // 清除 Socket 实例
    connect(uuid);                     // 重新获取 Token 并连接
});
```

**重连后的 Node 路由关键点**:
| 重连触发时机 | archived 状态 | Token 指向 | 证据等级 |
|-------------|--------------|-----------|----------|
| processing（归档中） | `false` | 源节点 | 「代码可直接证明」 |
| completed（迁移完成） | `true` | 目标节点 | 「代码可直接证明」 |
| failed/failure（失败） | 取决于失败阶段 | 原节点或目标节点 | 「代码可直接证明」 |

---

### 9.6 迁移失败场景

```
Source/Target Node Wings        Panel              Browser
     |                           |                    |
     | push: transfer status     |                    |
     |   args: ["failed"]        |                    |
     | [基于跨服务推断]           |                    |
     |------------------------------------------------->|
     |                           |                    | 触发重连 [代码可直接证明]
     |                           |                    | socket.close()
     |                           |                    | connect(uuid)
     |                           |                    |   │
     |                           |                    |   │ GET /websocket
     |                           |                    |   │ Panel: transfer 记录
     |                           |                    |   │ 可能已被清理或标记
     |                           |                    |   │ → 连接回原节点
     |                           |                    |
     | push: transfer status     |                    |
     |   args: ["failure"]       |                    |
     | [基于跨服务推断]           |                    |
     |------------------------------------------------->|
     |                           |                    | 触发重连（同上）
     |                           |                    | Console 终端显示:
     |                           |                    | "Transfer has failed."
     |                           |                    | [代码可直接证明]
     |                           |                    | TransferListener:
     |                           |                    | isTransferring = false
     |                           |                    | [代码可直接证明]
```

**失败回调证据链**:
- 目标节点或源节点失败 → 回调 Panel `/api/remote/servers/{uuid}/transfer/failure` 「代码可直接证明」
- Panel 设置 `successful = false` 「代码可直接证明：`Remote/ServerTransferController.php:121`」
- 清理新分配的端口等资源

---

### 9.7 Console 组件的迁移感知

**文件**: `Console.tsx:80-87, 180-184`

Console 组件对迁移的处理体现在两方面：

1. **迁移失败终端提示** (`Console.tsx:80-87`)「代码可直接证明」:
```typescript
const handleTransferStatus = (status: string) => {
    switch (status) {
        case 'failure':
            terminal.writeln(TERMINAL_PRELUDE + 'Transfer has failed.\u001b[0m');
            return;
    }
};
```

2. **迁移期间不清空终端** (`Console.tsx:180-184`)「代码可直接证明」:
```typescript
if (connected && instance) {
    if (!isTransferring) {
        terminal.clear();    // 非迁移时：重连清空终端
    }
    // 迁移时：保留终端历史，避免丢失迁移日志
}
```

> `isTransferring` 是前端 React 状态，由 `TransferListener.tsx:13` 根据 `transfer status` 事件设置，与后端 `is_transferring` API 字段是两个独立概念。

---

### 9.8 迁移权限要求

迁移期间获取 WebSocket Token 需要额外权限：

**文件**: `WebsocketController.php:44-46`「代码可直接证明」
```php
if (!in_array('admin.websocket.transfer', $permissions)) {
    throw new HttpForbiddenException('You do not have permission to view server transfer logs.');
}
```

- 普通用户在迁移期间**无法获取** WebSocket Token
- 仅管理员（`root_admin`）拥有 `admin.websocket.transfer` 权限
- 这意味着迁移期间只有管理员能在控制台看到迁移日志

## 10. 连接复用架构总结

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

## 11. 核心文件索引

| 文件路径 | 功能 |
|---------|------|
| `app/Http/Controllers/Api/Client/Servers/WebsocketController.php` | Token 签发控制器（含迁移 Node 路由） |
| `app/Services/Nodes/NodeJWTService.php` | JWT 生成服务 |
| `app/Services/Servers/GetUserPermissionsService.php` | 用户权限获取 |
| `app/Models/Permission.php` | 权限常量定义 |
| `app/Models/ServerTransfer.php` | 服务器迁移模型 |
| `app/Transformers/Api/Client/ServerTransformer.php` | 服务器数据转换（含 is_transferring 定义） |
| `app/Http/Controllers/Api/Remote/Servers/ServerTransferController.php` | 迁移成功/失败回调（Wings→Panel） |
| `app/Http/Controllers/Admin/Servers/ServerTransferController.php` | 管理员发起迁移 |
| `app/Repositories/Wings/DaTransferRepository.php` | Panel→源节点通知迁移 |
| `resources/scripts/plugins/Websocket.ts` | WebSocket 客户端封装 |
| `resources/scripts/components/server/WebsocketHandler.tsx` | WebSocket 连接管理（含迁移重连） |
| `resources/scripts/api/server/getWebsocketToken.ts` | Token 获取 API |
| `resources/scripts/components/server/console/Console.tsx` | 控制台组件（日志订阅、命令发送、迁移感知） |
| `resources/scripts/components/server/console/PowerButtons.tsx` | 电源按钮组件 |
| `resources/scripts/components/server/console/StatGraphs.tsx` | 统计图表组件 |
| `resources/scripts/components/server/events.ts` | Socket 事件常量 |
| `resources/scripts/plugins/useWebsocketEvent.ts` | 事件订阅 Hook |
| `resources/scripts/state/server/socket.ts` | Socket 状态管理 |
| `resources/scripts/components/server/InstallListener.tsx` | 安装事件监听 |
| `resources/scripts/components/server/TransferListener.tsx` | 迁移事件监听 |
| `resources/scripts/components/server/ConflictStateRenderer.tsx` | 冲突状态渲染（含迁移中提示） |
| `routes/api-client.php` | API 路由定义 |
