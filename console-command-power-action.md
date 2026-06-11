# Pterodactyl Panel 控制台命令与电源动作 — 代码走向分析

> **范围说明**：本仓库仅包含 **Panel**（PHP Laravel + React 前端）代码，不包含 Wings（Go 守护进程）。因此，文档将严格分为两部分：
> - ✅ **Panel 可证实**：仓库内有对应代码，引用具体文件/行号
> - ⚠️ **Wings 侧推断**：Panel 调用协议暗示的行为，无本地代码佐证，仅供参考

## 0. 目录

- [1. 架构概览（Panel 可证实部分）](#1-架构概览panel-可证实部分)
- [2. 前端：电源按钮与命令输入框](#2-前端电源按钮与命令输入框)
  - [2.1 电源按钮：PowerButtons.tsx](#21-电源按钮powerbuttonstsx)
  - [2.2 命令输入框：Console.tsx](#22-命令输入框consoletsx)
  - [2.3 前端 WebSocket 封装](#23-前端-websocket-封装)
  - [2.4 WebSocket Token 获取与刷新](#24-websocket-token-获取与刷新)
  - [2.5 前端状态模型（ServerStatus vs Server.status）](#25-前端状态模型serverstatus-vs-serverstatus)
- [3. Panel 后端：路由注册与中间件管道](#3-panel-后端路由注册与中间件管道)
  - [3.1 路由入口](#31-路由入口)
  - [3.2 中间件栈](#32-中间件栈)
- [4. 节流策略（Panel 可证实）](#4-节流策略panel-可证实)
  - [4.1 全局限流](#41-全局限流)
  - [4.2 资源级限流（仅 WebSocket）](#42-资源级限流仅-websocket)
  - [4.3 前端 UI 防重](#43-前端-ui-防重)
- [5. 白名单与语义鉴别（Panel 可证实）](#5-白名单与语义鉴别panel-可证实)
  - [5.1 电源动作白名单](#51-电源动作白名单)
  - [5.2 控制台命令白名单](#52-控制台命令白名单)
  - [5.3 调度任务白名单](#53-调度任务白名单)
- [6. 权限鉴权模型（Panel 可证实）](#6-权限鉴权模型panel-可证实)
  - [6.1 请求层权限解析](#61-请求层权限解析)
  - [6.2 权限判定策略（ServerPolicy）](#62-权限判定策略serverpolicy)
  - [6.3 服务器状态门禁（AuthenticateServerAccess）](#63-服务器状态门禁authenticateserveraccess)
- [7. Panel → Wings：REST HTTP 调用链（Panel 可证实）](#7-panel--wingsrest-http-调用链panel-可证实)
  - [7.1 DaemonRepository 基类](#71-daemonrepository-基类)
  - [7.2 DaemonPowerRepository：电源动作下发](#72-daemonpowerrepository电源动作下发)
  - [7.3 DaemonCommandRepository：控制台命令下发](#73-daemoncommandrepository控制台命令下发)
  - [7.4 Activity 审计日志埋点](#74-activity-审计日志埋点)
- [8. Panel → Wings：WebSocket 通道（Panel 可证实）](#8-panel--wingswebsocket-通道panel-可证实)
  - [8.1 JWT 签发服务（NodeJWTService）](#81-jwt-签发服务nodejwtservice)
  - [8.2 WebSocket 事件枚举](#82-websocket-事件枚举)
- [9. Wings 侧：接收约定与行为（⚠️ 推断）](#9-wings-侧接收约定与行为️-推断)
  - [9.1 REST Endpoint 约定](#91-rest-endpoint-约定)
  - [9.2 WebSocket Event 约定](#92-websocket-event-约定)
  - [9.3 并发安全与内部实现（⚠️ 推断，无本地代码）](#93-并发安全与内部实现️-推断无本地代码)
- [10. 调度系统：计划任务中的电源/命令（Panel 可证实）](#10-调度系统计划任务中的电源命令panel-可证实)
  - [10.1 ProcessScheduleService：事务入队 + only_when_online](#101-processscheduleservice事务入队--only_when_online)
  - [10.2 RunTaskJob：按序执行与失败降级](#102-runtaskjob按序执行与失败降级)
- [11. 失败处理与状态回滚（Panel 可证实）](#11-失败处理与状态回滚panel-可证实)
  - [11.1 DaemonConnectionException：异常封装](#111-daemonconnectionexception异常封装)
  - [11.2 CommandController：502 → 用户可读错误](#112-commandcontroller502--用户可读错误)
  - [11.3 电源动作失败：无回滚，乐观状态模型](#113-电源动作失败无回滚乐观状态模型)
  - [11.4 调度任务失败：continue_on_failure 开关](#114-调度任务失败continue_on_failure-开关)
  - [11.5 Panel 数据库：乐观设计，无状态回滚](#115-panel-数据库乐观设计无状态回滚)
- [12. 客服视角：Power Action 语义对照表](#12-客服视角power-action-语义对照表)
- [13. 关键文件索引（Panel 代码）](#13-关键文件索引panel-代码)
- [14. 附：客服 FAQ 速查](#14-附客服-faq-速查)

---

## 1. 架构概览（Panel 可证实部分）

```
 ┌─────────────────────────────────────────────────────────────────────┐
 │                    ✅ 浏览器前端 (React) — 代码可证实                  │
 │  ┌──────────────────────┐     ┌──────────────────────────────┐      │
 │  │  PowerButtons.tsx    │     │  Console.tsx                 │      │
 │  │  Start/Stop/         │     │  (命令输入框 + xterm.js)     │      │
 │  │  Restart/Kill        │     │                              │      │
 │  └──────────┬───────────┘     └──────────────┬───────────────┘      │
 │             │ WebSocket                        │ WebSocket          │
 │             │ "set state"                      │ "send command"     │
 │             ▼                                  ▼                    │
 │  ┌──────────────────────────────────────────────────────────┐       │
 │  │  Websocket.ts (Sockette 封装)                            │       │
 │  │  - 重连：最多 20 次, 指数退避                             │       │
 │  └───────────────────────┬──────────────────────────────────┘       │
 │                         │  REST (token 获取)                         │
 │                         ▼                                            │
 │          GET /api/client/servers/{uuid}/websocket                    │
 │          → 返回 JWT(10min) + socket URL                              │
 └─────────────────────────┬───────────────────────────────────────────┘
                           │ HTTPS (withCredentials: true)
 ┌─────────────────────────┼───────────────────────────────────────────┐
 │              ✅ Panel (PHP Laravel) — 代码可证实                      │
 │  ┌──────────────────────▼──────────────────────────────┐            │
 │  │  throttle:api.client (256 req/min / user)           │            │
 │  └──────────────────────┬──────────────────────────────┘            │
 │          ┌──────────────┼─────────────────┐                          │
 │          ▼              ▼                 ▼                          │
 │  POST /power     POST /command    GET /websocket                     │
 │  PowerController CommandController WebsocketController               │
 │          │              │                 │                          │
 │          └──────────────┴─────────────────┘                          │
 │                         │                                            │
 │          ┌──────────────▼──────────────────┐                         │
 │          │ DaemonPowerRepository            │                         │
 │          │ DaemonCommandRepository          │                         │
 │          │ (Guzzle + Node Bearer Token)     │                         │
 │          └──────────────┬──────────────────┘                         │
 └──────────────────────────┼──────────────────────────────────────────┘
                            │ HTTPS
 ┌──────────────────────────▼──────────────────────────────────────────┐
 │            ⚠️ Wings (Go 守护进程) — 本仓库无代码，以下为推断            │
 │  POST /api/servers/{uuid}/power                                      │
 │  POST /api/servers/{uuid}/commands                                   │
 │  GET  /api/servers/{uuid}/ws  (WebSocket Upgrade)                    │
 │         ↓                                                            │
 │  游戏服进程 (Docker)                                                  │
 └──────────────────────────────────────────────────────────────────────┘
```

---

## 2. 前端：电源按钮与命令输入框

### 2.1 电源按钮：PowerButtons.tsx

**✅ Panel 可证实**：[PowerButtons.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/resources/scripts/components/server/console/PowerButtons.tsx)

```tsx
// L14-17: 取前端 store 中的 status（不是后端 Server.status）
const status = ServerContext.useStoreState((state) => state.status.value);
const killable = status === 'stopping';

// L23-30: 点击处理
const onButtonClick = (action: PowerAction | 'kill-confirmed', e) => {
    e.preventDefault();
    if (action === 'kill') return setOpen(true);  // 二次确认弹窗
    if (instance) {
        setOpen(false);
        // ✅ 直接走 WebSocket 发送 "set state" 事件
        instance.send('set state', action === 'kill-confirmed' ? 'kill' : action);
    }
};
```

**4 种 Power Action 语义**（前端类型定义见 [ServerConsoleContainer.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/resources/scripts/components/server/console/ServerConsoleContainer.tsx)）：

| Action | 触发方式 | 实际发送值 |
|--------|---------|-----------|
| `start` | 点 Start 按钮 | `"start"` |
| `restart` | 点 Restart 按钮 | `"restart"` |
| `stop` | 点 Stop 按钮（status ≠ `stopping`） | `"stop"` |
| `kill` | ① Stop 过程中按钮变为 Kill ② 或二次确认弹窗后 | `"kill"` |

**权限渲染控制**：[Can.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/resources/scripts/components/elements/Can.tsx)
- Start：`<Can action={'control.start'}>`
- Restart：`<Can action={'control.restart'}>`
- Stop/Kill：`<Can action={'control.stop'}>`

**Kill 二次确认**：使用 `Dialog.Confirm`，文案：
> "Forcibly stopping a server can lead to data corruption."

### 2.2 命令输入框：Console.tsx

**✅ Panel 可证实**：[Console.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/resources/scripts/components/server/console/Console.tsx)

```tsx
// L97-124: 命令发送逻辑
const handleCommandKeyDown = (e: React.KeyboardEvent<HTMLInputElement>) => {
    // ↑/↓ 翻历史命令，最多 32 条
    if (e.key === 'ArrowUp') { /* ... */ }
    if (e.key === 'ArrowDown') { /* ... */ }

    const command = e.currentTarget.value;
    if (e.key === 'Enter' && command.length > 0) {
        setHistory(prev => [command, ...prev!].slice(0, 32));  // localStorage 持久化
        // ✅ WebSocket 发送 "send command" 事件
        instance && instance.send('send command', command);
        e.currentTarget.value = '';
    }
};
```

**渲染权限**（L66, L211）：
```tsx
const [canSendCommands] = usePermissions(['control.console']);
// 无权限则不渲染输入框
{canSendCommands && <div className={...}><input ... onKeyDown={handleCommandKeyDown} /></div>}
```

**禁用条件**（L218）：`disabled={!instance || !connected}` — WebSocket 未连接时不可发送。

### 2.3 前端 WebSocket 封装

**✅ Panel 可证实**：[Websocket.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/resources/scripts/plugins/Websocket.ts)

```ts
export class Websocket extends EventEmitter {
    connect(url): this {
        this.socket = new Sockette(url, {
            timeout: 1000,
            maxAttempts: 20,                  // 最多 20 次重连
            onmessage: (e) => {
                const { event, args } = JSON.parse(e.data);
                args ? this.emit(event, ...args) : this.emit(event);
            },
            onopen: () => this.authenticate(),  // 连接建立后立即发 auth
        });
    }

    // ✅ 发送协议：{ event: string, args: string[] }
    send(event: string, payload?: string | string[]) {
        this.socket?.json({ event, args: Array.isArray(payload) ? payload : [payload] });
    }
}
```

### 2.4 WebSocket Token 获取与刷新

**✅ Panel 可证实**：[WebsocketHandler.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/resources/scripts/components/server/WebsocketHandler.tsx)

流程：
```
1. getWebsocketToken(uuid) → GET /api/client/servers/{uuid}/websocket
   → 返回 { token: JWT(10min), socket: "wss://node:.../api/servers/{uuid}/ws" }

2. socket.setToken(token).connect(socket)

3. SOCKET_OPEN → authenticate() → send("auth", token)

4. Wings 校验通过 → "auth success"

5. Wings 推送 "token expiring"（剩 3 分钟时） → 重新取 token 并 setToken
```

**自动刷新关键代码**（[WebsocketHandler.tsx#L20-L30](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/resources/scripts/components/server/WebsocketHandler.tsx#L20-L30)）：
```ts
const updateToken = (uuid, socket) => {
    if (updatingToken) return;   // 防抖锁
    updatingToken = true;
    getWebsocketToken(uuid).then(data => socket.setToken(data.token, true));
};
```

### 2.5 前端状态模型（ServerStatus vs Server.status）

**✅ Panel 可证实**：[state/server/index.ts#L11](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/resources/scripts/state/server/index.ts#L11)

```ts
export type ServerStatus = 'offline' | 'starting' | 'stopping' | 'running' | null;
```

**关键区分**（Panel 代码证实的重要事实）：

| 来源 | 类型 | 可能值 | 更新方式 |
|------|------|--------|---------|
| 前端 `state.status.value` | `ServerStatus` | `'offline' \| 'starting' \| 'stopping' \| 'running' \| null` | WebSocket `status` 事件驱动（[WebsocketHandler.tsx#L44](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/resources/scripts/components/server/WebsocketHandler.tsx#L44)）|
| 后端 `Server.status`（数据库字段）| `string \| null` | `'installing' \| 'install_failed' \| 'reinstall_failed' \| 'suspended' \| 'restoring_backup' \| null` | Panel 数据库事务写入，**不存 running/offline** |

> ⚠️ **客服/开发重要事实**：Panel 数据库 `servers.status` 字段**从来不存 running 或 offline**。游戏服的运行状态完全由 Wings 通过 WebSocket 推送到前端 store，Panel 自身不维护权威副本。

---

## 3. Panel 后端：路由注册与中间件管道

### 3.1 路由入口

**✅ Panel 可证实**：[routes/api-client.php#L57-L74](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/routes/api-client.php#L57-L74)

```php
Route::group([
    'prefix' => '/servers/{server}',
    'middleware' => [
        ServerSubject::class,              // Activity 日志：标记 server_id
        AuthenticateServerAccess::class,   // ★ 核心鉴权中间件
        ResourceBelongsToServer::class,    // 子资源归属校验
    ],
], function () {
    Route::get('/websocket', Client\Servers\WebsocketController::class)
        ->middleware([ResourceLimit::Websocket->middleware()]);

    // ✅ 路由归属：/command → CommandController
    Route::post('/command', [Client\Servers\CommandController::class, 'index']);
    // ✅ 路由归属：/power → PowerController
    Route::post('/power',   [Client\Servers\PowerController::class, 'index']);
    // ...
});
```

### 3.2 中间件栈

**✅ Panel 可证实**：[Http/Kernel.php#L82-L85](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Http/Kernel.php#L82-L85)

```
throttle:api.client          ← 最外层：全局限流
└─ api 组
   ├─ EnsureStatefulRequests (Cookie/Session)
   ├─ auth:sanctum           (用户鉴权)
   ├─ IsValidJson
   ├─ TrackAPIKey
   ├─ RequireTwoFactorAuthentication
   └─ AuthenticateIPAccess
      └─ client-api 组
         └─ SubstituteClientBindings (用 uuid 解析 Server 模型)
            └─ servers/{server} 组
               ├─ ServerSubject
               ├─ AuthenticateServerAccess  ★ 服务器状态门禁
               └─ ResourceBelongsToServer
```

---

## 4. 节流策略（Panel 可证实）

### 4.1 全局限流

**✅ Panel 可证实**：[RouteServiceProvider.php#L93-L100](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Providers/RouteServiceProvider.php#L93-L100)

```php
RateLimiter::for('api.client', function (Request $request) {
    $key = optional($request->user())->uuid ?: $request->ip();
    return Limit::perMinutes(
        config('http.rate_limit.client_period'),   // 默认 1 分钟
        config('http.rate_limit.client')            // 默认 256 次
    )->by($key);
});
```

**默认配置**（[config/http.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/config/http.php)）：
- `client_period = 1` 分钟
- `client = 256` 次

**限流 key 策略**：已登录用户优先使用 `user.uuid`，未登录使用 IP。避免用户通过换 IP 绕过限流。

### 4.2 资源级限流（仅 WebSocket）

**✅ Panel 可证实**：[Enum/ResourceLimit.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Enum/ResourceLimit.php)

```php
enum ResourceLimit {
    case Websocket;
    // ...

    public function limit(): Limit {
        return match($this) {
            self::Backup     => Limit::perMinutes(15, 3),
            self::Websocket  => Limit::perMinute(5),  // ✅ WebSocket 握手：5 次/分钟/服务器
            // ...
        };
    }

    public static function boot(): void {
        foreach (self::cases() as $case) {
            RateLimiter::for($case->throttleKey(), function (Request $request) use ($case) {
                $server = $request->route()->parameter('server');
                return $case->limit()->by($server->uuid);  // ✅ 按 server_uuid 限流，非按用户
            });
        }
    }
}
```

**关键事实**：
- `/power` 和 `/command` **没有**资源级限流，仅依赖全局 256/min。
- WebSocket 握手按 `server_uuid` 限流 5/min，即同一台服务器所有用户共享配额（100 个标签页连接同一个服务器，共用 5 次/分钟）。

### 4.3 前端 UI 防重

**✅ Panel 可证实**：[PowerButtons.tsx#L52-L69](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/resources/scripts/components/server/console/PowerButtons.tsx#L52-L69)

```tsx
<Button.Success
    disabled={status !== 'offline'}   // Start 仅在 offline 时可用
    onClick={onButtonClick.bind(this, 'start')}
>Start</Button.Success>

<Button.Text
    disabled={!status}                // Restart 需要有已知状态
    onClick={onButtonClick.bind(this, 'restart')}
>Restart</Button.Text>

<Button.Danger
    disabled={status === 'offline'}   // Stop/Kill 仅在非 offline 时可用
    onClick={onButtonClick.bind(this, killable ? 'kill' : 'stop')}
>{killable ? 'Kill' : 'Stop'}</Button.Danger>
```

---

## 5. 白名单与语义鉴别（Panel 可证实）

### 5.1 电源动作白名单

**✅ Panel 可证实**：[SendPowerRequest.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Http/Requests/Api/Client/Servers/SendPowerRequest.php)

```php
class SendPowerRequest extends ClientApiRequest
{
    public function permission(): string {
        switch ($this->input('signal')) {
            case 'start':   return Permission::ACTION_CONTROL_START;    // control.start
            case 'stop':
            case 'kill':    return Permission::ACTION_CONTROL_STOP;     // control.stop
            case 'restart': return Permission::ACTION_CONTROL_RESTART;  // control.restart
        }
        return '__invalid';  // ✅ 未知 signal：授权层会拒绝
    }

    public function rules(): array {
        return [
            // ✅ 白名单：仅允许这 4 个值
            'signal' => 'required|string|in:start,stop,restart,kill',
        ];
    }
}
```

**两层保护**：
1. Laravel Validation `in:` — 非法值直接 422
2. `permission()` 返回 `__invalid` — 即使绕过验证，Gate 也查不到该权限 → 403

### 5.2 控制台命令白名单

**✅ Panel 可证实**：[SendCommandRequest.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Http/Requests/Api/Client/Servers/SendCommandRequest.php)

```php
class SendCommandRequest extends ClientApiRequest
{
    public function permission(): string {
        return Permission::ACTION_CONTROL_CONSOLE;  // control.console
    }

    public function rules(): array {
        return [
            'command' => 'required|string|min:1',  // ✅ 无命令内容白名单
        ];
    }
}
```

**关键事实**：
- Panel **不做命令内容过滤**。原因：游戏服命令集差异极大（Minecraft `/op`、Source `sm_kick`、Rust `server.save` 等），Panel 无法穷举。
- 权限门槛：仅需拥有 `control.console` 权限。
- **命令过滤职责在 Wings**（⚠️ 推断：Wings Egg 的 `config.yml` 中通常配置了 disallow 黑名单）。

### 5.3 调度任务白名单

**✅ Panel 可证实**：[StoreTaskRequest.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Http/Requests/Api/Client/Servers/Schedules/StoreTaskRequest.php)

```php
public function rules(): array {
    return [
        // ✅ action 白名单
        'action'              => 'required|in:command,power,backup',
        'payload'             => 'required_unless:action,backup|string|nullable',
        'time_offset'         => 'required|numeric|min:0|max:900',  // 最大延迟 15 分钟
        'continue_on_failure' => 'sometimes|required|boolean',
    ];
}
```

> ⚠️ 注意：`action=power` 时的 `payload`（start/stop/restart/kill）**仅在 Wings 侧校验**，Panel 的 StoreTaskRequest 不二次校验 payload 枚举。执行时才在 [RunTaskJob.php#L62-L63](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Jobs/Schedule/RunTaskJob.php#L62-L63) 交给 `DaemonPowerRepository::send()` 发出，由 Wings 验证。

---

## 6. 权限鉴权模型（Panel 可证实）

### 6.1 请求层权限解析

**✅ Panel 可证实**：[ClientApiRequest.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Http/Requests/Api/Client/ClientApiRequest.php)

```php
class ClientApiRequest extends ApplicationApiRequest
{
    public function authorize(): bool {
        if ($this instanceof ClientPermissionsRequest || method_exists($this, 'permission')) {
            $server = $this->route()->parameter('server');
            if ($server instanceof Server) {
                return $this->user()->can($this->permission(), $server);  // → ServerPolicy
            }
            return false;
        }
        return true;
    }
}
```

### 6.2 权限判定策略（ServerPolicy）

**✅ Panel 可证实**：[ServerPolicy.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Policies/ServerPolicy.php)

```php
class ServerPolicy
{
    public function before(User $user, string $ability, Server $server): ?bool {
        // ✅ 管理员 / 服务器所有者：无条件放行
        if ($user->root_admin || $server->owner_id === $user->id) {
            return true;
        }
        return $this->checkPermission($user, $server, $ability);
    }

    protected function checkPermission(User $user, Server $server, string $permission): bool {
        $subuser = $server->subusers->where('user_id', $user->id)->first();
        if (!$subuser || empty($permission)) return false;
        // ✅ 子用户权限：存储为 JSON/text 数组 ['control.start', ...]
        return in_array($permission, $subuser->permissions);
    }
}
```

**权限常量汇总**（[Permission.php#L18-L22](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Models/Permission.php#L18-L22)）：

| 常量 | 值 | 对应动作 |
|------|-----|---------|
| `ACTION_WEBSOCKET_CONNECT` | `websocket.connect` | 建立 WebSocket |
| `ACTION_CONTROL_CONSOLE` | `control.console` | 发送命令 |
| `ACTION_CONTROL_START` | `control.start` | Start |
| `ACTION_CONTROL_STOP` | `control.stop` | Stop / Kill |
| `ACTION_CONTROL_RESTART` | `control.restart` | Restart |

### 6.3 服务器状态门禁（AuthenticateServerAccess）

**✅ Panel 可证实**：[AuthenticateServerAccess.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Http/Middleware/Api/Client/Server/AuthenticateServerAccess.php)

```php
public function handle(Request $request, \Closure $next): mixed
{
    // 1. 归属校验：非 owner/admin/subuser → 404（隐私保护，不让探测）
    if ($user->id !== $server->owner_id && !$user->root_admin) {
        if (!$server->subusers->contains('user_id', $user->id)) {
            throw new NotFoundHttpException();
        }
    }

    // 2. 状态冲突校验
    try {
        $server->validateCurrentState();
    } catch (ServerStateConflictException $exception) {
        // 例外：允许查看 server 基本信息 / 资源使用情况
        if (!$request->routeIs('api:client:server.view')
            && !$request->routeIs('api:client:server.resources')) {
            // WebSocket 也例外：管理员可连以查看安装/迁移日志
            if (!$user->root_admin || !$request->routeIs($this->except)) {
                throw $exception;  // HTTP 409 Conflict
            }
        }
    }
    return $next($request);
}
```

**`validateCurrentState()`**（[Server.php#L390-L401](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Models/Server.php#L390-L401)）：

```php
public function validateCurrentState()
{
    if (
        $this->isSuspended()                         // status = 'suspended'
        || $this->node->isUnderMaintenance()         // node.maintenance_mode
        || !$this->isInstalled()                     // status = installing/install_failed
        || $this->status === self::STATUS_RESTORING_BACKUP
        || !is_null($this->transfer)                 // 跨节点迁移中
    ) {
        throw new ServerStateConflictException($this);  // 409
    }
}
```

**结论**：
- 服务器处于 `suspended / 维护中 / 安装中 / 还原备份 / 迁移中` 时，`/power` 和 `/command` **均被拒绝（409）**。
- 唯一例外：管理员的 WebSocket 连接，用于查看安装/迁移日志。

---

## 7. Panel → Wings：REST HTTP 调用链（Panel 可证实）

### 7.1 DaemonRepository 基类

**✅ Panel 可证实**：[DaemonRepository.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Repositories/Wings/DaemonRepository.php)

```php
abstract class DaemonRepository
{
    public function getHttpClient(array $headers = []): Client {
        Assert::isInstanceOf($this->node, Node::class);

        return new Client([
            'verify'          => $this->app->environment('production'),
            'base_uri'        => $this->node->getConnectionAddress(),
            'timeout'         => config('pterodactyl.guzzle.timeout'),         // 默认 15s
            'connect_timeout' => config('pterodactyl.guzzle.connect_timeout'), // 默认 5s
            'headers' => array_merge($headers, [
                // ✅ 使用 Node 的加密密钥（AES-256-CBC 存储）作为 Bearer Token
                'Authorization' => 'Bearer ' . $this->node->getDecryptedKey(),
                'Accept'        => 'application/json',
                'Content-Type'  => 'application/json',
            ]),
        ]);
    }
}
```

**超时配置**（[config/pterodactyl.php#L79-L82](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/config/pterodactyl.php#L79-L82)）：
- `GUZZLE_TIMEOUT = 15` 秒（总请求）
- `GUZZLE_CONNECT_TIMEOUT = 5` 秒（TCP 连接）

### 7.2 DaemonPowerRepository：电源动作下发

**✅ Panel 可证实**：[DaemonPowerRepository.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Repositories/Wings/DaemonPowerRepository.php)

```php
public function send(string $action): ResponseInterface
{
    Assert::isInstanceOf($this->server, Server::class);

    try {
        return $this->getHttpClient()->post(
            sprintf('/api/servers/%s/power', $this->server->uuid),
            ['json' => ['action' => $action]]  // ✅ body: {"action": "start"|"stop"|"restart"|"kill"}
        );
    } catch (TransferException $exception) {
        throw new DaemonConnectionException($exception);
    }
}
```

对应 Wings Endpoint：`POST /api/servers/{uuid}/power`

### 7.3 DaemonCommandRepository：控制台命令下发

**✅ Panel 可证实**：[DaemonCommandRepository.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Repositories/Wings/DaemonCommandRepository.php)

```php
public function send(array|string $command): ResponseInterface
{
    try {
        return $this->getHttpClient()->post(
            sprintf('/api/servers/%s/commands', $this->server->uuid),
            [
                // ✅ 总是包装为数组
                'json' => ['commands' => is_array($command) ? $command : [$command]],
            ]
        );
    } catch (TransferException $exception) {
        throw new DaemonConnectionException($exception);
    }
}
```

对应 Wings Endpoint：`POST /api/servers/{uuid}/commands`

### 7.4 Activity 审计日志埋点

**✅ Panel 可证实**：

- 电源动作日志（[PowerController.php#L31](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Http/Controllers/Api/Client/Servers/PowerController.php#L31)）：
  ```php
  Activity::event(strtolower("server:power.{$request->input('signal')}"))->log();
  // 事件名: server:power.start / .stop / .restart / .kill
  ```

- 命令日志（[CommandController.php#L46](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Http/Controllers/Api/Client/Servers/CommandController.php#L46)）：
  ```php
  Activity::event('server:console.command')
      ->property('command', $request->input('command'))  // ✅ 记录具体命令内容
      ->log();
  ```

**写入时机**：Activity 记录在 **Wings 调用成功之后**（try 块外部）。如果 Wings 调用失败抛异常，Activity 不会被写入，避免假阳性。

---

## 8. Panel → Wings：WebSocket 通道（Panel 可证实）

### 8.1 JWT 签发服务（NodeJWTService）

**✅ Panel 可证实**：[NodeJWTService.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Services/Nodes/NodeJWTService.php)

由 [WebsocketController.php#L55-L62](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Http/Controllers/Api/Client/Servers/WebsocketController.php#L55-L62) 调用：

```php
$token = $this->jwtService
    ->setExpiresAt(CarbonImmutable::now()->addMinutes(10))   // ✅ 10 分钟过期
    ->setUser($request->user())
    ->setClaims([
        'server_uuid'  => $server->uuid,
        'permissions'  => $permissions,   // ✅ 用户在该服务器的权限列表嵌入 JWT
    ])
    ->handle($node, $user->id . $server->uuid);
```

**JWT 结构（Panel 代码证实）**：

```
Header:
  alg: HS256               // 对称加密，用 Node 的密钥签名
  jti: md5(user_id + uuid) // 用于 deny-list（踢人/注销）

Payload:
  iss: APP_URL             // 签发者
  aud: node 连接地址       // 接收方
  iat: 签发时间
  nbf: iat - 5min          // 5 分钟时钟偏差
  exp: iat + 10min         // ✅ 过期时间 10 分钟
  sub: 用户标识
  user_uuid: 用户 UUID
  user_id: 用户 ID (兼容)
  server_uuid: 服务器 UUID
  permissions: [ "websocket.connect", "control.start", ... ]  // ✅ Wings 自行鉴权
  unique_id: Str::random() // 防重放
```

**设计含义**：Wings 收到 WebSocket 事件时，检查 JWT 中的 `permissions` 数组即可决定放行与否，**无需回查 Panel**，保证低延迟。

### 8.2 WebSocket 事件枚举

**✅ Panel 可证实**：[events.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/resources/scripts/components/server/events.ts)

```ts
export enum SocketRequest {  // 前端 → Wings
    SEND_LOGS = 'send logs',    // 拉取历史控制台缓冲
    SEND_STATS = 'send stats',  // 请求立即发一次统计
    SET_STATE = 'set state',    // ✅ 电源动作
}

export enum SocketEvent {     // Wings → 前端
    DAEMON_MESSAGE = 'daemon message',
    DAEMON_ERROR = 'daemon error',
    CONSOLE_OUTPUT = 'console output',
    STATUS = 'status',           // ✅ 电源状态变化
    STATS = 'stats',
    // 安装 / 迁移等
}
```

> 注意：`"send command"` 事件未在枚举中定义，但 [Console.tsx#L121](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/resources/scripts/components/server/console/Console.tsx#L121) 直接用字符串字面量 `instance.send('send command', command)` 发送。

---

## 9. Wings 侧：接收约定与行为（⚠️ 推断）

> **⚠️ 重要**：本仓库无 Wings 代码。本节内容仅依据 Panel 的调用协议（Endpoint 路径、请求体结构、JWT Claims、前端事件枚举等）**推断**，不保证与实际 Wings 实现完全一致。

### 9.1 REST Endpoint 约定

由 Panel 的 Repository 代码可推断 Wings 暴露以下接口：

| Method | Path | Body（Panel 发送） | Panel 预期行为 |
|--------|------|-------------------|--------------|
| POST | `/api/servers/{uuid}/power` | `{"action":"start\|stop\|restart\|kill"}` | 204 No Content 立即返回 |
| POST | `/api/servers/{uuid}/commands` | `{"commands":["cmd1",...]}` | 服务器必须 running，否则 502（Panel 有特判） |
| GET | `/api/servers/{uuid}` | — | 返回 `{"state": "..." }` 数组，含 `state` 字段（调度 `only_when_online` 用）|

### 9.2 WebSocket Event 约定

由 Panel 前端代码可推断 Wings 支持以下 WebSocket 消息：

**Client → Wings**（Panel 前端发送的事件）：

| Event | Args | JWT 所需权限（推断） |
|-------|------|---------------------|
| `auth` | `[jwt]` | — |
| `send logs` | `[]` | `websocket.connect` |
| `send stats` | `[]` | `websocket.connect` |
| `set state` | `["start"\|"stop"\|"restart"\|"kill"]` | 对应 `control.start\|stop\|restart` |
| `send command` | `["/op Notch"]` | `control.console` |

**Wings → Client**（Panel 前端监听的事件）：

| Event | 触发时机（推断） |
|-------|----------------|
| `auth success` | JWT 校验通过 |
| `token expiring` | JWT 距过期 < 3min |
| `token expired` | JWT 已过期 |
| `jwt error` | JWT 校验失败 |
| `status` | 电源状态迁移（`"starting"\|"running"\|"stopping"\|"offline"`） |
| `console output` | 游戏服 stdout 有新行 |
| `stats` | CPU/内存/磁盘/网络统计心跳 |

### 9.3 并发安全与内部实现（⚠️ 推断，无本地代码）

以下内容**完全是基于架构常识的推断**，Panel 代码中无任何对应实现：

- **电源动作并发控制**：推断 Wings 内部对每个 Server Instance 有互斥锁（如 Go `sync.Mutex`），避免 Start 与 Kill 同时到达导致进程孤儿
- **Stop 流程**：按 Egg 配置的 `stop_command`（如 `/stop`）写入 stdin → 等待 `stop_timeout`（默认 30s）→ 超时再 SIGTERM → 再超时 SIGKILL
- **Kill 流程**：直接 SIGKILL 容器进程组
- **命令限频**：推断 Wings 内有令牌桶限频器（如 60 条/分钟/实例），防止命令刷屏
- **控制台环形缓冲**：推断 Wings 维护一个最近 2MB 的控制台输出 ring buffer，用于 `send logs` 历史拉取

> 以上推断如有疑问，需查阅 Pterodactyl Wings 仓库（Go 代码）确认。

---

## 10. 调度系统：计划任务中的电源/命令（Panel 可证实）

### 10.1 ProcessScheduleService：事务入队 + only_when_online

**✅ Panel 可证实**：[ProcessScheduleService.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Services/Schedules/ProcessScheduleService.php)

```php
public function handle(Schedule $schedule, bool $now = false): void
{
    $task = $schedule->tasks()->orderBy('sequence_id')->first();

    // ✅ 数据库事务：标记 is_processing + is_queued
    $this->connection->transaction(function () use ($schedule, $task) {
        $schedule->forceFill([
            'is_processing' => true,
            'next_run_at'   => $schedule->getNextRunDate(),
        ])->saveOrFail();
        $task->update(['is_queued' => true]);
    });

    $job = new RunTaskJob($task, $now);
    if ($schedule->only_when_online) {
        // ✅ 调 Wings GET /api/servers/{uuid} 获取 state
        try {
            $details = $this->serverRepository->setServer($schedule->server)->getDetails();
            $state = $details['state'] ?? 'offline';
            // ✅ 只对 offline/stopping 做跳过
            if (in_array($state, ['offline', 'stopping'])) {
                $job->failed();   // 默默标记完成，不抛异常
                return;
            }
        } catch (\Exception $exception) {
            if (!$exception instanceof DaemonConnectionException) {
                $job->failed($exception);
            }
            $job->failed();
            return;
        }
    }

    // ✅ 延迟 time_offset 秒后执行
    if (!$now) {
        $this->dispatcher->dispatch($job->delay($task->time_offset));
    } else {
        try { $this->dispatcher->dispatchNow($job); }
        catch (\Exception $e) { $job->failed($e); throw $e; }
    }
}
```

### 10.2 RunTaskJob：按序执行与失败降级

**✅ Panel 可证实**：[RunTaskJob.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Jobs/Schedule/RunTaskJob.php)

```php
public function handle(DaemonCommandRepository $commandRepository,
                       InitiateBackupService $backupService,
                       DaemonPowerRepository $powerRepository)
{
    // ✅ 防御性检查：服务器状态不是 null（即 suspended/installing 等）
    if (!is_null($server->status)) { $this->failed(); return; }

    try {
        switch ($this->task->action) {
            case Task::ACTION_POWER:   // "power"
                $powerRepository->setServer($server)->send($this->task->payload); break;
            case Task::ACTION_COMMAND: // "command"
                $commandRepository->setServer($server)->send($this->task->payload); break;
            case Task::ACTION_BACKUP:  // "backup"
                $backupService->...; break;
        }
    } catch (\Exception $exception) {
        // ✅ 唯一降级路径：continue_on_failure + DaemonConnectionException
        if (!($this->task->continue_on_failure && $exception instanceof DaemonConnectionException)) {
            throw $exception;   // 其他异常：任务链中断
        }
    }

    $this->markTaskNotQueued();
    $this->queueNextTask();  // ✅ 取下一个 sequence_id，delay 后入队
}
```

**任务链执行模型**：

```
Schedule (is_processing=true)
  ├─ Task #1 (sequence_id=1, time_offset=0)
  │    ├─ success → markTaskNotQueued → dispatch Task #2
  │    ├─ DaemonConnectionException + continue_on_failure=true → dispatch Task #2
  │    └─ 其他异常 → failed()，中断链，释放 is_processing
  ├─ Task #2 (sequence_id=2, time_offset=30)
  ...
  └─ 最后一个 Task → markScheduleComplete (is_processing=false, last_run_at=now)
```

---

## 11. 失败处理与状态回滚（Panel 可证实）

### 11.1 DaemonConnectionException：异常封装

**✅ Panel 可证实**：[DaemonConnectionException.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Exceptions/Http/Connection/DaemonConnectionException.php)

```php
public function __construct(GuzzleException $previous, bool $useStatusCode = true)
{
    $response = method_exists($previous, 'getResponse') ? $previous->getResponse() : null;
    $this->requestId = $response?->getHeaderLine('X-Request-Id'); // ✅ 关联 Wings 日志

    if ($useStatusCode) {
        $this->statusCode = is_null($response) ? 504 : $response->getStatusCode();
        if ($this->statusCode < 400) $this->statusCode = 502; // 2XX 却异常 → 升级为 502
    }

    // ✅ 日志分级：5XX（除 504）= ERROR，其余 = WARNING
    $level = $this->statusCode >= 500 && $this->statusCode !== 504
        ? DisplayException::LEVEL_ERROR
        : DisplayException::LEVEL_WARNING;
}
```

**`report()` 方法**：自动写 Laravel Log，含 `request_id`，便于在 Wings 侧用同一 X-Request-Id 关联排查。

### 11.2 CommandController：502 → 用户可读错误

**✅ Panel 可证实**：[CommandController.php#L32-L44](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Http/Controllers/Api/Client/Servers/CommandController.php#L32-L44)

```php
try {
    $this->repository->setServer($server)->send($request->input('command'));
} catch (DaemonConnectionException $exception) {
    $previous = $exception->getPrevious();
    if ($previous instanceof BadResponseException) {
        // ✅ Wings 返回 502 → 判断为游戏服未运行
        if ($previous->getResponse()->getStatusCode() === Response::HTTP_BAD_GATEWAY) {
            throw new HttpException(
                Response::HTTP_BAD_GATEWAY,
                'Server must be online in order to send commands.',  // ✅ 用户可读
                $exception
            );
        }
    }
    throw $exception;
}
```

### 11.3 电源动作失败：无回滚，乐观状态模型

**✅ Panel 可证实**：[PowerController.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Http/Controllers/Api/Client/Servers/PowerController.php)

`DaemonPowerRepository::send()` 抛出的 `DaemonConnectionException` **直接冒泡**，无 catch 处理。

**为什么不需要"回滚状态"**（Panel 代码证实的核心设计）：
1. Panel 数据库 `Server.status` 字段**不存** running/offline/starting/stopping（只存 installing/suspended/restoring_backup 等元状态）
2. 电源运行状态完全由 Wings WebSocket 的 `status` 事件驱动前端 store
3. 即使 Panel 发 Start 请求失败，只要服务器实际还是 offline，WebSocket 会把 `status=offline` 推给前端，Start 按钮重新可用
4. **最终一致，而非强一致**

### 11.4 调度任务失败：continue_on_failure 开关

**✅ Panel 可证实**（汇总自上文代码）：

| 场景 | 失败行为 |
|------|---------|
| `DaemonConnectionException` + `continue_on_failure=true` | 记录日志，**继续执行后续 Task** |
| `DaemonConnectionException` + `continue_on_failure=false` | 抛异常 → Job 失败 → 调度链中断 |
| 任何其他 Exception | 抛异常 → 调度链中断 |
| `only_when_online=true` + 服务器 offline | 默默标记完成，不抛异常 |
| `server.status !== null`（suspended/installing 等）| Job 直接 failed() |

**中断后的清理**：`RunTaskJob::failed()`（[L89-L93](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Jobs/Schedule/RunTaskJob.php#L89-L93)）会调用：
- `markTaskNotQueued()` — 释放 task 锁
- `markScheduleComplete()` — 释放 `schedules.is_processing` 锁，避免 Schedule 永久卡死

### 11.5 Panel 数据库：乐观设计，无状态回滚

整个电源/命令链路中，Panel 数据库只有以下写入（均为"事后记录"或"锁标记"）：

| 写入点 | 数据 | 失败时行为 |
|--------|------|-----------|
| PowerController | Activity Log（`server:power.*`） | ✅ 异常未捕获则不写入 |
| CommandController | Activity Log（`server:console.command`，含命令内容） | ✅ 异常未捕获则不写入 |
| `ProcessScheduleService` | `schedules.is_processing`, `tasks.is_queued` | ✅ 在 DB 事务中，异常自动回滚 |
| `RunTaskJob::failed()` | `is_processing=false`, `last_run_at=now` | 非事务，保证最终释放锁 |

**无运行状态回滚的理由**：Panel 本质上是**控制平面**，游戏服实际运行状态在 Wings/容器中。Panel 只发指令 + 记日志，不维护权威状态副本。

---

## 12. 客服视角：Power Action 语义对照表

> 针对客服把按钮当命令行的场景，整理"按钮按下 → 代码路径 → 用户体感"：

| 按钮 | 前端发送值 | Panel 路径 | Wings Endpoint（推断） | 用户体感 | 风险 |
|------|-----------|-----------|---------------------|---------|------|
| **Start** | `"start"` | `POST /power` → `DaemonPowerRepository` | `POST /api/servers/{uuid}/power {"action":"start"}` | 服务器启动，数秒到数分钟后可进入 | 正常操作，风险极低 |
| **Stop** | `"stop"` | 同上 | 同上 | 玩家收到关服提示，地图存档 | **优雅关闭**，应默认使用 |
| **Restart** | `"restart"` | 同上 | 同上 | 玩家被踢出 → 等待重连 | 可能触发玩家被踢 |
| **Kill** | `"kill"` | 同上 | 同上 | 瞬间掉线，无存档过程 | ⚠️ 可能损坏地图/世界，仅在 Stop 卡死时使用 |

**客服操作原则**：
1. 日常关服一律 **Stop**，不用 Kill
2. Stop 超过 2 分钟控制台无任何响应，才升级到 Kill
3. Kill 后建议检查最新存档时间戳，再执行 Start
4. **按钮 ≠ 命令行**：游戏内命令（`/op`, `whitelist add` 等）必须使用 Console 下方的命令输入框

---

## 13. 关键文件索引（Panel 代码）

### 前端 (React/TypeScript)

| 文件 | 职责 |
|------|------|
| [PowerButtons.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/resources/scripts/components/server/console/PowerButtons.tsx) | 电源按钮 UI + WebSocket "set state" 发送 |
| [Console.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/resources/scripts/components/server/console/Console.tsx) | xterm 终端 + WebSocket "send command" + 历史记录 |
| [WebsocketHandler.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/resources/scripts/components/server/WebsocketHandler.tsx) | WebSocket 生命周期 + JWT 自动刷新 + `status` 事件接收 |
| [Websocket.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/resources/scripts/plugins/Websocket.ts) | Sockette 封装 + 协议序列化 |
| [Can.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/resources/scripts/components/elements/Can.tsx) | 权限条件渲染组件 |
| [events.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/resources/scripts/components/server/events.ts) | WebSocket 事件枚举 |
| [state/server/index.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/resources/scripts/state/server/index.ts) | 前端状态定义（ServerStatus = `'offline'\|'starting'\|'stopping'\|'running'\|null`）|
| [getWebsocketToken.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/resources/scripts/api/server/getWebsocketToken.ts) | REST 请求 WebSocket JWT |
| [http.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/resources/scripts/api/http.ts) | Axios 实例（withCredentials=true, 20s 超时） |

### Panel 后端 (PHP Laravel)

#### 路由 & 中间件

| 文件 | 职责 |
|------|------|
| [routes/api-client.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/routes/api-client.php) | `/api/client/servers/{server}/*` 路由（/command → CommandController） |
| [Http/Kernel.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Http/Kernel.php) | 中间件栈（client-api / api / throttle） |
| [Providers/RouteServiceProvider.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Providers/RouteServiceProvider.php) | 路由挂载 + 全局限流器（256/min）+ ResourceLimit::boot() |
| [Enum/ResourceLimit.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Enum/ResourceLimit.php) | 资源级限流（Websocket 5/min/server） |
| [Middleware/.../AuthenticateServerAccess.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Http/Middleware/Api/Client/Server/AuthenticateServerAccess.php) | 归属校验 + 状态冲突门禁（409） |

#### 控制器

| 文件 | 职责 |
|------|------|
| [PowerController.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Http/Controllers/Api/Client/Servers/PowerController.php) | `POST /power` → DaemonPowerRepository + Activity |
| [CommandController.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Http/Controllers/Api/Client/Servers/CommandController.php) | `POST /command` → DaemonCommandRepository + 502 转用户可读 + Activity |
| [WebsocketController.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Http/Controllers/Api/Client/Servers/WebsocketController.php) | `GET /websocket` → 签发 10min JWT（含 permissions） |

#### 请求验证（白名单）

| 文件 | 职责 |
|------|------|
| [SendPowerRequest.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Http/Requests/Api/Client/Servers/SendPowerRequest.php) | `signal in:start,stop,restart,kill` + 权限映射 |
| [SendCommandRequest.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Http/Requests/Api/Client/Servers/SendCommandRequest.php) | `command required\|min:1` + `control.console` 权限（无内容白名单）|
| [ClientApiRequest.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Http/Requests/Api/Client/ClientApiRequest.php) | `authorize()` → `user()->can(permission(), server)` |
| [StoreTaskRequest.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Http/Requests/Api/Client/Servers/Schedules/StoreTaskRequest.php) | `action in:command,power,backup` + `time_offset max:900` |

#### Wings 通信层

| 文件 | 职责 |
|------|------|
| [DaemonRepository.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Repositories/Wings/DaemonRepository.php) | Guzzle 客户端 + Node Bearer Token + 超时配置（15s/5s）|
| [DaemonPowerRepository.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Repositories/Wings/DaemonPowerRepository.php) | `POST /api/servers/{uuid}/power` body `{"action":...}` |
| [DaemonCommandRepository.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Repositories/Wings/DaemonCommandRepository.php) | `POST /api/servers/{uuid}/commands` body `{"commands":[...]}` |
| [DaemonServerRepository.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Repositories/Wings/DaemonServerRepository.php) | `GET /api/servers/{uuid}`（调度 only_when_online 用）|
| [DaemonConnectionException.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Exceptions/Http/Connection/DaemonConnectionException.php) | Wings 异常封装 + X-Request-Id 关联 + 日志分级 |

#### 权限 & 模型

| 文件 | 职责 |
|------|------|
| [Permission.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Models/Permission.php) | 权限常量（control.* / websocket.connect）|
| [Policies/ServerPolicy.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Policies/ServerPolicy.php) | Gate 判定：管理员/所有者放行，子用户查 permissions 数组 |
| [Models/Server.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Models/Server.php) | `validateCurrentState()`（suspended/维护/安装/还原/迁移 → 409），`status` 仅存元状态 |
| [Services/Nodes/NodeJWTService.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Services/Nodes/NodeJWTService.php) | HS256 JWT 签发，嵌入 permissions + server_uuid |

#### 调度系统

| 文件 | 职责 |
|------|------|
| [Services/Schedules/ProcessScheduleService.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Services/Schedules/ProcessScheduleService.php) | 事务入队 + only_when_online 检查 + 延迟分发 |
| [Jobs/Schedule/RunTaskJob.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Jobs/Schedule/RunTaskJob.php) | 按 sequence_id 顺序执行 + continue_on_failure 降级 + failed() 释放锁 |
| [Models/Task.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Models/Task.php) | Task::ACTION_POWER/COMMAND/BACKUP 常量 |

#### 配置

| 文件 | 职责 |
|------|------|
| [config/http.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/config/http.php) | API 限流阈值（client 256/min） |
| [config/pterodactyl.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/config/pterodactyl.php) | Guzzle 超时（15s/5s） |

---

## 14. 附：客服 FAQ 速查

**Q1: 点 Stop 按钮等了好久都没反应怎么办？**
A: Stop 是优雅关闭，先让游戏服自己存档。正常 10~30 秒。超过 2 分钟控制台无任何输出，说明游戏服进程卡死，才可升级到 **Kill** 按钮（会弹二次确认）。Kill 有未存档丢失风险。

**Q2: Start 按钮灰的点不了？**
A: 只有当 `state.status.value === 'offline'`（WebSocket 推送的前端状态）时 Start 才可用。检查 Console 页面左上角状态标签。

**Q3: 命令输入框输 "stop" 回车 和点 Stop 按钮一样吗？**
A: **完全不同！**
- 命令输入框发的是**游戏服控制台命令**（Minecraft 里要输 `/stop` 才是关服，直接 `stop` 在大多数游戏里不是合法指令）。
- Stop **按钮**走的是电源动作，由 Wings 负责完整优雅关闭流程，**推荐用按钮**。

**Q4: Kill 和 Stop 到底有啥区别？**
A: Stop = 给游戏服机会存档 + 正常退出（安全）；Kill = 直接杀死进程（等同拔电源）。Kill 只应在 Stop 卡死时用。用完建议检查存档后再 Start。

**Q5: 服务器被暂停（suspended）了，能发命令吗？**
A: Panel 后端 `AuthenticateServerAccess` 中间件会拒绝 `/power` 和 `/command` 请求（HTTP 409 Conflict）。即使 WebSocket 连上也只能看日志。先解除暂停。

**Q6: 命令输入框没了？**
A: 检查账号有没有 `control.console` 子用户权限（由所有者在 Users 标签页分配）。无权限时输入框不渲染，Wings 侧 JWT 权限校验也会拦截。
