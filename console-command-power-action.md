# Pterodactyl Panel 控制台命令与电源动作 — 代码走向全链路分析

## 0. 目录

- [1. 整体架构概览](#1-整体架构概览)
- [2. 前端：两条独立下发通道](#2-前端两条独立下发通道)
  - [2.1 电源按钮（WebSocket 通道）](#21-电源按钮websocket-通道)
  - [2.2 控制台命令（WebSocket 通道）](#22-控制台命令websocket-通道)
  - [2.3 WebSocket Token 获取与刷新（REST → WebSocket 桥）](#23-websocket-token-获取与刷新rest--websocket-桥)
  - [2.4 REST API 通道（供外部集成 / 调度系统）](#24-rest-api-通道供外部集成--调度系统)
- [3. Panel 后端：路由注册与中间件管道](#3-panel-后端路由注册与中间件管道)
  - [3.1 路由入口](#31-路由入口)
  - [3.2 全局节流（Rate Limiting）](#32-全局节流rate-limiting)
  - [3.3 资源级节流（ResourceLimit Enum）](#33-资源级节流resourcelimit-enum)
- [4. Panel 后端：白名单与语义鉴别](#4-panel-后端白名单与语义鉴别)
  - [4.1 电源动作白名单](#41-电源动作白名单)
  - [4.2 控制台命令白名单](#42-控制台命令白名单)
  - [4.3 调度任务白名单](#43-调度任务白名单)
- [5. Panel 后端：权限鉴权模型](#5-panel-后端权限鉴权模型)
  - [5.1 请求层权限解析（ClientApiRequest::authorize）](#51-请求层权限解析clientapirequestauthorize)
  - [5.2 权限判定策略（ServerPolicy）](#52-权限判定策略serverpolicy)
  - [5.3 服务器状态门禁（AuthenticateServerAccess）](#53-服务器状态门禁authenticateserveraccess)
- [6. Panel → Wings：HTTP 调用链](#6-panel--wingshttp-调用链)
  - [6.1 DaemonRepository 基类：Guzzle HTTP 客户端](#61-daemonrepository-基类guzzle-http-客户端)
  - [6.2 DaemonPowerRepository：电源动作下发](#62-daemonpowerrepository电源动作下发)
  - [6.3 DaemonCommandRepository：控制台命令下发](#63-daemoncommandrepository控制台命令下发)
  - [6.4 Activity 审计日志埋点](#64-activity-审计日志埋点)
- [7. Panel → Wings：WebSocket 通道（JWT 鉴权）](#7-panel--wingswebsocket-通道jwt-鉴权)
  - [7.1 JWT 签发服务（NodeJWTService）](#71-jwt-签发服务nodejwtservice)
  - [7.2 前端 WebSocket 封装（Websocket.ts + Sockette）](#72-前端-websocket-封装websockett--sockette)
  - [7.3 事件枚举：前端发什么、收什么](#73-事件枚举前端发什么收什么)
- [8. Wings 侧：接收约定与并发模型（推断自 Panel 调用协议）](#8-wings-侧接收约定与并发模型推断自-panel-调用协议)
  - [8.1 REST Endpoint 约定](#81-rest-endpoint-约定)
  - [8.2 WebSocket Event 约定](#82-websocket-event-约定)
  - [8.3 并发安全与节流（Panel 侧配合）](#83-并发安全与节流panel-侧配合)
- [9. 调度系统：计划任务中的电源/命令](#9-调度系统计划任务中的电源命令)
  - [9.1 ProcessScheduleService：事务入队](#91-processscheduleservice事务入队)
  - [9.2 RunTaskJob：按序执行与失败降级](#92-runtaskjob按序执行与失败降级)
- [10. 失败处理与状态回滚](#10-失败处理与状态回滚)
  - [10.1 DaemonConnectionException：Wings 通讯异常封装](#101-daemonconnectionexceptionwings-通讯异常封装)
  - [10.2 CommandController：502 → 用户可读错误](#102-commandcontroller502--用户可读错误)
  - [10.3 电源动作失败：无回滚，靠 WebSocket 状态事件最终一致](#103-电源动作失败无回滚靠-websocket-状态事件最终一致)
  - [10.4 调度任务失败：continue_on_failure 开关](#104-调度任务失败continue_on_failure-开关)
  - [10.5 Panel 数据库状态：乐观设计，无事务回滚](#105-panel-数据库状态乐观设计无事务回滚)
- [11. 客服视角：Power Action 语义对照表](#11-客服视角power-action-语义对照表)
- [12. 关键文件索引](#12-关键文件索引)

---

## 1. 整体架构概览

```
 ┌─────────────────────────────────────────────────────────────────────┐
 │                         浏览器前端 (React)                           │
 │  ┌──────────────────────┐     ┌──────────────────────────────┐      │
 │  │  PowerButtons.tsx    │     │  Console.tsx                 │      │
 │  │  (Start/Stop/        │     │  (命令输入框 + xterm.js)     │      │
 │  │   Restart/Kill)      │     │                              │      │
 │  └──────────┬───────────┘     └──────────────┬───────────────┘      │
 │             │ WebSocket                        │ WebSocket          │
 │             │ "set state"                      │ "send command"     │
 │             ▼                                  ▼                    │
 │  ┌──────────────────────────────────────────────────────────┐       │
 │  │            Websocket.ts (Sockette 封装)                  │       │
 │  │  - 鉴权：JWT (10min 过期, 自动刷新)                      │       │
 │  │  - 重连：最多 20 次, 指数退避                             │       │
 │  └───────────────────────┬──────────────────────────────────┘       │
 │                          │                                          │
 │          ┌───────────────┴───────────────┐                          │
 │          │ REST /api/client/servers/     │                          │
 │          │   /{uuid}/websocket           │                          │
 │          │   (获取 JWT + socket URL)     │                          │
 │          └───────────────┬───────────────┘                          │
 └──────────────────────────┼──────────────────────────────────────────┘
                            │ HTTPS (withCredentials: true)
 ┌──────────────────────────┼──────────────────────────────────────────┐
 │                     Panel (PHP Laravel)                             │
 │  ┌───────────────────────▼──────────────────────────────┐           │
 │  │  RouteServiceProvider → throttle:api.client         │           │
 │  │  (256 req/min per user)                              │           │
 │  └───────────────────────┬──────────────────────────────┘           │
 │          ┌────────────────┼─────────────────┐                       │
 │          ▼                ▼                 ▼                       │
 │  /power (POST)    /command (POST)   /websocket (GET)                │
 │  PowerController  CommandController  WebsocketController            │
 │          │                │                 │                       │
 │          └────────────────┴─────────────────┘                       │
 │                           │                                         │
 │          ┌────────────────▼──────────────────┐                      │
 │          │  DaemonPowerRepository             │                      │
 │          │  DaemonCommandRepository           │                      │
 │          │  (Guzzle + Node Bearer Token)      │                      │
 │          └────────────────┬──────────────────┘                      │
 └───────────────────────────┼──────────────────────────────────────────┘
                             │ HTTPS (Node 对称加密密钥鉴权)
 ┌───────────────────────────▼──────────────────────────────────────────┐
 │                          Wings (Go 守护进程)                          │
 │  ┌─────────────────────────────────────────────────────────────┐     │
 │  │  REST:   POST /api/servers/{uuid}/power                     │     │
 │  │          POST /api/servers/{uuid}/commands                  │     │
 │  │  WS:     /api/servers/{uuid}/ws                             │     │
 │  │          → "set state" | "send command" | "send logs" ...   │     │
 │  └─────────────────────────────┬───────────────────────────────┘     │
 │                                ▼                                     │
 │  ┌─────────────────────────────────────────────────────────────┐     │
 │  │  每个 Server: 1 个 Instance struct                           │     │
 │  │  - powerMux sync.Mutex（电源动作串行化）                     │     │
 │  │  - env.LimitedRateLimiter（命令限频）                        │     │
 │  │  - console 通道 chan []string（环形缓冲）                    │     │
 │  └─────────────────────────────┬───────────────────────────────┘     │
 │                                ▼                                     │
 │                      游戏服进程 (Docker/Systemd)                      │
 └──────────────────────────────────────────────────────────────────────┘
```

**关键设计原则**：
1. **双轨下发**：电源/命令既可以走 REST（同步确认），也可以走 WebSocket（实时双向）。前端默认 WebSocket；外部 API 调用者 / 调度系统走 REST。
2. **JWT 短时令牌**：WebSocket 通信使用 10 分钟过期的 JWT，由 Panel 签发，Wings 独立校验，无需回查 Panel。
3. **白名单优先**：`signal`/`action` 字段均使用 `in:...` 枚举验证，拒绝未知值。
4. **乐观状态模型**：Panel 不维护电源状态数据库字段，完全依赖 Wings 推送的 `status` 事件，失败时无需"回滚状态"。

---

## 2. 前端：两条独立下发通道

### 2.1 电源按钮（WebSocket 通道）

**组件位置**：[PowerButtons.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/resources/scripts/components/server/console/PowerButtons.tsx)

```tsx
// 第 17-31 行：按钮点击核心逻辑
const killable = status === 'stopping';

const onButtonClick = (action: PowerAction | 'kill-confirmed', e) => {
    e.preventDefault();
    if (action === 'kill') return setOpen(true);  // 二次确认弹窗

    if (instance) {
        setOpen(false);
        // 直接通过 WebSocket 发送 "set state" 事件
        instance.send('set state', action === 'kill-confirmed' ? 'kill' : action);
    }
};
```

**4 种 Power Action 语义**（前端类型定义见 [ServerConsoleContainer.tsx#L14](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/resources/scripts/components/server/console/ServerConsoleContainer.tsx#L14)）：

| Action | 触发条件 | 前端禁用条件 | 实际发送值 |
|--------|---------|------------|-----------|
| `start` | 点击 Start | `status !== 'offline'` | `"start"` |
| `restart` | 点击 Restart | `!status`（状态未知） | `"restart"` |
| `stop` | 点击 Stop（当 status≠`stopping`） | `status === 'offline'` | `"stop"` |
| `kill` | ① Stop 过程中点按钮（killable=true）② 或二次确认后 | `status === 'offline'` | `"kill"` |

**前端权限渲染控制**（[Can.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/resources/scripts/components/elements/Can.tsx)）：
- Start 按钮：`<Can action={'control.start'}>`（L51）
- Restart 按钮：`<Can action={'control.restart'}>`（L60）
- Stop/Kill 按钮：`<Can action={'control.stop'}>`（L65）

**Kill 二次确认**：使用 `Dialog.Confirm` 组件，文案：
> "Forcibly stopping a server can lead to data corruption."

### 2.2 控制台命令（WebSocket 通道）

**组件位置**：[Console.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/resources/scripts/components/server/console/Console.tsx)

```tsx
// 第 97-124 行：命令输入与发送
const handleCommandKeyDown = (e: React.KeyboardEvent<HTMLInputElement>) => {
    // 历史命令：↑/↓ 翻页，最多保存 32 条
    if (e.key === 'ArrowUp') { /* ... */ }
    if (e.key === 'ArrowDown') { /* ... */ }

    const command = e.currentTarget.value;
    if (e.key === 'Enter' && command.length > 0) {
        // 持久化到 localStorage，key: ${serverId}:command_history
        setHistory(prev => [command, ...prev!].slice(0, 32));

        // 通过 WebSocket 发送 "send command" 事件
        instance && instance.send('send command', command);
        e.currentTarget.value = '';
    }
};
```

**渲染权限控制**（L66, L211）：
```tsx
const [canSendCommands] = usePermissions(['control.console']);
// 仅当拥有 control.console 权限时才渲染命令输入框
{canSendCommands && <div className={...}><input ... onKeyDown={handleCommandKeyDown} /></div>}
```

**输入框禁用条件**（L218）：`disabled={!instance || !connected}` — WebSocket 未连接时无法发送。

### 2.3 WebSocket Token 获取与刷新（REST → WebSocket 桥）

**核心组件**：[WebsocketHandler.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/resources/scripts/components/server/WebsocketHandler.tsx)

连接流程：
```
1. getWebsocketToken(uuid)
   → GET /api/client/servers/{uuid}/websocket
   → 返回 { token: JWT(10min), socket: "wss://node:.../api/servers/{uuid}/ws" }

2. socket.setToken(token).connect(socket)

3. SOCKET_OPEN 事件触发 authenticate()
   → send("auth", token)

4. Wings 校验通过 → "auth success" → setConnectionState(true)

5. Wings 提前预警 → "token expiring"（剩 3 分钟时）
   → 重新调用 getWebsocketToken() → setToken(newToken, true)
```

**自动刷新关键代码**（[WebsocketHandler.tsx#L20-L30](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/resources/scripts/components/server/WebsocketHandler.tsx#L20-L30)）：
```ts
const updateToken = (uuid, socket) => {
    if (updatingToken) return;         // 防抖锁
    updatingToken = true;
    getWebsocketToken(uuid)
        .then(data => socket.setToken(data.token, true));
};
```

**重连策略**：[Websocket.ts#L22-L55](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/resources/scripts/plugins/Websocket.ts#L22-L55) 使用 Sockette 库：
- `timeout: 1000ms`
- `maxAttempts: 20`（最多重连 20 次）
- 收到 Wings 关闭码 4400/4409（服务器挂起/暂停）时停止重连。

### 2.4 REST API 通道（供外部集成 / 调度系统）

前端虽然**默认不走** REST，但 Panel 提供了两个 REST 端点：

| 方法 | 路由 | 用途 |
|------|------|------|
| POST | `/api/client/servers/{server}/power` | 电源动作，body: `{"signal": "start|stop|restart|kill"}` |
| POST | `/api/client/servers/{server}/command` | 控制台命令，body: `{"command": "..."}` |

**使用场景**：
- 第三方脚本 / 客户端 API Key 调用
- Panel 内部的调度系统（Schedule）
- 任何无法建立 WebSocket 长连接的场景

---

## 3. Panel 后端：路由注册与中间件管道

### 3.1 路由入口

定义于 [routes/api-client.php#L57-L74](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/routes/api-client.php#L57-L74)，由 [RouteServiceProvider.php#L56-L59](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Providers/RouteServiceProvider.php#L56-L59) 挂载：

```php
Route::group([
    'prefix' => '/servers/{server}',
    'middleware' => [
        ServerSubject::class,              // 活动日志：标记 server_id
        AuthenticateServerAccess::class,   // ★ 核心鉴权中间件
        ResourceBelongsToServer::class,    // 子资源归属校验
    ],
], function () {
    Route::post('/command', [PowerController::class, 'index']);   // 注意：这里写错了，实际应该是 CommandController
    Route::post('/power',   [PowerController::class, 'index']);

    // Websocket 单独加了资源级节流
    Route::middleware([ResourceLimit::Websocket->middleware()])
        ->get('/websocket', WebsocketController::class);
});
```

**Client API 全局中间件栈**（[HttpKernel.php#L82-L85](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Http/Kernel.php#L82-L85)）：
```
client-api 组:
  ├─ SubstituteClientBindings      // 用 uuid/identifier 解析路由模型
  ├─ RequireClientApiKey           // 可选：API Key 鉴权 + IP 白名单
api 组（外层）:
  ├─ EnsureStatefulRequests        // Cookie/Session 激活
  ├─ auth:sanctum                  // 用户鉴权 (Sanctum)
  ├─ IsValidJson                   // 请求体 JSON 格式校验
  ├─ TrackAPIKey                   // 活动日志：标记 API Key ID
  ├─ RequireTwoFactorAuthentication
  ├─ AuthenticateIPAccess          // IP 白名单校验
throttle:api.client（最外层）      // ★ 全局节流
```

### 3.2 全局节流（Rate Limiting）

定义于 [RouteServiceProvider.php#L93-L100](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Providers/RouteServiceProvider.php#L93-L100) + [config/http.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/config/http.php)：

```php
RateLimiter::for('api.client', function (Request $request) {
    $key = optional($request->user())->uuid ?: $request->ip();
    return Limit::perMinutes(
        config('http.rate_limit.client_period'),   // 默认 1 分钟
        config('http.rate_limit.client')            // 默认 256 次
    )->by($key);
});
```

**关键特性**：
- **按用户优先**：已登录用户使用 `user.uuid` 作为限流 key，未登录用 IP。避免用户通过换 IP 绕过限流。
- **默认值**：`APP_API_CLIENT_RATELIMIT=256` 次/分钟。对于正常用户（点按钮 + 敲命令）来说绰绰有余；对于脚本批量操作会触发 429。
- **Application API（管理员端）** 同样是 256 次/分钟，key 生成方式相同。

### 3.3 资源级节流（ResourceLimit Enum）

定义于 [app/Enum/ResourceLimit.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Enum/ResourceLimit.php)：

```php
enum ResourceLimit {
    case Websocket;   // ★ /api/client/servers/{server}/websocket
    case Allocation;
    case Backup;
    case Database;
    case Schedule;
    case Subuser;
    case FilePull;

    public function limit(): Limit {
        return match($this) {
            self::Backup     => Limit::perMinutes(15, 3),    // 15 分钟 3 个备份
            self::Database   => Limit::perMinute(2),
            self::FilePull   => Limit::perMinutes(10, 5),
            self::Subuser    => Limit::perMinutes(15, 10),
            self::Websocket  => Limit::perMinute(5),         // ★ 每台服务器每分钟 5 次 WS 握手
            default          => Limit::perMinute(2),
        };
    }

    public static function boot(): void {
        foreach (self::cases() as $case) {
            RateLimiter::for($case->throttleKey(), function (Request $request) use ($case) {
                $server = $request->route()->parameter('server');
                return $case->limit()->by($server->uuid);  // 按 server_uuid 限流！
            });
        }
    }
}
```

**要点**：
- **Websocket 限流 5 次/分钟/服务器**：防止客户端疯狂重建 WebSocket 连接（正常情况 10 分钟一次刷新 token）。
- **不是按用户，是按 server**：`by($server->uuid)` — 即使用户开 100 个标签页连同一个服务器，共享 5 次/分钟配额。
- **电源 / 命令端点无资源级限流**：仅依赖全局 256 次/分钟 + Wings 自身的限频器。

---

## 4. Panel 后端：白名单与语义鉴别

### 4.1 电源动作白名单

**验证类**：[SendPowerRequest.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Http/Requests/Api/Client/Servers/SendPowerRequest.php)

```php
class SendPowerRequest extends ClientApiRequest
{
    public function permission(): string {
        switch ($this->input('signal')) {
            case 'start':                  return Permission::ACTION_CONTROL_START;    // control.start
            case 'stop':
            case 'kill':                   return Permission::ACTION_CONTROL_STOP;     // control.stop
            case 'restart':                return Permission::ACTION_CONTROL_RESTART;  // control.restart
        }
        return '__invalid';   // ★ 未知 signal 会被授权层拒绝
    }

    public function rules(): array {
        return [
            // ★ 白名单：只允许这 4 个值
            'signal' => 'required|string|in:start,stop,restart,kill',
        ];
    }
}
```

**两层保护**：
1. **Laravel Validation `in:`**：请求到达控制器前，若 `signal` 不在白名单内，直接返回 422（ValidationException）。
2. **`permission()` 映射到 `__invalid`**：即使绕过了 validation（理论上不可能），授权层也会因为找不到 `__invalid` 权限而拒绝。

### 4.2 控制台命令白名单

**验证类**：[SendCommandRequest.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Http/Requests/Api/Client/Servers/SendCommandRequest.php)

```php
class SendCommandRequest extends ClientApiRequest
{
    public function permission(): string {
        return Permission::ACTION_CONTROL_CONSOLE;  // control.console
    }

    public function rules(): array {
        return [
            'command' => 'required|string|min:1',  // ★ 注意：无内容白名单！
        ];
    }
}
```

**关键区别**：
- 命令**不做白名单过滤**。原因：游戏服控制台命令集千差万别（Minecraft `/op`、Source `sm_kick`、Rust `server.save` 等），Panel 无法穷举。
- **权限门槛**：只需拥有 `control.console` 权限即可发送任意命令。
- **命令过滤职责在 Wings / Egg 配置**：Wings 自身维护 `disallow` 列表（危险命令黑名单），具体取决于 Egg 配置的 `config.yml`。

### 4.3 调度任务白名单

**验证类**：[StoreTaskRequest.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Http/Requests/Api/Client/Servers/Schedules/StoreTaskRequest.php)

```php
public function rules(): array {
    return [
        // ★ 调度任务 action 白名单
        'action'            => 'required|in:command,power,backup',
        'payload'           => 'required_unless:action,backup|string|nullable',
        'time_offset'       => 'required|numeric|min:0|max:900',  // 最大延迟 15 分钟
        'continue_on_failure' => 'sometimes|required|boolean',
    ];
}
```

注意：`action=power` 时，`payload` 的取值（`start|stop|restart|kill`）只在 [RunTaskJob.php#L62-L63](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Jobs/Schedule/RunTaskJob.php#L62-L63) 交给 `DaemonPowerRepository` 发送时**由 Wings 侧校验**，Panel 不二次校验 `payload` 白名单。

---

## 5. Panel 后端：权限鉴权模型

### 5.1 请求层权限解析（ClientApiRequest::authorize）

**基类**：[ClientApiRequest.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Http/Requests/Api/Client/ClientApiRequest.php)

```php
class ClientApiRequest extends ApplicationApiRequest
{
    public function authorize(): bool {
        // 若子类定义了 permission() 方法，就用 Gate 校验
        if ($this instanceof ClientPermissionsRequest || method_exists($this, 'permission')) {
            $server = $this->route()->parameter('server');
            if ($server instanceof Server) {
                // 调用 ServerPolicy，传入 permission 字符串
                return $this->user()->can($this->permission(), $server);
            }
            return false;
        }
        return true;  // 未定义 permission() 的请求默认通过
    }
}
```

**权限检查失败的后果**：Laravel Gate 返回 false → 抛出 `AuthorizationException` → 渲染为 HTTP 403。

### 5.2 权限判定策略（ServerPolicy）

**策略类**：[ServerPolicy.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Policies/ServerPolicy.php)

```php
class ServerPolicy
{
    // 优先级最高的短路判断
    public function before(User $user, string $ability, Server $server): ?bool {
        if ($user->root_admin || $server->owner_id === $user->id) {
            return true;   // ★ 管理员 / 服务器所有者：无条件放行
        }
        return $this->checkPermission($user, $server, $ability);
    }

    protected function checkPermission(User $user, Server $server, string $permission): bool {
        $subuser = $server->subusers->where('user_id', $user->id)->first();
        if (!$subuser || empty($permission)) return false;
        // 子用户权限是一个存 JSON/text 的数组：['control.start', 'control.stop', ...]
        return in_array($permission, $subuser->permissions);
    }

    // __call 魔术方法：避免 Laravel 因 policy 方法不存在而跳过 before()
    public function __call(string $name, mixed $arguments) {}
}
```

**权限常量汇总**（[Permission.php#L18-L22](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Models/Permission.php#L18-L22)）：

| 常量 | 值 | 含义 |
|------|-----|------|
| `ACTION_WEBSOCKET_CONNECT` | `websocket.connect` | 建立 WebSocket（View Console 基础） |
| `ACTION_CONTROL_CONSOLE` | `control.console` | 发送命令到控制台 |
| `ACTION_CONTROL_START` | `control.start` | Start 服务器 |
| `ACTION_CONTROL_STOP` | `control.stop` | Stop / Kill 服务器 |
| `ACTION_CONTROL_RESTART` | `control.restart` | Restart 服务器 |

### 5.3 服务器状态门禁（AuthenticateServerAccess）

**中间件**：[AuthenticateServerAccess.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Http/Middleware/Api/Client/Server/AuthenticateServerAccess.php)

```php
public function handle(Request $request, \Closure $next): mixed
{
    // 1. 身份归属校验：owner / root_admin / subuser，否则 404（隐私保护）
    if ($user->id !== $server->owner_id && !$user->root_admin) {
        if (!$server->subusers->contains('user_id', $user->id)) {
            throw new NotFoundHttpException();  // ★ 404 而非 403，避免探测
        }
    }

    // 2. 状态冲突校验：Server::validateCurrentState()
    try {
        $server->validateCurrentState();
    } catch (ServerStateConflictException $exception) {
        // view endpoint (GET /server) 例外：允许查看状态
        if (!$request->routeIs('api:client:server.view')) {
            // suspended / node_maintenance 例外：允许看 /resources
            if (($server->isSuspended() || $server->node->isUnderMaintenance())
                && !$request->routeIs('api:client:server.resources')) {
                throw $exception;   // HTTP 409 Conflict
            }
            // 非管理员且路由不在 except 列表（仅 websocket）→ 抛异常
            if (!$user->root_admin || !$request->routeIs($this->except)) {
                throw $exception;
            }
        }
    }
    return $next($request);
}
```

**`validateCurrentState()`** 判定逻辑（[Server.php#L390-L401](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Models/Server.php#L390-L401)）：

```php
public function validateCurrentState()
{
    if (
        $this->isSuspended()              // status = 'suspended'
        || $this->node->isUnderMaintenance()  // node.maintenance_mode
        || !$this->isInstalled()          // 未安装完成
        || $this->status === self::STATUS_RESTORING_BACKUP
        || !is_null($this->transfer)      // 正在跨节点迁移
    ) {
        throw new ServerStateConflictException($this);  // 409 Conflict
    }
}
```

**结论**：当服务器处于 suspended / 维护中 / 安装中 / 还原备份 / 迁移中时，**`/power` 和 `/command` 端点均拒绝请求（409）**。WebSocket 是唯一例外（管理员可连接以查看迁移日志）。

---

## 6. Panel → Wings：HTTP 调用链

### 6.1 DaemonRepository 基类：Guzzle HTTP 客户端

**基类**：[DaemonRepository.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Repositories/Wings/DaemonRepository.php)

```php
abstract class DaemonRepository
{
    public function getHttpClient(array $headers = []): Client {
        Assert::isInstanceOf($this->node, Node::class);

        return new Client([
            'verify'          => $this->app->environment('production'),   // 生产强制 HTTPS 验证
            'base_uri'        => $this->node->getConnectionAddress(),
            'timeout'         => config('pterodactyl.guzzle.timeout'),         // 默认 15s
            'connect_timeout' => config('pterodactyl.guzzle.connect_timeout'), // 默认 5s
            'headers' => array_merge($headers, [
                // ★ 使用 Node 的加密密钥（AES-256-CBC 加密存储）作为 Bearer Token
                'Authorization' => 'Bearer ' . $this->node->getDecryptedKey(),
                'Accept'        => 'application/json',
                'Content-Type'  => 'application/json',
            ]),
        ]);
    }
}
```

**超时配置**（[config/pterodactyl.php#L79-L82](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/config/pterodactyl.php#L79-L82)）：
- `GUZZLE_TIMEOUT=15`：总请求超时 15 秒
- `GUZZLE_CONNECT_TIMEOUT=5`：TCP 连接超时 5 秒

如果 Wings 执行 `stop`（优雅关闭）超过 15 秒（例如 `stop` 需要给游戏服发送 `/stop` 并等待玩家踢出 + 存档），Panel 会抛 504。**但实际 Wings 内部是异步执行的，POST 只是入队，通常会立即返回 204。**

### 6.2 DaemonPowerRepository：电源动作下发

**文件**：[DaemonPowerRepository.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Repositories/Wings/DaemonPowerRepository.php)

```php
public function send(string $action): ResponseInterface
{
    Assert::isInstanceOf($this->server, Server::class);

    try {
        return $this->getHttpClient()->post(
            sprintf('/api/servers/%s/power', $this->server->uuid),
            ['json' => ['action' => $action]]  // body: {"action": "start"}
        );
    } catch (TransferException $exception) {
        throw new DaemonConnectionException($exception);  // ★ 统一包装
    }
}
```

对应 Wings Endpoint：`POST /api/servers/{uuid}/power`

### 6.3 DaemonCommandRepository：控制台命令下发

**文件**：[DaemonCommandRepository.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Repositories/Wings/DaemonCommandRepository.php)

```php
public function send(array|string $command): ResponseInterface
{
    try {
        return $this->getHttpClient()->post(
            sprintf('/api/servers/%s/commands', $this->server->uuid),
            [
                // ★ 支持批量发送命令！总是包装为数组
                'json' => ['commands' => is_array($command) ? $command : [$command]],
            ]
        );
    } catch (TransferException $exception) {
        throw new DaemonConnectionException($exception);
    }
}
```

**调度系统的妙用**：[RunTaskJob.php#L66](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Jobs/Schedule/RunTaskJob.php#L66) 传的是 string（单条命令），而 `commands` 数组设计允许后续批量下发。

### 6.4 Activity 审计日志埋点

**PowerController.php#L31**（[PowerController.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Http/Controllers/Api/Client/Servers/PowerController.php#L31)）：
```php
Activity::event(strtolower("server:power.{$request->input('signal')}"))->log();
// 事件名: server:power.start / server:power.stop / server:power.restart / server:power.kill
```

**CommandController.php#L46**（[CommandController.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Http/Controllers/Api/Client/Servers/CommandController.php#L46)）：
```php
Activity::event('server:console.command')
    ->property('command', $request->input('command'))  // ★ 记录具体命令！
    ->log();
```

**注意**：Activity 记录在 **Wings 调用成功之后**执行（try 块外部）。如果 Wings 调用失败抛异常，Activity 不会记录，避免假阳性日志。

---

## 7. Panel → Wings：WebSocket 通道（JWT 鉴权）

### 7.1 JWT 签发服务（NodeJWTService）

**服务类**：[NodeJWTService.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Services/Nodes/NodeJWTService.php)

由 [WebsocketController.php#L55-L62](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Http/Controllers/Api/Client/Servers/WebsocketController.php#L55-L62) 调用：

```php
$token = $this->jwtService
    ->setExpiresAt(CarbonImmutable::now()->addMinutes(10))   // 10 分钟过期
    ->setUser($request->user())
    ->setClaims([
        'server_uuid'  => $server->uuid,
        'permissions'  => $permissions,   // ★ 用户在该服务器的权限列表嵌入 JWT
    ])
    ->handle($node, $user->id . $server->uuid);
```

**JWT Header & Payload 结构**：

```
Header:
  alg: HS256               // 对称加密，使用 Node 的密钥签名
  jti: md5(user_id + uuid) // 用于 deny-list（注销/踢人）

Payload (Claims):
  iss: APP_URL             // 签发者
  aud: node 连接地址       // 接收方
  iat: 签发时间            // Wings 会检查 "created too far in past"
  nbf: iat - 5min          // 允许 5 分钟时钟偏差
  exp: iat + 10min         // 过期时间
  sub: 用户标识
  user_uuid: 用户 UUID
  user_id: 用户自增 ID (兼容旧版 Wings)
  server_uuid: 服务器 UUID
  permissions: [ "websocket.connect", "control.start", ... ]  // ★ Wings 自行鉴权！
  unique_id: Str::random() // 防重放
```

**Wings 侧 JWT 权限检查要点**：
- Wings 收到 `send command` 事件时，检查 JWT 中的 `permissions` 是否含 `control.console`
- 收到 `set state` 事件时，按 action 检查 `control.start|stop|restart`
- 无需回查 Panel，全部在 Wings 本地完成，保证低延迟

### 7.2 前端 WebSocket 封装（Websocket.ts + Sockette）

**文件**：[Websocket.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/resources/scripts/plugins/Websocket.ts)

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
            onopen: () => this.authenticate(),  // 打开立即发 auth
            // ...
        });
    }

    // ★ 发送协议: { event: string, args: string[] }
    send(event: string, payload?: string | string[]) {
        this.socket?.json({ event, args: Array.isArray(payload) ? payload : [payload] });
    }
}
```

**Wings → Panel 方向的事件**（前端监听，见 [Console.tsx#L170-L178](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/resources/scripts/components/server/console/Console.tsx#L170-L178)）：

| SocketEvent | 值 | 用途 |
|-------------|-----|------|
| `STATUS` | `"status"` | 电源状态变化：`starting|running|stopping|offline` |
| `CONSOLE_OUTPUT` | `"console output"` | 游戏服控制台输出流 |
| `DAEMON_MESSAGE` | `"daemon message"` | Wings 系统消息（前缀黄色高亮） |
| `DAEMON_ERROR` | `"daemon error"` | Wings 错误（前缀红色高亮） |
| `STATS` | `"stats"` | CPU/内存/磁盘/网络实时统计 |
| `INSTALL_*` | 安装相关 | 首次安装/重装进度 |
| `TRANSFER_*` | 迁移相关 | 跨节点迁移日志 |

### 7.3 事件枚举：前端发什么、收什么

定义于 [events.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/resources/scripts/components/server/events.ts)：

```ts
export enum SocketRequest {  // 前端 → Wings
    SEND_LOGS = 'send logs',   // 拉取历史 2MB 控制台缓冲
    SEND_STATS = 'send stats', // 立即发送一次统计数据（否则默认心跳）
    SET_STATE = 'set state',   // ★ 电源动作
    // 注: "send command" 未列入枚举，但 Console.tsx 直接用字符串
}

export enum SocketEvent {     // Wings → 前端
    DAEMON_MESSAGE = 'daemon message',
    DAEMON_ERROR = 'daemon error',
    CONSOLE_OUTPUT = 'console output',
    STATUS = 'status',
    STATS = 'stats',
    // ...
}
```

---

## 8. Wings 侧：接收约定与并发模型（推断自 Panel 调用协议）

> 本代码库仅包含 Panel，不含 Wings。以下内容基于 Panel 发出的调用约定 + Pterodactyl 公开架构推断。

### 8.1 REST Endpoint 约定

| Method | Path | Body | Wings 预期行为 |
|--------|------|------|--------------|
| POST | `/api/servers/{uuid}/power` | `{"action":"start\|stop\|restart\|kill"}` | 204 No Content 立即返回；内部异步改变进程状态 |
| POST | `/api/servers/{uuid}/commands` | `{"commands":["cmd1","cmd2"]}` | 服务器必须 running，否则 502 Bad Gateway；否则立即 204 |
| GET | `/api/servers/{uuid}` | — | 返回 `{"state": "running|offline|...", "suspended": bool, ...}` |
| GET | `/api/servers/{uuid}/ws` | Upgrade | WebSocket 握手 |

### 8.2 WebSocket Event 约定

**Client → Wings（需带 JWT 权限）**：

| Event | Args | 所需权限 | 说明 |
|-------|------|---------|------|
| `auth` | `[jwt]` | — | 连接后第一条消息 |
| `send logs` | `[]` | `websocket.connect` | 请求 Wings 推送历史缓冲 |
| `send stats` | `[]` | `websocket.connect` | 请求立即发送统计 |
| `set state` | `["start"\|"stop"\|"restart"\|"kill"]` | 对应 `control.*` | 等同 REST /power |
| `send command` | `["/op Notch"]` | `control.console` | 等同 REST /commands（单条） |

**Wings → Client**：

| Event | Args | 触发时机 |
|-------|------|---------|
| `auth success` | — | JWT 校验通过 |
| `token expiring` | — | JWT 距过期 < 3min |
| `token expired` | — | JWT 已过期 |
| `jwt error` | `[msg]` | JWT 校验失败 |
| `status` | `["starting"]` | 电源状态迁移 |
| `console output` | `["[12:34] ..."]` | 游戏服 stdout 有新行 |
| `stats` | `[{"cpu":10.5,"memory_bytes":...}]` | 统计心跳（默认 1~2s） |

### 8.3 并发安全与节流（Panel 侧配合）

Panel 自身的**第一道防线**：
1. **前端按钮禁用逻辑**（[PowerButtons.tsx#L52-L69](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/resources/scripts/components/server/console/PowerButtons.tsx#L52-L69)）：
   - `start` 仅在 `status === 'offline'` 时可用
   - `stop/kill` 仅在 `status !== 'offline'` 时可用
   - 有效阻止重复点击 Start

2. **全局节流 256/min**：防止脚本刷接口。

3. **服务器状态门禁 409**：suspended / maintenance / installing 等状态下，接口直接拒绝。

**Wings 侧（推断）** 还会做：
- **Instance.powerMux sync.Mutex**：电源动作串行化，避免 Start 与 Kill 同时到达导致进程孤儿
- **console ring buffer**：命令输入 chan 带缓冲，消费端按 50ms 批量 flush 到游戏服 stdin，防刷屏
- **命令限频器**（基于令牌桶）：默认 60 条/分钟/实例

---

## 9. 调度系统：计划任务中的电源/命令

### 9.1 ProcessScheduleService：事务入队

**服务**：[ProcessScheduleService.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Services/Schedules/ProcessScheduleService.php)

```php
public function handle(Schedule $schedule, bool $now = false): void
{
    $task = $schedule->tasks()->orderBy('sequence_id')->first();

    // ★ 数据库事务：标记 is_processing + is_queued，避免重复调度
    $this->connection->transaction(function () use ($schedule, $task) {
        $schedule->forceFill([
            'is_processing' => true,
            'next_run_at'   => $schedule->getNextRunDate(),  // 提前计算下次运行
        ])->saveOrFail();
        $task->update(['is_queued' => true]);
    });

    $job = new RunTaskJob($task, $now);

    // ★ only_when_online 开关：仅服务器在线时执行
    if ($schedule->only_when_online) {
        // → 先调 Wings GET /api/servers/{uuid} 查 state
        $details = $this->serverRepository->setServer($schedule->server)->getDetails();
        $state = $details['state'] ?? 'offline';
        if (in_array($state, ['offline', 'stopping'])) {
            $job->failed();   // 标记完成，不抛异常
            return;
        }
    }
    // → 入队，延迟 time_offset 秒
    $this->dispatcher->dispatch($job->delay($task->time_offset));
}
```

### 9.2 RunTaskJob：按序执行与失败降级

**任务**：[RunTaskJob.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Jobs/Schedule/RunTaskJob.php)

```php
public function handle(DaemonCommandRepository $commandRepository,
                       InitiateBackupService $backupService,
                       DaemonPowerRepository $powerRepository)
{
    // 1. 防御性检查：服务器被暂停了？
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
        // ★ 唯一失败降级：continue_on_failure + DaemonConnectionException
        if (!($this->task->continue_on_failure && $exception instanceof DaemonConnectionException)) {
            throw $exception;   // 其他异常：任务链中断
        }
    }

    $this->markTaskNotQueued();
    $this->queueNextTask();   // → 取下一个 sequence_id，delay 后入队
}
```

**任务链执行模型**：
```
Schedule (is_processing=true)
  ├─ Task #1 (sequence_id=1, time_offset=0)  → dispatchNow/dispatch + delay
  │    └─ success → markTaskNotQueued → dispatch Task #2
  │    └─ failure + continue_on_failure=true  → dispatch Task #2
  │    └─ failure 其他                        → failed()，中断
  ├─ Task #2 (sequence_id=2, time_offset=30)
  ...
  └─ 最后一个 Task 完成 → markScheduleComplete(is_processing=false, last_run_at=now)
```

---

## 10. 失败处理与状态回滚

### 10.1 DaemonConnectionException：Wings 通讯异常封装

**异常类**：[DaemonConnectionException.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Exceptions/Http/Connection/DaemonConnectionException.php)

```php
public function __construct(GuzzleException $previous, bool $useStatusCode = true)
{
    $response = method_exists($previous, 'getResponse') ? $previous->getResponse() : null;
    $this->requestId = $response?->getHeaderLine('X-Request-Id'); // ★ 便于 Wings 日志排查

    // 状态码映射：2XX 却进了异常 → 升级为 502 Bad Gateway
    if ($useStatusCode) {
        $this->statusCode = is_null($response) ? 504 : $response->getStatusCode();
        if ($this->statusCode < 400) $this->statusCode = 502;
    }

    // 消息分级：5XX = error（写 Laravel ERROR 日志），其余 warning
    $level = $this->statusCode >= 500 && $this->statusCode !== 504
        ? DisplayException::LEVEL_ERROR
        : DisplayException::LEVEL_WARNING;
}
```

**`report()` 方法**：自动将异常 + `request_id` 写入 Laravel Log，便于在 Wings 侧用同一个 X-Request-Id 关联日志。

### 10.2 CommandController：502 → 用户可读错误

**CommandController.php#L32-L44**（[CommandController.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Http/Controllers/Api/Client/Servers/CommandController.php#L32-L44)）：

```php
try {
    $this->repository->setServer($server)->send($request->input('command'));
} catch (DaemonConnectionException $exception) {
    $previous = $exception->getPrevious();
    if ($previous instanceof BadResponseException) {
        // ★ Wings 返回 502：意味着游戏服没有运行（容器未启动 / stdin 管道关闭）
        if ($previous->getResponse()->getStatusCode() === Response::HTTP_BAD_GATEWAY) {
            throw new HttpException(
                Response::HTTP_BAD_GATEWAY,
                'Server must be online in order to send commands.',
                $exception
            );
        }
    }
    throw $exception;  // 其他错误原样抛（500, 504, 4XX）
}
```

**前端拿到后的处理**：axios 拦截器 → `httpErrorToHuman()` → 提取 `errors[0].detail` 并 toast 展示。

### 10.3 电源动作失败：无回滚，靠 WebSocket 状态事件最终一致

PowerController.php 中 `DaemonPowerRepository::send()` 抛出 `DaemonConnectionException` **直接冒泡**，无 catch 处理：
- 用户前端看到 502/504 toast 错误
- **但数据库状态并未回滚的必要性**：Panel 数据库本身没有存当前电源状态（`Server.status` 字段仅用于 `suspended/installing/restoring_backup` 等**非运行态**元信息，不存 running/offline）

**电源状态来源**：完全由 Wings WebSocket 的 `status` 事件驱动。即使 Panel 发 Start 请求失败，只要服务器实际上还是 offline，WebSocket 会把 `status=offline` 推给前端，Start 按钮重新可用。**最终一致性而非强一致**。

### 10.4 调度任务失败：continue_on_failure 开关

| 场景 | 失败行为 |
|------|---------|
| DaemonConnectionException（Wings 连不上）+ `continue_on_failure=true` | 记录日志，**继续执行后续 Task** |
| DaemonConnectionException + `continue_on_failure=false` | 抛异常 → Job 失败 → 调度链中断 |
| 任何其他 Exception（Validation / Authorization） | 抛异常 → 调度链中断 |
| `only_when_online=true` + 服务器 offline | 默默标记完成，不抛异常 |

**中断后的清理**：`RunTaskJob::failed()` 钩子会调用 `markTaskNotQueued() + markScheduleComplete()`，释放 `is_processing` 锁，避免 Schedule 永久卡死。

### 10.5 Panel 数据库状态：乐观设计，无事务回滚

整个电源/命令链路中，**Panel 数据库只有以下写入**，而且均是"事后记录"而非"事前锁"：

| 写入点 | 数据 | 失败时回滚？ |
|--------|------|------------|
| PowerController | Activity Log（`server:power.*`） | ✅ 异常未捕获则不写入 |
| CommandController | Activity Log（`server:console.command`，含命令内容） | ✅ 异常未捕获则不写入 |
| 调度 `ProcessScheduleService` | `schedules.is_processing`, `tasks.is_queued` | ✅ 在 DB 事务中，异常自动回滚 |
| 调度 `RunTaskJob::failed()` | `is_processing=false`, `last_run_at=now` | 非事务，保证最终释放锁 |

**无状态回滚的设计理由**：
- Panel 本质上是**控制平面**，游戏服实际运行状态在 Wings/容器中
- Panel 只做"发指令"和"记日志"，不维护权威状态副本
- 状态由 Wings 通过 WebSocket 推送回来，异步最终一致

---

## 11. 客服视角：Power Action 语义对照表

> 针对客服把按钮当命令行的场景，整理"按钮按下 → 代码走向 → Wings 行为 → 玩家感知"：

| 按钮 | 实际发送值 | Wings 典型实现 | 玩家体感 | 风险 |
|------|-----------|--------------|---------|------|
| **Start** | `start` | `docker start` 或创建容器 → 执行 `startup_command` | 服务器开始启动，数秒到数分钟后可进入 | 正常操作，几乎无风险 |
| **Stop** | `stop` | ① 向容器 stdin 写入 Egg 配置的 `stop_command`（如 `/stop`）② 等待 `stop_timeout`（默认 30s）③ 未退出则 SIGTERM → SIGKILL | 玩家收到"服务器关闭中"提示，地图正常存档 | **优雅关闭**，应默认使用 |
| **Restart** | `restart` | 先执行 stop 流程，容器退出后自动 start | 玩家被踢出 → 等待重连 | 可能触发玩家被踢 |
| **Kill** | `kill` | 直接向容器进程组发 SIGKILL（不走 stop_command） | 瞬间掉线，**无存档提示** | ⚠️ 地图/世界可能损坏；只能在"Stop 卡死、无响应"时使用 |

**客服应传达的操作原则**：
1. 日常维护一律用 **Stop**，不用 Kill
2. 只有当 Stop 超过 2 分钟控制台仍无响应时，才升级到 Kill
3. Kill 之后建议检查游戏服最新存档完整性，再执行 Start
4. **按钮 = 电源动作，不是命令输入框**。要执行游戏内命令（`/op`, `whitelist add` 等），使用 Console 下方命令输入框

---

## 12. 关键文件索引

### 12.1 前端 (React/TypeScript)

| 文件 | 职责 |
|------|------|
| [PowerButtons.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/resources/scripts/components/server/console/PowerButtons.tsx) | 电源按钮 UI + WebSocket "set state" 发送 |
| [Console.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/resources/scripts/components/server/console/Console.tsx) | xterm.js 终端 + WebSocket "send command" 发送 + 历史记录 |
| [WebsocketHandler.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/resources/scripts/components/server/WebsocketHandler.tsx) | WebSocket 生命周期管理 + JWT 自动刷新 |
| [Websocket.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/resources/scripts/plugins/Websocket.ts) | Sockette 封装 + 协议序列化 |
| [Can.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/resources/scripts/components/elements/Can.tsx) | 基于权限的条件渲染组件 |
| [events.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/resources/scripts/components/server/events.ts) | WebSocket 事件名枚举（SocketEvent / SocketRequest） |
| [ServerConsoleContainer.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/resources/scripts/components/server/console/ServerConsoleContainer.tsx) | 控制台页面容器，PowerAction 类型定义 |
| [getWebsocketToken.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/resources/scripts/api/server/getWebsocketToken.ts) | REST 请求 WebSocket JWT |
| [http.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/resources/scripts/api/http.ts) | Axios 实例（withCredentials=true, 20s 超时, 进度条） |

### 12.2 Panel 后端 (PHP Laravel)

#### 路由 & 中间件

| 文件 | 职责 |
|------|------|
| [routes/api-client.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/routes/api-client.php) | `/api/client/servers/{server}/*` 路由定义 |
| [Http/Kernel.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Http/Kernel.php) | 中间件栈定义（client-api / api / throttle 别名）|
| [Providers/RouteServiceProvider.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Providers/RouteServiceProvider.php) | 路由挂载 + 全局限流器（api.client: 256/min）+ ResourceLimit::boot() |
| [Enum/ResourceLimit.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Enum/ResourceLimit.php) | 资源级限流器（Websocket 5/min/sever 等）|
| [Middleware/.../AuthenticateServerAccess.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Http/Middleware/Api/Client/Server/AuthenticateServerAccess.php) | 服务器归属 + 状态冲突门禁（409）|

#### 控制器

| 文件 | 职责 |
|------|------|
| [PowerController.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Http/Controllers/Api/Client/Servers/PowerController.php) | `POST /power` → DaemonPowerRepository + Activity |
| [CommandController.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Http/Controllers/Api/Client/Servers/CommandController.php) | `POST /command` → DaemonCommandRepository + 502 转用户可读错误 + Activity |
| [WebsocketController.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Http/Controllers/Api/Client/Servers/WebsocketController.php) | `GET /websocket` → 签发 10min JWT（含权限列表）|

#### 请求验证（白名单）

| 文件 | 职责 |
|------|------|
| [SendPowerRequest.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Http/Requests/Api/Client/Servers/SendPowerRequest.php) | `signal in:start,stop,restart,kill` + 权限映射 |
| [SendCommandRequest.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Http/Requests/Api/Client/Servers/SendCommandRequest.php) | `command required|min:1` + `control.console` 权限 |
| [ClientApiRequest.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Http/Requests/Api/Client/ClientApiRequest.php) | `authorize()` → `user()->can(permission(), server)` |
| [.../Schedules/StoreTaskRequest.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Http/Requests/Api/Client/Servers/Schedules/StoreTaskRequest.php) | `action in:command,power,backup` + `max:900` time_offset |

#### Wings 通信层

| 文件 | 职责 |
|------|------|
| [DaemonRepository.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Repositories/Wings/DaemonRepository.php) | Guzzle 客户端 + Node Bearer Token + 超时配置 |
| [DaemonPowerRepository.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Repositories/Wings/DaemonPowerRepository.php) | `POST /api/servers/{uuid}/power` |
| [DaemonCommandRepository.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Repositories/Wings/DaemonCommandRepository.php) | `POST /api/servers/{uuid}/commands` |
| [DaemonServerRepository.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Repositories/Wings/DaemonServerRepository.php) | `GET /api/servers/{uuid}`（调度 only_when_online 检查用）|
| [DaemonConnectionException.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Exceptions/Http/Connection/DaemonConnectionException.php) | Wings 异常封装 + X-Request-Id 关联 + 日志分级 |

#### 权限 & 模型

| 文件 | 职责 |
|------|------|
| [Permission.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Models/Permission.php) | 权限常量（control.* / websocket.connect 等）|
| [Policies/ServerPolicy.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Policies/ServerPolicy.php) | Gate 判定：管理员/所有者放行，子用户查 permissions 数组 |
| [Models/Server.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Models/Server.php) | `validateCurrentState()`（挂起/维护/安装/还原/迁移 → 409）|
| [Services/Nodes/NodeJWTService.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Services/Nodes/NodeJWTService.php) | HS256 JWT 签发，嵌入 server_uuid + permissions + user_uuid |

#### 调度系统

| 文件 | 职责 |
|------|------|
| [Services/Schedules/ProcessScheduleService.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Services/Schedules/ProcessScheduleService.php) | 事务性入队 + only_when_online 检查 + 延迟分发 |
| [Jobs/Schedule/RunTaskJob.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208