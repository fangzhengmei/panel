# Pterodactyl Panel 控制台命令与电源动作 — 代码走向全链路分析

> **关于本仓库的范围声明**：此代码库仅包含 Panel（PHP Laravel + React 前端），不含 Wings 守护进程（Go）源码。凡是涉及 Wings 内部实现的描述，均在标题或段落开头明确标注【推断】，以示与 Panel 侧可确认代码的区别。

---

## 0. 目录

- [1. 整体架构概览](#1-整体架构概览)
- [2. 前端：两条下发通道（Panel 代码可确认）](#2-前端两条下发通道panel-代码可确认)
  - [2.1 电源按钮（WebSocket 通道）](#21-电源按钮websocket-通道)
  - [2.2 控制台命令（WebSocket 通道）](#22-控制台命令websocket-通道)
  - [2.3 WebSocket Token 获取（REST 桥）](#23-websocket-token-获取rest-桥)
  - [2.4 REST API 通道（供外部 / 调度系统）](#24-rest-api-通道供外部--调度系统)
- [3. Panel 后端：路由注册与中间件管道](#3-panel-后端路由注册与中间件管道)
  - [3.1 路由入口（已精确核对）](#31-路由入口已精确核对)
  - [3.2 全局限流（Rate Limiting）](#32-全局限流rate-limiting)
  - [3.3 资源级节流（ResourceLimit Enum）](#33-资源级节流resourcelimit-enum)
- [4. Panel 后端：白名单与语义鉴别](#4-panel-后端白名单与语义鉴别)
  - [4.1 电源动作白名单](#41-电源动作白名单)
  - [4.2 控制台命令白名单](#42-控制台命令白名单)
  - [4.3 调度任务白名单](#43-调度任务白名单)
- [5. Panel 后端：权限鉴权模型](#5-panel-后端权限鉴权模型)
  - [5.1 请求层权限解析（ClientApiRequest::authorize）](#51-请求层权限解析clientapirequestauthorize)
  - [5.2 权限判定策略（ServerPolicy）](#52-权限判定策略serverpolicy)
  - [5.3 服务器状态门禁（AuthenticateServerAccess）](#53-服务器状态门禁authenticateserveraccess)
- [6. Panel → Wings：HTTP 调用链（Panel 代码可确认）](#6-panel--wingshttp-调用链panel-代码可确认)
  - [6.1 DaemonRepository 基类：Guzzle HTTP 客户端](#61-daemonrepository-基类guzzle-http-客户端)
  - [6.2 DaemonPowerRepository：电源动作下发](#62-daemonpowerrepository电源动作下发)
  - [6.3 DaemonCommandRepository：控制台命令下发](#63-daemoncommandrepository控制台命令下发)
  - [6.4 Activity 审计日志埋点](#64-activity-审计日志埋点)
- [7. Panel → Wings：WebSocket 通道（JWT 鉴权）](#7-panel--wingswebsocket-通道jwt-鉴权)
  - [7.1 JWT 签发服务（NodeJWTService）](#71-jwt-签发服务nodejwtservice)
  - [7.2 前端 WebSocket 封装（Websocket.ts + Sockette）](#72-前端-websocket-封装websockett--sockette)
  - [7.3 事件枚举（前端发出 & 接收）](#73-事件枚举前端发出--接收)
- [8. Wings 侧（标注：以下均为推断）](#8-wings-侧标注以下均为推断)
  - [8.1 【推断】REST Endpoint 约定](#81-推断rest-endpoint-约定)
  - [8.2 【推断】WebSocket Event 约定](#82-推断websocket-event-约定)
  - [8.3 【推断】并发安全与节流](#83-推断并发安全与节流)
- [9. 调度系统：计划任务中的电源/命令](#9-调度系统计划任务中的电源命令)
  - [9.1 ProcessScheduleService：事务入队](#91-processscheduleservice事务入队)
  - [9.2 RunTaskJob：按序执行与失败降级](#92-runtaskjob按序执行与失败降级)
- [10. 失败处理与状态回滚（Panel 代码可确认）](#10-失败处理与状态回滚panel-代码可确认)
  - [10.1 DaemonConnectionException：Wings 通讯异常封装](#101-daemonconnectionexceptionwings-通讯异常封装)
  - [10.2 CommandController：502 → 用户可读错误](#102-commandcontroller502--用户可读错误)
  - [10.3 电源动作失败：无数据库状态回滚](#103-电源动作失败无数据库状态回滚)
  - [10.4 调度任务失败：continue_on_failure + failed() 释放锁](#104-调度任务失败continue_on_failure--failed-释放锁)
- [11. 客服视角：Power Action 语义对照表](#11-客服视角power-action-语义对照表)
- [12. 关键文件索引（Panel 侧可确认）](#12-关键文件索引panel-侧可确认)
- [附：客服 FAQ 速查](#附客服-faq-速查)

---

## 1. 整体架构概览

```
 ┌─────────────────────────────────────────────────────────────────────┐
 │                    浏览器前端 (React — Panel 代码)                    │
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
 │          │ REST GET /api/client/servers/ │                          │
 │          │   /{uuid}/websocket           │                          │
 │          │   (获取 JWT + socket URL)     │                          │
 │          └───────────────┬───────────────┘                          │
 └──────────────────────────┼──────────────────────────────────────────┘
                            │ HTTPS (withCredentials: true)
 ┌──────────────────────────┼──────────────────────────────────────────┐
 │                   Panel (PHP Laravel — 本仓库代码)                    │
 │  ┌───────────────────────▼──────────────────────────────┐           │
 │  │  RouteServiceProvider → throttle:api.client         │           │
 │  │  (默认 256 req/min per user)                         │           │
 │  └───────────────────────┬──────────────────────────────┘           │
 │          ┌────────────────┼─────────────────┐                       │
 │          ▼                ▼                 ▼                       │
 │  POST /command    POST /power      GET /websocket                   │
 │  CommandController PowerController  WebsocketController             │
 │  (L30-L48)         (L25-L33)        (L33-L71)                       │
 │          │                │                 │                       │
 │          └────────────────┴─────────────────┘                       │
 │                           │                                         │
 │          ┌────────────────▼──────────────────┐                      │
 │          │  DaemonPowerRepository             │                      │
 │          │  DaemonCommandRepository           │                      │
 │          │  (Guzzle + Node 对称加密密钥鉴权)    │                      │
 │          └────────────────┬──────────────────┘                      │
 └───────────────────────────┼──────────────────────────────────────────┘
                             │ HTTPS (Bearer: Node.decrypted_key)
 ┌───────────────────────────▼──────────────────────────────────────────┐
 │                 Wings (Go 守护进程 — 【不在本仓库，以下均为推断】)      │
 │  ┌─────────────────────────────────────────────────────────────┐     │
 │  │  REST:   POST /api/servers/{uuid}/power                     │     │
 │  │          POST /api/servers/{uuid}/commands                  │     │
 │  │  WS:     /api/servers/{uuid}/ws                             │     │
 │  └─────────────────────────────┬───────────────────────────────┘     │
 │                                ▼                                     │
 │                      游戏服进程 (Docker/Systemd) 【推断】              │
 └──────────────────────────────────────────────────────────────────────┘
```

**代码可确认的设计原则**（仅 Panel 侧，不含 Wings）：
1. **双轨下发**：电源/命令既可以走 REST（同步确认，供外部/调度），也可以走 WebSocket（实时双向，前端默认使用）。
2. **JWT 短时令牌**：WebSocket 通信使用 10 分钟过期的 JWT，由 Panel 签发并嵌入权限列表（代码可确认）。【推断】Wings 可本地校验 JWT 签名与权限列表而无需回查 Panel。
3. **白名单优先**：`signal` 字段使用 `in:start,stop,restart,kill` 枚举验证，拒绝未知值（代码可确认）。
4. **乐观状态模型**：Panel 数据库不维护 `running/offline` 电源状态字段（代码可确认），完全依赖 Wings 推送的 WebSocket `status` 事件；失败时无"数据库回滚"。

---

## 2. 前端：两条下发通道（Panel 代码可确认）

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

| Action | 触发方式 | 前端禁用条件（第 54/61/68 行） | 实际发送值 |
|--------|---------|------------------------------|-----------|
| `start` | 点击 Start | `status !== 'offline'` | `"start"` |
| `restart` | 点击 Restart | `!status`（状态未知时） | `"restart"` |
| `stop` | 点击 Stop（`status !== 'stopping'`） | `status === 'offline'` | `"stop"` |
| `kill` | ① Stop 过程中点按钮（killable=true）② 或二次确认弹窗点 Continue | `status === 'offline'` | `"kill"` |

**前端权限渲染控制**（使用 [Can.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/resources/scripts/components/elements/Can.tsx) 组件）：
- Start 按钮：`<Can action={'control.start'}>`（第 51 行）
- Restart 按钮：`<Can action={'control.restart'}>`（第 60 行）
- Stop/Kill 按钮：`<Can action={'control.stop'}>`（第 65 行）

**Kill 二次确认**：使用 `Dialog.Confirm`（第 41-50 行），文案：
> "Forcibly stopping a server can lead to data corruption."

### 2.2 控制台命令（WebSocket 通道）

**组件位置**：[Console.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/resources/scripts/components/server/console/Console.tsx)

```tsx
// 第 97-124 行：命令输入与发送
const handleCommandKeyDown = (e: React.KeyboardEvent<HTMLInputElement>) => {
    // 历史命令：↑/↓ 翻页，最多保存 32 条，持久化到 localStorage
    if (e.key === 'ArrowUp') { /* 取出历史上一条 */ }
    if (e.key === 'ArrowDown') { /* 取出历史下一条 */ }

    const command = e.currentTarget.value;
    if (e.key === 'Enter' && command.length > 0) {
        setHistory(prev => [command, ...prev!].slice(0, 32));

        // 通过 WebSocket 发送 "send command" 事件
        instance && instance.send('send command', command);
        e.currentTarget.value = '';
    }
};
```

**渲染权限控制**（第 66 行 + 第 211 行）：
```tsx
const [canSendCommands] = usePermissions(['control.console']);
// 仅当拥有 control.console 权限时才渲染命令输入框
{canSendCommands && <div className={...}><input ... onKeyDown={handleCommandKeyDown} /></div>}
```

**输入框禁用条件**（第 218 行）：`disabled={!instance || !connected}` — WebSocket 未连接时无法发送。

### 2.3 WebSocket Token 获取（REST 桥）

**核心组件**：[WebsocketHandler.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/resources/scripts/components/server/WebsocketHandler.tsx)

连接流程（代码可确认）：
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
    if (updatingToken) return;         // 防抖锁，避免并发刷新
    updatingToken = true;
    getWebsocketToken(uuid)
        .then(data => socket.setToken(data.token, true));
};
```

**重连策略**：[Websocket.ts#L22-L55](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/resources/scripts/plugins/Websocket.ts#L22-L55) 使用 Sockette 库：
- `timeout: 1000ms`
- `maxAttempts: 20`（最多重连 20 次）
- 收到 Wings 关闭码 4400/4409（服务器挂起/暂停）时停止重连。

### 2.4 REST API 通道（供外部 / 调度系统）

前端默认不走 REST，但 Panel 提供了以下两个 REST 端点（见 [routes/api-client.php#L72-L73](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/routes/api-client.php#L72-L73)）：

| 方法 | 路由 | Body | Controller |
|------|------|------|-----------|
| POST | `/api/client/servers/{server}/command` | `{"command": "..."}` | [CommandController@index](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Http/Controllers/Api/Client/Servers/CommandController.php#L30-L48) |
| POST | `/api/client/servers/{server}/power` | `{"signal": "start\|stop\|restart\|kill"}` | [PowerController@index](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Http/Controllers/Api/Client/Servers/PowerController.php#L25-L33) |

**使用场景**：
- 第三方脚本 / 客户端 API Key 调用
- Panel 内部调度系统（Schedule）
- 任何无法建立 WebSocket 长连接的场景

---

## 3. Panel 后端：路由注册与中间件管道

### 3.1 路由入口（已精确核对）

定义于 [routes/api-client.php#L66-L73](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/routes/api-client.php#L66-L73)，由 [RouteServiceProvider.php#L56-L59](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Providers/RouteServiceProvider.php#L56-L59) 挂载：

```php
// WebSocket 单独加了资源级节流
Route::middleware([ResourceLimit::Websocket->middleware()])
    ->get('/websocket', Client\Servers\WebsocketController::class)
    ->name('api:client:server.ws');

// 命令 & 电源
Route::post('/command', [Client\Servers\CommandController::class, 'index']);  // ★ 精确核对：CommandController
Route::post('/power',   [Client\Servers\PowerController::class, 'index']);    // ★ 精确核对：PowerController
```

**Client API 全局中间件栈**（[HttpKernel.php#L82-L85](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Http/Kernel.php#L82-L85)）：
```
最外层:
  └─ throttle:api.client                 // ★ 全局节流（见 3.2）
api 组:
  ├─ SubstituteBindings / ThrottleRequests
  ├─ EnsureStatefulRequests              // Cookie/Session 激活
  ├─ auth:sanctum                        // 用户鉴权 (Laravel Sanctum)
  ├─ IsValidJson                         // 请求体 JSON 格式校验
  ├─ TrackAPIKey                         // 活动日志：标记 API Key ID
  ├─ RequireTwoFactorAuthentication
  └─ AuthenticateIPAccess                // IP 白名单校验
client-api 组:
  ├─ SubstituteClientBindings            // 用 uuid/identifier 解析 Server 模型
  └─ RequireClientApiKey                 // 可选：API Key 鉴权 + IP 白名单
server 组 (嵌套在 /servers/{server} 下):
  ├─ ServerSubject (Activity Log 埋点)   // 标记 server_id
  ├─ AuthenticateServerAccess            // ★ 核心鉴权中间件（见 5.3）
  └─ ResourceBelongsToServer             // 子资源归属校验
```

### 3.2 全局限流（Rate Limiting）

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

**代码可确认的关键特性**：
- **按用户优先**：已登录用户使用 `user.uuid` 作为限流 key，未登录用 IP。避免用户通过换 IP 绕过限流。
- **默认值**：`APP_API_CLIENT_RATELIMIT=256` 次/分钟。
- `api.application`（管理员端）同样是 256 次/分钟，key 生成方式相同。

### 3.3 资源级节流（ResourceLimit Enum）

定义于 [app/Enum/ResourceLimit.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Enum/ResourceLimit.php)：

```php
enum ResourceLimit
{
    case Websocket;   // 已使用：/api/client/servers/{server}/websocket
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
                return $case->limit()->by($server->uuid);  // ★ 按 server_uuid 限流，非按用户
            });
        }
    }
}
```

**代码可确认的要点**：
- **Websocket 限流 5 次/分钟/服务器**：防止客户端疯狂重建 WebSocket 连接（正常情况 10 分钟一次刷新 token）。
- **按 server 限流，不是按用户**：`by($server->uuid)` — 即使用户开 100 个标签页连同一个服务器，共享 5 次/分钟配额。
- **电源 / 命令端点无资源级限流**：代码中 `/command` 和 `/power` 路由上没有附加 `ResourceLimit` 中间件，仅依赖全局 256 次/分钟限流。

---

## 4. Panel 后端：白名单与语义鉴别

### 4.1 电源动作白名单

**验证类**：[SendPowerRequest.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Http/Requests/Api/Client/Servers/SendPowerRequest.php)

```php
class SendPowerRequest extends ClientApiRequest
{
    public function permission(): string {
        switch ($this->input('signal')) {
            case 'start':                  return Permission::ACTION_CONTROL_START;    // 'control.start'
            case 'stop':
            case 'kill':                   return Permission::ACTION_CONTROL_STOP;     // 'control.stop'
            case 'restart':                return Permission::ACTION_CONTROL_RESTART;  // 'control.restart'
        }
        return '__invalid';   // ★ 未知 signal：权限名无法匹配，授权层必然拒绝
    }

    public function rules(): array {
        return [
            // ★ 白名单：只允许这 4 个字符串
            'signal' => 'required|string|in:start,stop,restart,kill',
        ];
    }
}
```

**两层保护（代码可确认）**：
1. **Laravel Validation `in:`**：请求到达控制器前，若 `signal` 不在白名单内，直接返回 422（ValidationException）。
2. **`permission()` 映射到 `__invalid`**：即使绕过了 validation（理论上不可能），授权层也会因为找不到 `__invalid` 权限而返回 403。

### 4.2 控制台命令白名单

**验证类**：[SendCommandRequest.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Http/Requests/Api/Client/Servers/SendCommandRequest.php)

```php
class SendCommandRequest extends ClientApiRequest
{
    public function permission(): string {
        return Permission::ACTION_CONTROL_CONSOLE;  // 'control.console'
    }

    public function rules(): array {
        return [
            'command' => 'required|string|min:1',  // ★ 注意：无命令内容白名单！
        ];
    }
}
```

**关键区别（代码可确认）**：
- Panel 对命令**不做白名单过滤**。原因：游戏服控制台命令集千差万别（Minecraft `/op`、Source `sm_kick`、Rust `server.save` 等），Panel 无法穷举。
- **权限门槛**：只需拥有 `control.console` 权限即可发送任意命令。
- **命令过滤职责（【推断】）**：【推断】由 Wings 或 Egg 配置的 `config.yml` disallow 列表负责，Panel 侧无对应实现。

### 4.3 调度任务白名单

**验证类**：[StoreTaskRequest.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Http/Requests/Api/Client/Servers/Schedules/StoreTaskRequest.php)

```php
public function rules(): array {
    return [
        // ★ 调度任务 action 白名单
        'action'              => 'required|in:command,power,backup',
        'payload'             => 'required_unless:action,backup|string|nullable',
        'time_offset'         => 'required|numeric|min:0|max:900',  // 最大延迟 15 分钟
        'continue_on_failure' => 'sometimes|required|boolean',
    ];
}
```

**代码可确认**：`action=power` 时，`payload` 的值（应为 `start|stop|restart|kill`）在 Panel 侧**不再二次校验白名单**，而是在 [RunTaskJob.php#L62-L63](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Jobs/Schedule/RunTaskJob.php#L62-L63) 直接交给 `DaemonPowerRepository::send()` 转发至 Wings。

---

## 5. Panel 后端：权限鉴权模型

### 5.1 请求层权限解析（ClientApiRequest::authorize）

**基类**：[ClientApiRequest.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Http/Requests/Api/Client/ClientApiRequest.php)

```php
class ClientApiRequest extends ApplicationApiRequest
{
    public function authorize(): bool {
        // 若子类实现了 permission() 方法（或实现了 ClientPermissionsRequest 接口），用 Gate 校验
        if ($this instanceof ClientPermissionsRequest || method_exists($this, 'permission')) {
            $server = $this->route()->parameter('server');

            if ($server instanceof Server) {
                // 调用 ServerPolicy，传入 permission 字符串
                return $this->user()->can($this->permission(), $server);
            }
            return false;  // 路由参数里找不到 Server → 拒绝
        }
        return true;  // 未定义 permission() 的请求默认通过
    }
}
```

**权限检查失败的后果（代码可确认）**：Laravel Gate 返回 false → 抛出 `AuthorizationException` → 渲染为 HTTP 403。

### 5.2 权限判定策略（ServerPolicy）

**策略类**：[ServerPolicy.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Policies/ServerPolicy.php)

```php
class ServerPolicy
{
    // before() 优先级最高，短路判断
    public function before(User $user, string $ability, Server $server): ?bool {
        if ($user->root_admin || $server->owner_id === $user->id) {
            return true;   // ★ 管理员 / 服务器所有者：无条件放行
        }
        return $this->checkPermission($user, $server, $ability);
    }

    protected function checkPermission(User $user, Server $server, string $permission): bool {
        $subuser = $server->subusers->where('user_id', $user->id)->first();
        if (!$subuser || empty($permission)) return false;
        // 子用户权限是数据库中存储的数组：['control.start', 'control.stop', ...]
        return in_array($permission, $subuser->permissions);
    }

    // __call 魔术方法：避免 Laravel 因 policy 方法不存在而跳过 before()
    public function __call(string $name, mixed $arguments) {}
}
```

**权限常量汇总（代码可确认，见 [Permission.php#L18-L22](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Models/Permission.php#L18-L22)）**：

| 常量 | 值 | 含义 |
|------|-----|------|
| `ACTION_WEBSOCKET_CONNECT` | `websocket.connect` | 建立 WebSocket（查看控制台的基础权限） |
| `ACTION_CONTROL_CONSOLE` | `control.console` | 发送命令到控制台 |
| `ACTION_CONTROL_START` | `control.start` | Start 服务器 |
| `ACTION_CONTROL_STOP` | `control.stop` | Stop / Kill 服务器 |
| `ACTION_CONTROL_RESTART` | `control.restart` | Restart 服务器 |

### 5.3 服务器状态门禁（AuthenticateServerAccess）

**中间件**：[AuthenticateServerAccess.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Http/Middleware/Api/Client/Server/AuthenticateServerAccess.php)

```php
public function handle(Request $request, \Closure $next): mixed
{
    $user = $request->user();
    $server = $request->route()->parameter('server');

    // 1. 身份归属校验：owner / root_admin / subuser，否则 404（用 404 替代 403，避免被探测）
    if ($user->id !== $server->owner_id && !$user->root_admin) {
        if (!$server->subusers->contains('user_id', $user->id)) {
            throw new NotFoundHttpException();
        }
    }

    // 2. 状态冲突校验：Server::v`validateCurrentState()`
    try {
        $server->v`validateCurrentState()`;
    } catch (ServerStateConflictException $exception) {
        // 例外 1：view endpoint (GET /server) 允许查看状态
        if (!$request->routeIs('api:client:server.view')) {
            // 例外 2：suspended / node_maintenance 允许看 /resources
            if (($server->isSuspended() || $server->node->isUnderMaintenance())
                && !$request->routeIs('api:client:server.resources')) {
                throw $exception;
            }
            // 例外 3：管理员可连 WebSocket（api:client:server.ws）
            if (!$user->root_admin || !$request->routeIs($this->except)) {
                throw $exception;   // HTTP 409 Conflict
            }
        }
    }
    return $next($request);
}
```

**`v`validateCurrentState()`` 判定逻辑**（[Server.php#L390-L401](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Models/Server.php#L390-L401)，代码可确认）：

```php
public function v`validateCurrentState()`
{
    if (
        $this->isSuspended()              // status = 'suspended'
        || $this->node->isUnderMaintenance()  // node.maintenance_mode = true
        || !$this->isInstalled()          // 未安装完成
        || $this->status === self::STATUS_RESTORING_BACKUP
        || !is_null($this->transfer)      // 正在跨节点迁移
    ) {
        throw new ServerStateConflictException($this);  // 409 Conflict
    }
}
```

**结论（代码可确认）**：当服务器处于 suspended / 维护中 / 安装中 / 还原备份 / 迁移中时，**`/power` 和 `/command` 端点均拒绝请求（409）**。WebSocket 是唯一例外（仅管理员可连接，用于查看迁移/安装日志）。

---

## 6. Panel → Wings：HTTP 调用链（Panel 代码可确认）

### 6.1 DaemonRepository 基类：Guzzle HTTP 客户端

**基类**：[DaemonRepository.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Repositories/Wings/DaemonRepository.php)

```php
abstract class DaemonRepository
{
    public function getHttpClient(array $headers = []): Client {
        Assert::isInstanceOf($this->node, Node::class);

        return new Client([
            'verify'          => $this->app->environment('production'),   // 生产环境强制 HTTPS 证书验证
            'base_uri'        => $this->node->getConnectionAddress(),
            'timeout'         => config('pterodactyl.guzzle.timeout'),         // 默认 15 秒
            'connect_timeout' => config('pterodactyl.guzzle.connect_timeout'), // 默认 5 秒
            'headers' => array_merge($headers, [
                // ★ 使用 Node 的解密后密钥（AES-256-CBC 存储，解密后作为 Bearer Token）
                'Authorization' => 'Bearer ' . $this->node->getDecryptedKey(),
                'Accept'        => 'application/json',
                'Content-Type'  => 'application/json',
            ]),
        ]);
    }
}
```

**超时配置（[config/pterodactyl.php#L79-L82](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/config/pterodactyl.php#L79-L82)，代码可确认）**：
- `GUZZLE_TIMEOUT=15`：总请求超时 15 秒
- `GUZZLE_CONNECT_TIMEOUT=5`：TCP 连接超时 5 秒

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
        throw new DaemonConnectionException($exception);  // ★ 统一包装为面板可读异常
    }
}
```

**代码可确认**：请求 Wings Endpoint = `POST /api/servers/{server.uuid}/power`，body = `{"action": "<signal>"}`。

### 6.3 DaemonCommandRepository：控制台命令下发

**文件**：[DaemonCommandRepository.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Repositories/Wings/DaemonCommandRepository.php)

```php
public function send(array|string $command): ResponseInterface
{
    try {
        return $this->getHttpClient()->post(
            sprintf('/api/servers/%s/commands', $this->server->uuid),
            [
                // ★ 支持批量：总是包装为 commands 数组
                'json' => ['commands' => is_array($command) ? $command : [$command]],
            ]
        );
    } catch (TransferException $exception) {
        throw new DaemonConnectionException($exception);
    }
}
```

**代码可确认**：请求 Wings Endpoint = `POST /api/servers/{server.uuid}/commands`，body = `{"commands": ["cmd"]}`（即使是单条命令也包装为数组）。

### 6.4 Activity 审计日志埋点

**PowerController**（[PowerController.php#L31](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Http/Controllers/Api/Client/Servers/PowerController.php#L31)）：
```php
Activity::event(strtolower("server:power.{$request->input('signal')}"))->log();
// 可能的事件名: server:power.start / server:power.stop / server:power.restart / server:power.kill
```

**CommandController**（[CommandController.php#L46](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Http/Controllers/Api/Client/Servers/CommandController.php#L46)）：
```php
Activity::event('server:console.command')
    ->property('command', $request->input('command'))  // ★ 记录具体命令内容！
    ->log();
```

**代码可确认的关键点**：
- 两处 `Activity::...->log()` 均位于 Wings 调用之后。若 Wings 调用抛出异常，PHP 会中断执行，Activity 不会被写入数据库。
- 即：**Wings 调用成功后才记录审计日志**，避免假阳性记录。

---

## 7. Panel → Wings：WebSocket 通道（JWT 鉴权）

### 7.1 JWT 签发服务（NodeJWTService）

由 [WebsocketController.php#L55-L62](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Http/Controllers/Api/Client/Servers/WebsocketController.php#L55-L62) 调用：

```php
$token = $this->jwtService
    ->setExpiresAt(CarbonImmutable::now()->addMinutes(10))   // 10 分钟过期
    ->setUser($request->user())
    ->setClaims([
        'server_uuid'  => $server->uuid,
        'permissions'  => $permissions,   // ★ 用户在该服务器上的全部权限列表，嵌入 JWT
    ])
    ->handle($node, $user->id . $server->uuid);
```

**代码可确认的 JWT Claims 结构**（见 [NodeJWTService.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Services/Nodes/NodeJWTService.php)）：

```
Header:
  alg: HS256               // 对称加密，使用 Node 的密钥签名

Payload (Claims):
  iss: config('app.url')   // 签发者 = APP_URL
  aud: node 连接地址       // 接收方
  iat: 签发时间戳
  nbf: iat - 5min          // 允许 5 分钟时钟偏差
  exp: iat + 10min         // 过期时间
  sub: 用户标识
  user_uuid: 用户 UUID
  user_id: 用户自增 ID (兼容字段)
  server_uuid: 服务器 UUID
  permissions: [ "websocket.connect", "control.start", ... ]  // ★ Panel 写入 JWT，【推断】Wings 据此独立鉴权
  unique_id: Str::random() // 防重放
  jti: md5(user_id + server_uuid)
```

**代码可确认的 Websocket 权限门槛**：
- 在签发 JWT 前，[WebsocketController.php#L36-L38](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Http/Controllers/Api/Client/Servers/WebsocketController.php#L36-L38) 首先检查 `Permission::ACTION_WEBSOCKET_CONNECT`，无权限直接抛 403。
- Panel 将用户在该服务器上的全部权限列表通过 `permissions` claim 写入 JWT（代码可确认，见 [WebsocketController.php#L58-L60](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Http/Controllers/Api/Client/Servers/WebsocketController.php#L58-L60)）。【推断】Wings 收到 WebSocket 事件（如 `send command`、`set state`）时，会基于 JWT 中的 `permissions` 列表做独立鉴权，无需回查 Panel。

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
                // Wings → 前端: {event: string, args: string[]}
                args ? this.emit(event, ...args) : this.emit(event);
            },
            onopen: () => this.authenticate(),  // 连接打开立即发 auth
        });
    }

    // ★ 前端 → Wings 发送协议: { event: string, args: string[] }
    send(event: string, payload?: string | string[]) {
        this.socket?.json({ event, args: Array.isArray(payload) ? payload : [payload] });
    }
}
```

### 7.3 事件枚举（前端发出 & 接收）

定义于 [events.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/resources/scripts/components/server/events.ts)：

```ts
export enum SocketRequest {  // 前端 → Wings（发出）
    SEND_LOGS = 'send logs',   // 请求 Wings 推送历史控制台缓冲
    SEND_STATS = 'send stats', // 请求立即发送一次统计数据
    SET_STATE = 'set state',   // ★ 电源动作（注: Console.tsx 中的 "send command" 未列入此枚举，直接用字符串）
}

export enum SocketEvent {     // Wings → 前端（接收）
    DAEMON_MESSAGE = 'daemon message',
    DAEMON_ERROR = 'daemon error',
    CONSOLE_OUTPUT = 'console output',
    STATUS = 'status',
    STATS = 'stats',
    // ... 安装 / 迁移 / 备份 相关
}
```

---

## 8. Wings 侧（标注：以下均为推断）

> ⚠️ **重要声明**：本代码库仅包含 Panel，不含 Wings（Go 守护进程）。以下内容基于 Panel 代码中与 Wings 的交互协议（URL、body 结构、WebSocket 事件名）推导，非源码实证。

### 8.1 【推断】REST Endpoint 约定

基于 Panel 发出的 HTTP 调用结构推断：

| Method | Path | Body | 【推断】Wings 行为 |
|--------|------|------|--------------|
| POST | `/api/servers/{uuid}/power` | `{"action":"start\|stop\|restart\|kill"}` | 【推断】立即返回 204 No Content；内部异步改变进程状态 |
| POST | `/api/servers/{uuid}/commands` | `{"commands":["cmd1","cmd2"]}` | 【推断】要求服务器处于 running 状态，否则返回 502 Bad Gateway；否则立即 204 |
| GET | `/api/servers/{uuid}` | — | 【推断】返回 `{"state": "running\|offline\|...", ...}`（Panel 调度 `only_when_online` 检查依赖此接口） |
| GET | `/api/servers/{uuid}/ws` | Upgrade | WebSocket 握手 |

### 8.2 【推断】WebSocket Event 约定

基于前端 WebSocket 代码推断：

**Client → Wings（需携带 JWT 且 JWT claims 含对应权限）**：

| Event | Args | 所需 JWT permission | 说明 |
|-------|------|-------------------|------|
| `auth` | `[jwt]` | — | 连接后第一条消息 |
| `send logs` | `[]` | `websocket.connect` | 请求 Wings 推送历史控制台缓冲 |
| `send stats` | `[]` | `websocket.connect` | 请求立即发送一次统计 |
| `set state` | `["start"\|"stop"\|"restart"\|"kill"]` | 对应 `control.start\|stop\|restart` | 等同 REST /power |
| `send command` | `["/op Notch"]` | `control.console` | 等同 REST /commands（单条命令） |

**Wings → Client（前端监听）**：

| Event | Args | 【推断】触发时机 |
|-------|------|----------------|
| `auth success` | — | JWT 校验通过 |
| `token expiring` | — | JWT 距过期 < 3 分钟时 |
| `token expired` | — | JWT 已过期 |
| `jwt error` | `[msg]` | JWT 校验失败 |
| `status` | `["starting"]` | 电源状态迁移时 |
| `console output` | `["[12:34] ..."]` | 游戏服 stdout 有新行时 |
| `stats` | `[{"cpu":10.5,"memory_bytes":...}]` | 【推断】统计心跳（间隔由 Wings 决定） |

### 8.3 【推断】并发安全与节流

基于 Panel 的设计模式推断 Wings 侧可能采取的机制：
- 电源动作串行化：【推断】用互斥锁避免 Start 与 Kill 同时到达导致进程孤儿
- 控制台命令限频：【推断】使用令牌桶限频，防止通过 WebSocket `send command` 刷屏
- 命令黑名单：【推断】读取 Egg 的 `config.yml` 中的 disallow 列表，拦截危险命令

这些机制的具体实现（结构体、变量名、调用顺序）**无法从 Panel 代码确认**。

---

## 9. 调度系统：计划任务中的电源/命令

### 9.1 ProcessScheduleService：事务入队

**服务**：[ProcessScheduleService.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Services/Schedules/ProcessScheduleService.php)

```php
public function handle(Schedule $schedule, bool $now = false): void
{
    $task = $schedule->tasks()->orderBy('sequence_id')->first();

    // ★ 数据库事务：标记 is_processing + is_queued，避免并发调度
    $this->connection->transaction(function () use ($schedule, $task) {
        $schedule->forceFill([
            'is_processing' => true,
            'next_run_at'   => $schedule->getNextRunDate(),  // 提前计算下次运行
        ])->saveOrFail();
        $task->update(['is_queued' => true]);
    });

    $job = new RunTaskJob($task, $now);
    if ($schedule->only_when_online) {
        // 调 Wings GET /api/servers/{uuid} 查 state
        try {
            $details = $this->serverRepository->setServer($schedule->server)->getDetails();
            $state = $details['state'] ?? 'offline';
            if (in_array($state, ['offline', 'stopping'])) {
                $job->failed();   // ★ 默默标记完成，不抛异常
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
    // → 入队，延迟 time_offset 秒
    if (!$now) {
        $this->dispatcher->dispatch($job->delay($task->time_offset));
    } else {
        $this->dispatcher->dispatchNow($job);  // 立即同步执行
    }
}
```

### 9.2 RunTaskJob：按序执行与失败降级

**任务**：[RunTaskJob.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Jobs/Schedule/RunTaskJob.php)

```php
public function handle(
    DaemonCommandRepository $commandRepository,
    InitiateBackupService $backupService,
    DaemonPowerRepository $powerRepository,
) {
    // 防御性检查：调度入队后，服务器状态被改了（suspended 等）
    if (!is_null($server->status)) { $this->failed(); return; }

    try {
        switch ($this->task->action) {
            case Task::ACTION_POWER:   // "power"
                $powerRepository->setServer($server)->send($this->task->payload); break;
            case Task::ACTION_COMMAND: // "command"
                $commandRepository->setServer($server)->send($this->task->payload); break;
            case Task::ACTION_BACKUP:  // "backup"
                $backupService->...; break;
            default:
                throw new \InvalidArgumentException('Invalid task action: ' . $this->task->action);
        }
    } catch (\Exception $exception) {
        // ★ 唯一失败降级：continue_on_failure=true + DaemonConnectionException（Wings 连不上）
        if (!($this->task->continue_on_failure && $exception instanceof DaemonConnectionException)) {
            throw $exception;   // 其他异常：任务链中断
        }
    }

    $this->markTaskNotQueued();
    $this->queueNextTask();   // → 取下一个 sequence_id，delay 后入队
}

// 失败钩子：释放 is_processing 锁，避免 Schedule 永久卡死
public function failed(?\Exception $exception = null) {
    $this->markTaskNotQueued();
    $this->markScheduleComplete();
}
```

**任务链执行模型（代码可确认）**：
```
Schedule (is_processing=true)
  ├─ Task #1 (sequence_id=1, time_offset=0)
  │    └─ success → markTaskNotQueued → dispatch Task #2
  │    └─ DaemonConnectionException + continue_on_failure=true → 静默跳过 → dispatch Task #2
  │    └─ 其他异常 → failed() 钩子 → is_processing=false → 调度链中断
  ├─ Task #2 (sequence_id=2, time_offset=30)
  ...
  └─ 最后一个 Task 完成 → markScheduleComplete(is_processing=false, last_run_at=now)
```

---

## 10. 失败处理与状态回滚（Panel 代码可确认）

### 10.1 DaemonConnectionException：Wings 通讯异常封装

**异常类**：[DaemonConnectionException.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Exceptions/Http/Connection/DaemonConnectionException.php)

```php
public function __construct(GuzzleException $previous, bool $useStatusCode = true)
{
    $response = method_exists($previous, 'getResponse') ? $previous->getResponse() : null;
    $this->requestId = $response?->getHeaderLine('X-Request-Id'); // ★ 便于 Wings 侧日志排查

    // 状态码映射：如果 2XX 却进入了异常（Wings panic 已写 header 后 crash），升级为 502 Bad Gateway
    if ($useStatusCode) {
        $this->statusCode = is_null($response) ? 504 : $response->getStatusCode();
        if ($this->statusCode < 400) $this->statusCode = 502;
    }

    // 日志分级：5XX（非 504）= ERROR 级别，其余 = WARNING 级别
    $level = $this->statusCode >= 500 && $this->statusCode !== 504
        ? DisplayException::LEVEL_ERROR
        : DisplayException::LEVEL_WARNING;

    parent::__construct($message, $previous, $level);
}

// 自动写 Laravel 日志，附带 request_id
public function report() {
    Log::{$this->getErrorLevel()}($this->getPrevious(), ['request_id' => $this->requestId]);
}
```

**代码可确认的要点**：
- 无响应（TCP 连接失败、超时）→ 状态码 504，提示 "Could not establish a connection..."
- 非 5XX 错误（如 400/401/403/422）→ 尝试从 Wings 响应体解析 `error` 字段，返回给前端
- `report()` 会自动调用，将异常堆栈 + `request_id` 写入 Laravel Log

### 10.2 CommandController：502 → 用户可读错误

[CommandController.php#L32-L44](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Http/Controllers/Api/Client/Servers/CommandController.php#L32-L44)：

```php
try {
    $this->repository->setServer($server)->send($request->input('command'));
} catch (DaemonConnectionException $exception) {
    $previous = $exception->getPrevious();
    if ($previous instanceof BadResponseException) {
        // ★ Wings 返回 502：游戏服进程未运行（容器未启动 / stdin 管道关闭）
        if ($previous->getResponse()->getStatusCode() === Response::HTTP_BAD_GATEWAY) {
            throw new HttpException(
                Response::HTTP_BAD_GATEWAY,
                'Server must be online in order to send commands.',
                $exception
            );
        }
    }
    throw $exception;  // 其他错误原样抛（500、504、4XX 等）
}
```

**PowerController 无此特殊处理**：[PowerController.php#L27-L29](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Http/Controllers/Api/Client/Servers/PowerController.php#L27-L29) 中 `DaemonPowerRepository::send()` 的异常直接冒泡，无 catch。

### 10.3 电源动作失败：无数据库状态回滚

**代码可确认的事实**：
1. Panel `Server` 模型的 `status` 字段仅用于以下值（非运行态）：`null`（正常）/ `suspended` / `installing` / `install_failed` / `restoring_backup` 等（见 [Server.php 模型常量](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Models/Server.php)）。**不存在** `running`、`offline`、`stopping`、`starting` 这类电源状态的数据库字段。
2. 电源状态完全由 Wings 通过 WebSocket 的 `status` 事件推送给前端，Panel 数据库不做任何持久化。
3. 因此：即使 `/power` 请求 Wings 失败，Panel 也**无数据库状态需要回滚**——没有"记录了 running 但实际没启动"的不一致。

**状态一致性模型**：最终一致（Eventual Consistency），权威数据源在 Wings，Panel 仅转发指令 + 展示 Wings 推送的状态。

### 10.4 调度任务失败：continue_on_failure + failed() 释放锁

| 场景（代码可确认） | 失败行为 |
|------------------|---------|
| DaemonConnectionException（Wings 连不上）+ `continue_on_failure=true` | 静默跳过，继续执行后续 Task |
| DaemonConnectionException + `continue_on_failure=false` | 抛异常 → Laravel Queue 调用 `failed()` 钩子 → 标记 `is_processing=false` → 链中断 |
| 任何其他 Exception（Validation / Authorization / InvalidArgumentException） | 同上：`failed()` 释放锁 + 链中断 |
| `only_when_online=true` + 服务器 offline | 默默标记完成，不抛异常 |
| 入队后服务器被 suspended（`!is_null($server->status)`） | 调用 `failed()` 释放锁，不抛异常 |

---

## 11. 客服视角：Power Action 语义对照表

> 针对客服把按钮当命令行的场景，整理"按钮按下 → Panel 代码行为 →（【推断】Wings 行为）→ 玩家体感"：

| 按钮 | 实际发送值（Panel 代码可确认） | 【推断】Wings 行为 | 【推断】玩家体感 | 风险（客服应知） |
|------|-----------------------------|-----------------|---------|----------------|
| **Start** | `"start"`（WebSocket `set state` 或 REST `signal`） | 【推断】启动容器并执行启动命令 | 【推断】服务器开始启动，可进入游戏的时间取决于游戏服类型 | 正常操作，几乎无风险 |
| **Stop** | `"stop"` | 【推断】执行优雅关闭流程，具体行为由 Wings 决定 | 【推断】玩家会收到服务器关闭相关提示，地图通常会正常存档 | **优雅关闭，应默认使用** |
| **Restart** | `"restart"` | 【推断】先执行关闭流程，容器退出后自动启动 | 【推断】玩家会被断开连接，需等待重启完成后重连 | 会导致玩家连接中断 |
| **Kill** | `"kill"` | 【推断】执行强制终止流程，不给游戏服预留处理时间 | 【推断】玩家会瞬间失去连接，可能存在未存档数据 | ⚠️ 建议仅在 Stop 无响应时使用；用完后建议检查存档完整性再开服 |

**客服应传达的操作原则**：
1. 日常维护一律用 **Stop**，不用 Kill
2. 当 Stop 后控制台长时间无任何输出时，可升级到 Kill（具体等待时长建议参考所运行游戏服的常规关闭耗时，本仓库代码未定义阈值）
3. Kill 之后建议检查游戏服最新存档完整性，再执行 Start
4. **按钮 = 电源动作，不是命令输入框**。要执行游戏内命令（`/op`, `whitelist add` 等），使用 Console 下方命令输入框

---

## 12. 关键文件索引（Panel 侧可确认）

### 12.1 前端 (React/TypeScript)

| 文件 | 职责 |
|------|------|
| [PowerButtons.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/resources/scripts/components/server/console/PowerButtons.tsx) | 电源按钮 UI + WebSocket "set state" 发送 |
| [Console.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/resources/scripts/components/server/console/Console.tsx) | xterm.js 终端 + WebSocket "send command" 发送 + 命令历史 |
| [WebsocketHandler.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/resources/scripts/components/server/WebsocketHandler.tsx) | WebSocket 生命周期管理 + JWT 自动刷新 |
| [Websocket.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/resources/scripts/plugins/Websocket.ts) | Sockette 封装 + 协议序列化 |
| [Can.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/resources/scripts/components/elements/Can.tsx) | 基于权限的条件渲染组件 |
| [events.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/resources/scripts/components/server/events.ts) | WebSocket 事件名枚举 |
| [ServerConsoleContainer.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/resources/scripts/components/server/console/ServerConsoleContainer.tsx) | 控制台页面容器 + `PowerAction` 类型定义 |
| [getWebsocketToken.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/resources/scripts/api/server/getWebsocketToken.ts) | REST 请求 WebSocket JWT |
| [http.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/resources/scripts/api/http.ts) | Axios 实例配置（withCredentials=true, 20s 超时） |

### 12.2 Panel 后端 (PHP Laravel)

#### 路由 & 中间件

| 文件 | 职责 |
|------|------|
| [routes/api-client.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/routes/api-client.php) | `/api/client/servers/{server}/*` 路由定义（已核对：/command→CommandController, /power→PowerController） |
| [Http/Kernel.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Http/Kernel.php) | 中间件栈定义 |
| [Providers/RouteServiceProvider.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Providers/RouteServiceProvider.php) | 路由挂载 + 全局限流器（api.client: 256/min）+ ResourceLimit::boot() |
| [Enum/ResourceLimit.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Enum/ResourceLimit.php) | 资源级限流器（Websocket 5/min/server） |
| [Middleware/.../AuthenticateServerAccess.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Http/Middleware/Api/Client/Server/AuthenticateServerAccess.php) | 服务器归属 + 状态冲突门禁（409）|

#### 控制器

| 文件 | 职责 |
|------|------|
| [PowerController.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Http/Controllers/Api/Client/Servers/PowerController.php) | `POST /power` → DaemonPowerRepository + Activity Log |
| [CommandController.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Http/Controllers/Api/Client/Servers/CommandController.php) | `POST /command` → DaemonCommandRepository + 502 转译 + Activity Log |
| [WebsocketController.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Http/Controllers/Api/Client/Servers/WebsocketController.php) | `GET /websocket` → 签发 10min JWT（含 permissions 列表）+ 返回 WS URL |

#### 请求验证（白名单）

| 文件 | 职责 |
|------|------|
| [SendPowerRequest.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Http/Requests/Api/Client/Servers/SendPowerRequest.php) | `signal in:start,stop,restart,kill` + 按 signal 映射权限 |
| [SendCommandRequest.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Http/Requests/Api/Client/Servers/SendCommandRequest.php) | `command required\|min:1` + `control.console` 权限 |
| [ClientApiRequest.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Http/Requests/Api/Client/ClientApiRequest.php) | `authorize()` → `user()->can(permission(), server)` |
| [StoreTaskRequest.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Http/Requests/Api/Client/Servers/Schedules/StoreTaskRequest.php) | 调度任务 `action in:command,power,backup` + `time_offset max:900` |

#### Wings 通信层

| 文件 | 职责 |
|------|------|
| [DaemonRepository.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Repositories/Wings/DaemonRepository.php) | Guzzle 客户端基类（Node Bearer Token + 超时 15s/5s） |
| [DaemonPowerRepository.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Repositories/Wings/DaemonPowerRepository.php) | `POST /api/servers/{uuid}/power` body: `{"action": "..."}` |
| [DaemonCommandRepository.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Repositories/Wings/DaemonCommandRepository.php) | `POST /api/servers/{uuid}/commands` body: `{"commands": [...]}` |
| [DaemonServerRepository.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Repositories/Wings/DaemonServerRepository.php) | `GET /api/servers/{uuid}`（调度 only_when_online 检查用）|
| [DaemonConnectionException.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Exceptions/Http/Connection/DaemonConnectionException.php) | Wings 异常封装 + X-Request-Id + 自动日志分级 |

#### 权限 & 模型

| 文件 | 职责 |
|------|------|
| [Permission.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Models/Permission.php) | 权限常量（`control.*` / `websocket.connect` 等）+ 权限描述 |
| [Policies/ServerPolicy.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Policies/ServerPolicy.php) | Gate 判定：root_admin/owner 放行，子用户查 permissions 数组 |
| [Models/Server.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Models/Server.php) | `v`validateCurrentState()``（suspended/维护/安装/还原/迁移 → 409）|
| [Services/Nodes/NodeJWTService.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Services/Nodes/NodeJWTService.php) | HS256 JWT 签发，嵌入 server_uuid + permissions + user_uuid |

#### 调度系统

| 文件 | 职责 |
|------|------|
| [Services/Schedules/ProcessScheduleService.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Services/Schedules/ProcessScheduleService.php) | DB 事务入队 + `only_when_online` 检查（调 Wings GET /api/servers/{uuid}）+ 延迟分发 |
| [Jobs/Schedule/RunTaskJob.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Jobs/Schedule/RunTaskJob.php) | 按 sequence_id 顺序执行 + `continue_on_failure` 降级 + `failed()` 释放锁 |
| [Models/Task.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Models/Task.php) | Task::ACTION_POWER/COMMAND/BACKUP 常量 |
| [.../Schedules/StoreTaskRequest.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Http/Requests/Api/Client/Servers/Schedules/StoreTaskRequest.php) | action 白名单 + payload/time_offset 校验 |

#### 异常处理 & 配置

| 文件 | 职责 |
|------|------|
| [DaemonConnectionException.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Exceptions/Http/Connection/DaemonConnectionException.php) | Wings 通讯异常封装（含 X-Request-Id、状态码重映射、自动报告日志） |
| [config/http.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/config/http.php) | API 限流阈值（client 256/min, application 256/min） |
| [config/pterodactyl.php](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/config/pterodactyl.php) | Guzzle 超时配置（`timeout=15s`, `connect_timeout=5s`） |

---

## 附：客服 FAQ 速查

**Q1: 点 Stop 按钮等了好久都没反应怎么办？**
A: Stop 是"优雅关闭"——【推断】Wings 会向游戏服发送停止指令，让其完成存档+踢玩家后再关容器。若控制台长时间无任何输出，说明游戏服进程可能无响应，可以升级到 **Kill** 按钮（会弹出二次确认），但存在未存档丢失风险。具体等待时长建议参考游戏服的常规停止耗时，本仓库代码未定义阈值。

**Q2: Start 按钮灰的，点不了？**
A: 代码可确认：[PowerButtons.tsx#L54](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/resources/scripts/components/server/console/PowerButtons.tsx#L54) `disabled={status !== 'offline'}`。只有当服务器状态是 **offline** 时 Start 才可用。检查 Console 页面左上角的状态标签，如果是 `starting`/
`running`/`stopping`，Start 会被前端禁用，避免并发启动。

**Q3: 用命令输入框输了 "stop" 回车和点 Stop 按钮一样吗？**
A: **完全不一样！**
- 命令输入框（[Console.tsx#L176](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/resources/scripts/components/server/console/Console.tsx#L176)）走的是 WebSocket `send command` 事件，内容作为游戏服**控制台命令**直接写入 stdin。大多数游戏里 `"stop"` 并不是合法指令（Minecraft 需要 `/stop` 带斜杠）。
- Stop **按钮**走的是 `set state` → `DaemonPowerRepository` → Wings `POST /power`，【推断】由 Wings 负责执行优雅关闭流程。**推荐用按钮**。

**Q4: Kill 和 Stop 到底有啥区别？**
A: Panel 代码可确认二者发出的值不同（`"kill"` vs `"stop"`，见 [PowerButtons.tsx#L29](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/resources/scripts/components/server/console/PowerButtons.tsx#L29) 和 [DaemonPowerRepository.php#L29](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Repositories/Wings/DaemonPowerRepository.php#L29)）。【推断】Stop = 优雅关闭流程（通常会给游戏服机会完成自身清理）；Kill = 强制终止（不等候游戏服自身处理）。Kill 建议只在 Stop 无响应时使用。用完 Kill 后建议检查最新存档完整性再开服。

**Q5: 服务器被暂停（suspended）了，能发命令吗？**
A: Panel 代码可确认：[AuthenticateServerAccess.php#L50-L59](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/app/Http/Middleware/Api/Client/Server/AuthenticateServerAccess.php#L50-L59) 中的 `validateCurrentState()` 会拒绝 `/power` 和 `/command` 请求（HTTP 409 Conflict）。管理员即使能连上 WebSocket 也只能看日志，无法执行操作。先联系主机商解除暂停。

**Q6: 命令输入框没了？**
A: 检查当前账号有没有 `control.console` 权限。代码可确认：[Console.tsx#L66](file:///d:/fz/0508-3/solo-dogfeeding/code/208-panel/resources/scripts/components/server/console/Console.tsx#L66) `canSendCommands` 由 `usePermissions(['control.console'])` 返回，子用户由所有者在 **Users** 标签页分配。没有此权限的账号不会渲染命令输入框；【推断】即使手动构造 WebSocket `send command` 事件，Wings 侧也会因为 JWT claims 不含 control.console 而被拦截。
