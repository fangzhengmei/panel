# API Key 流程梳理：密钥签发 → 范围校验 → 频率限制

## 一、全局概览

本项目（Pterodactyl Panel）存在 **两套独立的 API 接口**，共享同一张 `api_keys` 数据表和同一个 Sanctum 认证管道，但权限模型完全不同：

| 维度 | Application API（管理接口） | Client API（客户端接口） |
|------|---------------------------|------------------------|
| 前缀 | `/api/application` | `/api/client` |
| Key 类型 | `TYPE_APPLICATION = 2`（已标记 @deprecated） | `TYPE_ACCOUNT = 1` |
| Key 前缀 | `ptla_` | `ptlc_` |
| 权限模型 | `AdminAcl` — 列级 `r_{resource}` 位运算（NONE/READ/WRITE） | `Permission` + `ServerPolicy` — 子用户权限常量 |
| 签发入口 | Admin Web 后台（`AdminController@store`） | Client API 自助（`ApiKeyController@store`） |
| 管理员校验 | `AuthenticateApplicationUser`（必须 `root_admin`） | `RequireClientApiKey`（禁止 Application Key 闯入） |

两套接口在 `RouteServiceProvider` 中被挂载到 **同一个** `api` 中间件组上，共享认证与 IP 校验逻辑，之后才分流到各自专属的中间件栈。

---

## 二、密钥签发（Key Issuance）

### 2.1 统一数据模型 — `ApiKey`

文件：`app/Models/ApiKey.php`

```
api_keys 表核心字段：
├── id
├── user_id          → 归属用户
├── key_type         → 0=NONE, 1=ACCOUNT, 2=APPLICATION(已废弃), 3/4=DAEMON(已废弃)
├── identifier       → 前 16 字符，明文存储，用于查表（如 ptlc_xxxxxxxxxxxxxx）
├── token            → 后 32 字符，AES 加密存储
├── allowed_ips      → JSON 数组，IP/CIDR 白名单
├── memo             → 描述（最长 500 字）
├── last_used_at     → 最后使用时间
├── expires_at       → 过期时间
├── r_servers        → AdminAcl 资源权限（仅 TYPE_APPLICATION 使用）
├── r_nodes
├── r_allocations
├── r_users
├── r_locations
├── r_nests
├── r_eggs
├── r_database_hosts
└── r_server_databases
```

关键设计：
- `identifier` 是明文的前缀+随机字符串，用于快速定位记录
- `token` 是加密存储的后半部分，鉴权时解密比对
- 完整的 API Key = `identifier` + `token`（明文仅在签发瞬间返回一次）

### 2.2 查找令牌 — `ApiKey::findToken()`

文件：`app/Models/ApiKey.php:200-210`

```php
public static function findToken(string $token): ?self
{
    $identifier = substr($token, 0, self::IDENTIFIER_LENGTH);  // 取前 16 字符
    $model = static::where('identifier', $identifier)->first(); // 用 identifier 查表
    if (!is_null($model) && decrypt($model->token) === substr($token, strlen($identifier))) {
        return $model;  // 解密 token 后与请求中的后半段比对
    }
    return null;
}
```

这是 Sanctum 底层调用的入口。Sanctum 在 `auth:sanctum` 守卫中拿到 Bearer Token 后，调用此方法完成"查表 → 解密 → 比对"三步。

### 2.3 Client API Key 签发

文件：`app/Models/Traits/HasAccessTokens.php:30-43`

```php
public function createToken(?string $memo, ?array $ips): NewAccessToken
{
    $token = $this->tokens()->forceCreate([
        'user_id'    => $this->id,
        'key_type'   => ApiKey::TYPE_ACCOUNT,         // 固定为 ACCOUNT 类型
        'identifier' => ApiKey::generateTokenIdentifier(ApiKey::TYPE_ACCOUNT), // ptlc_ 前缀
        'token'      => encrypt($plain = Str::random(ApiKey::KEY_LENGTH)),     // 32 字符随机
        'memo'       => $memo ?? '',
        'allowed_ips'=> $ips ?? [],
    ]);
    return new NewAccessToken($token, $plain);
}
```

调用链：
1. 用户发 POST `/api/client/account/api-keys`
2. `StoreApiKeyRequest` 校验 `description` + `allowed_ips`（IP 格式校验，最多 50 个）
3. `ApiKeyController@store` 检查用户 Key 数量上限（25 个）
4. 调用 `$request->user()->createToken(...)` → 上述方法
5. 返回 `secret_token`（完整明文 Key），**只出现这一次**

### 2.4 Application API Key 签发

文件：`app/Services/Api/KeyCreationService.php:38-51`

```php
public function handle(array $data, array $permissions = []): ApiKey
{
    $data = array_merge($data, [
        'key_type'   => $this->keyType,
        'identifier' => ApiKey::generateTokenIdentifier($this->keyType), // ptla_ 前缀
        'token'      => $this->encrypter->encrypt(str_random(ApiKey::KEY_LENGTH)),
    ]);
    if ($this->keyType === ApiKey::TYPE_APPLICATION) {
        $data = array_merge($data, $permissions);  // 合并 r_servers, r_nodes 等权限列
    }
    return $this->repository->create($data, true, true);
}
```

调用链：
1. 管理员在 Web 后台 `/admin/api/new` 填写表单
2. `StoreApplicationApiKeyRequest` 校验 memo + 各 `r_{resource}` 字段（0-3 范围）
3. `getKeyPermissions()` 提取所有 `r_` 开头的字段
4. `AdminController@store` 调用 `KeyCreationService->setKeyType(TYPE_APPLICATION)->handle()`
5. 重定向回列表页，完整明文 Key **在页面上显示一次**

### 2.5 签发流程对比

```
┌──────────────────────────────────────────────────────────────┐
│                  Client API Key 签发                          │
│  POST /api/client/account/api-keys                           │
│  → StoreApiKeyRequest (description + allowed_ips 校验)        │
│  → ApiKeyController@store (数量上限 25)                       │
│  → User::createToken() (HasAccessTokens trait)               │
│    → key_type = TYPE_ACCOUNT, 前缀 ptlc_                     │
│    → 无 r_* 权限列                                            │
│  → 返回 JSON 含 secret_token                                 │
└──────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│                Application API Key 签发                       │
│  POST /admin/api (Web 表单)                                  │
│  → StoreApplicationApiKeyRequest (memo + r_* 校验)            │
│  → ApiController@store                                       │
│  → KeyCreationService->handle()                              │
│    → key_type = TYPE_APPLICATION, 前缀 ptla_                  │
│    → 合并 r_servers/r_nodes/... 等权限列                      │
│  → 重定向到列表页，页面展示明文 Key                            │
└──────────────────────────────────────────────────────────────┘
```

---

## 三、请求认证管道（Authentication Pipeline）

所有 API 请求共享同一个中间件栈，在 `RouteServiceProvider` 中组装：

```
Route::middleware(['api', RequireTwoFactorAuthentication::class])
  ├── /api/application  →  ['application-api', 'throttle:api.application']
  └── /api/client       →  ['client-api',       'throttle:api.client']
```

### 3.1 `api` 中间件组（Kernel.php:70-77）

```php
'api' => [
    EnsureStatefulRequests::class,    // ① 判断是否为前端 SPA 的 Cookie 请求
    'auth:sanctum',                   // ② Sanctum 认证（Bearer Token 或 Session Cookie）
    IsValidJson::class,               // ③ JSON 格式校验
    TrackAPIKey::class,               // ④ 记录当前 Key ID 到 Activity 日志上下文
    RequireTwoFactorAuthentication::class, // ⑤ 二次验证要求
    AuthenticateIPAccess::class,      // ⑥ IP 白名单校验
],
```

### 3.2 Sanctum 认证 (`auth:sanctum`)

文件：`app/Providers/AuthServiceProvider.php:22`

```php
Sanctum::usePersonalAccessTokenModel(ApiKey::class);
```

Sanctum 被配置为使用 `ApiKey` 模型作为 Personal Access Token。认证流程：

1. **Stateful 请求**（前端 SPA 带 Cookie）：`EnsureStatefulRequests` 判断请求是否来自受信任的前端域名或携带 Session Cookie。如果是，Sanctum 使用 Session 认证，认证后生成 `TransientToken`。
2. **Stateless 请求**（Bearer Token）：Sanctum 从 `Authorization: Bearer <token>` 中提取令牌，调用 `ApiKey::findToken()` 完成认证。

**关键区别**：
- `TransientToken`：表示通过 Session Cookie 认证的前端请求，不受 IP 限制，不受 Key 类型限制
- `ApiKey` 实例：表示通过 Bearer Token 认证的 API 请求，受所有后续中间件约束

### 3.3 IP 白名单校验 (`AuthenticateIPAccess`)

文件：`app/Http/Middleware/Api/AuthenticateIPAccess.php`

```
if token 是 TransientToken → 放行（前端 Cookie 请求不受限）
if token.allowed_ips 为空 → 放行（无 IP 限制）
否则逐一匹配 allowed_ips（支持 CIDR）→ 不匹配则记录日志并 403
```

### 3.4 分流中间件

认证完成后，请求进入各自的分流中间件组：

**Application API 专属**（`application-api` 组）：
```php
'application-api' => [
    SubstituteBindings::class,
    AuthenticateApplicationUser::class,  // 必须 root_admin
],
```

**Client API 专属**（`client-api` 组）：
```php
'client-api' => [
    SubstituteClientBindings::class,
    RequireClientApiKey::class,          // 禁止 Application Key
],
```

---

## 四、范围校验（Scope Validation）

### 4.1 Application API — AdminAcl 位运算

文件：`app/Services/Acl/Api/AdminAcl.php`

AdminAcl 定义了 9 种资源，每种资源有 3 个权限级别：

```
NONE = 0   (二进制 00)
READ = 1   (二进制 01)
WRITE = 2  (二进制 10)

READ | WRITE = 3 (二进制 11，读写均可)
```

校验方法：
```php
AdminAcl::check(ApiKey $key, string $resource, int $action = self::READ): bool
// → 取 $key->r_{resource} 的值，用位运算判断是否包含 $action
// → can($permission & $action)
```

**校验触发点**：`ApplicationApiRequest::authorize()`

文件：`app/Http/Requests/Api/Application/ApplicationApiRequest.php:34-50`

```php
public function authorize(): bool
{
    $token = $this->user()->currentAccessToken();
    if ($token instanceof TransientToken) return true;  // Cookie 请求放行
    if ($token->key_type === ApiKey::TYPE_ACCOUNT) return true;  // Account Key 放行
    return AdminAcl::check($token, $this->resource, $this->permission); // Application Key 校验
}
```

每个 Application API 的 Request 子类都声明了 `$resource` 和 `$permission`，例如：
- 查看 Servers 列表 → `$resource = 'servers'`, `$permission = AdminAcl::READ`
- 创建 Server → `$resource = 'servers'`, `$permission = AdminAcl::WRITE`

**注意**：`TYPE_ACCOUNT` 的 Key 访问 Application API 时**自动放行**（因为 Account Key 是用户自己的，而 Application API 要求 `root_admin`，所以前面 `AuthenticateApplicationUser` 中间件已经保证了只有管理员才能到这里）。但 `TYPE_APPLICATION` 的 Key **必须通过 AdminAcl 校验**。

### 4.2 Client API — Permission + ServerPolicy

文件：`app/Models/Permission.php`、`app/Http/Requests/Api/Client/ClientApiRequest.php`

Client API 的权限模型完全不同于 AdminAcl，它基于 **子用户（Subuser）+ 服务器级权限**：

```php
// ClientApiRequest::authorize()
if ($this instanceof ClientPermissionsRequest || method_exists($this, 'permission')) {
    $server = $this->route()->parameter('server');
    return $this->user()->can($this->permission(), $server);
    // → ServerPolicy 检查用户是否为服务器 Owner 或拥有对应 Permission
}
return true;  // 非 Server 相关请求（如账户 API）直接放行
```

Permission 模型定义了细粒度的操作权限（如 `websocket.connect`、`file.read-content`、`backup.restore` 等），存储在 `permissions` 表中，关联到 `subusers` 表。

### 4.3 Transformer 层的二次鉴权

即使控制器层通过了鉴权，在 Transformer 序列化时还有一道检查：

**Application Transformer**（`BaseTransformer::authorize`）：
```php
if ($token->key_type === ApiKey::TYPE_ACCOUNT) {
    return $this->request->user()->root_admin;  // Account Key 必须是管理员
}
return AdminAcl::check($token, $resource);       // Application Key 用 AdminAcl
```

**Client Transformer**（`BaseClientTransformer::authorize`）：
```php
return $this->request->user()->can($ability, [$server]);  // 用 ServerPolicy
```

这确保了 include 关联资源时不会泄露无权限的数据。

### 4.4 Key 类型隔离 — RequireClientApiKey

文件：`app/Http/Middleware/Api/Client/RequireClientApiKey.php`

```php
if ($token instanceof ApiKey && $token->key_type === ApiKey::TYPE_APPLICATION) {
    throw new AccessDeniedHttpException(
        'You are attempting to use an application API key on an endpoint that requires a client API key.'
    );
}
```

这是**单向隔离**：Application Key 不能访问 Client API，但 Client Account Key 在通过 `root_admin` 检查后可以访问 Application API。

### 4.5 范围校验全景图

```
请求到达
  │
  ▼
auth:sanctum 识别出 ApiKey(或 TransientToken)
  │
  ├─ /api/application ──→ AuthenticateApplicationUser (root_admin?)
  │                       │
  │                       ApplicationApiRequest::authorize()
  │                       ├─ TransientToken → 放行
  │                       ├─ TYPE_ACCOUNT  → 放行（已由 root_admin 保证）
  │                       └─ TYPE_APPLICATION → AdminAcl::check(r_{resource}, action)
  │
  └─ /api/client ──→ RequireClientApiKey (TYPE_APPLICATION → 403)
                      │
                      ClientApiRequest::authorize()
                      ├─ 非 Server 请求 → 放行
                      └─ Server 请求 → user()->can(permission, $server)
                                        ├─ Owner → 全部权限
                                        └─ Subuser → 查 permissions 表
```

---

## 五、频率限制（Rate Limiting）

### 5.1 三层频率限制体系

```
┌─────────────────────────────────────────────────────────┐
│ 第一层：全局 API 速率限制                                  │
│   由 RouteServiceProvider 注册，按 Key 维度限制             │
│   ┌────────────────┬─────────────────────────────┐      │
│   │ api.client     │ 256 次/分钟 (可配)            │      │
│   │ api.application│ 256 次/分钟 (可配)            │      │
│   └────────────────┴─────────────────────────────┘      │
│   限制维度：用户 UUID（已认证）或 IP（未认证）               │
├─────────────────────────────────────────────────────────┤
│ 第二层：资源创建限制（ResourceLimit 枚举）                   │
│   仅作用于 Client API 的特定创建端点                        │
│   限制维度：服务器 UUID（按服务器限，不是按用户）             │
│   ┌────────────┬──────────────────────────────┐         │
│   │ Backup     │ 3 次 / 15 分钟                │         │
│   │ Database   │ 2 次 / 分钟                   │         │
│   │ FilePull   │ 5 次 / 10 分钟                │         │
│   │ Subuser    │ 10 次 / 15 分钟               │         │
│   │ Websocket  │ 5 次 / 分钟                   │         │
│   │ Allocation │ 2 次 / 分钟 (default)          │         │
│   │ Schedule   │ 2 次 / 分钟 (default)          │         │
│   └────────────┴──────────────────────────────┘         │
├─────────────────────────────────────────────────────────┤
│ 第三层：认证端点限制                                        │
│   登录：10 次/分钟                                         │
│   忘记密码：2 次/分钟（按 IP）                              │
└─────────────────────────────────────────────────────────┘
```

### 5.2 全局 API 限制配置

文件：`config/http.php`

```php
'rate_limit' => [
    'client_period'      => 1,
    'client'             => env('APP_API_CLIENT_RATELIMIT', 256),
    'application_period' => 1,
    'application'        => env('APP_API_APPLICATION_RATELIMIT', 256),
],
```

### 5.3 限制器注册

文件：`app/Providers/RouteServiceProvider.php:93-109`

```php
RateLimiter::for('api.client', function (Request $request) {
    $key = optional($request->user())->uuid ?: $request->ip();
    return Limit::perMinutes(config('http.rate_limit.client_period'), config('http.rate_limit.client'))
        ->by($key);
});

RateLimiter::for('api.application', function (Request $request) {
    $key = optional($request->user())->uuid ?: $request->ip();
    return Limit::perMinutes(config('http.rate_limit.application_period'), config('http.rate_limit.application'))
        ->by($key);
});
```

**关键设计**：限制维度是 **用户 UUID** 而非 API Key。这意味着同一个用户的所有 Key 共享配额，切换 Key 无法绕过限制。未认证时退化为 IP 维度。

### 5.4 路由上的 Throttle 中间件

```php
// RouteServiceProvider 中路由注册
Route::middleware(['api', RequireTwoFactorAuthentication::class])->group(function () {
    Route::middleware(['application-api', 'throttle:api.application'])
        ->prefix('/api/application')
        ->group(...);

    Route::middleware(['client-api', 'throttle:api.client'])
        ->prefix('/api/client')
        ->group(...);
});
```

`throttle:api.application` 和 `throttle:api.client` 是在路由层声明的，确保每条路由都受到对应的速率限制。

### 5.5 ResourceLimit — 服务器级资源限制

文件：`app/Enum/ResourceLimit.php`

这是一个 PHP 8.1 枚举，用于对 **创建型操作** 施加更严格的限制：

```php
enum ResourceLimit {
    case Allocation;  // 2次/分钟
    case Backup;      // 3次/15分钟
    case Database;    // 2次/分钟
    case Schedule;    // 2次/分钟
    case Subuser;     // 10次/15分钟
    case Websocket;   // 5次/分钟
    case FilePull;    // 5次/10分钟
}
```

使用方式（在路由中）：
```php
// routes/api-client.php
Route::middleware([ResourceLimit::Backup->middleware()])
    ->post('/{backup}/restore', ...);
```

限制维度是 **服务器 UUID**（`$case->limit()->by($server->uuid)`），而非用户。这是为了防止单个服务器上的资源被过度创建，即使不同用户操作同一台服务器也共享配额。

---

## 六、完整请求生命周期

以一个典型的 Client API 请求为例，完整中间件执行顺序：

```
HTTP 请求: GET /api/client/servers/xxx/files/list
Authorization: Bearer ptlc_xxxxxxxxxxxxxxyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyy

1. 全局中间件 (Kernel::$middleware)
   ├── TrustProxies
   ├── HandleCors
   ├── PreventRequestsDuringMaintenance
   ├── ValidatePostSize
   ├── TrimStrings
   ├── ConvertEmptyStringsToNull
   └── SetSecurityHeaders

2. api 中间件组
   ├── EnsureStatefulRequests     → 非 SPA 请求，跳过
   ├── auth:sanctum               → Bearer Token → ApiKey::findToken() → 认证成功
   ├── IsValidJson                → GET 请求，跳过
   ├── TrackAPIKey                → LogTarget::setApiKeyId($token->id)
   ├── RequireTwoFactorAuthentication → 检查 2FA 配置
   └── AuthenticateIPAccess       → 检查 allowed_ips 白名单

3. client-api 中间件组
   ├── SubstituteClientBindings   → 路由模型绑定
   └── RequireClientApiKey        → key_type !== TYPE_APPLICATION → 放行

4. throttle:api.client            → 检查用户 UUID 维度速率限制

5. 路由中间件
   ├── ServerSubject              → Activity 日志绑定
   ├── AuthenticateServerAccess   → 用户是否有权访问此服务器
   └── ResourceBelongsToServer    → 资源是否属于此服务器

6. 控制器
   └── ClientApiRequest::authorize() → user()->can('file.read', $server)

7. Transformer 层
   └── BaseClientTransformer::authorize() → 二次鉴权
```

---

## 七、容易混淆的设计点

### 7.1 Account Key 为什么能通过 Application API 的 authorize()？

`ApplicationApiRequest::authorize()` 中有：
```php
if ($token->key_type === ApiKey::TYPE_ACCOUNT) return true;
```

这看起来是"Account Key 拥有全部 Application 权限"，但实际上 `AuthenticateApplicationUser` 中间件已经确保了只有 `root_admin` 用户才能到达这里。Account Key 本身就属于管理员，所以无需 AdminAcl 细分权限。

### 7.2 TYPE_APPLICATION 已标记 @deprecated

`ApiKey` 模型中 `TYPE_APPLICATION = 2` 已被标记为废弃。当前系统仍然支持签发和使用 Application Key，但未来可能移除。Account Key + root_admin 的组合正在成为访问管理 API 的推荐方式。

### 7.3 AdminAcl 的权限列只对 Application Key 有意义

`r_servers`、`r_nodes` 等列只在使用 `KeyCreationService` 签发 Application Key 时写入。Client Account Key 签发时不涉及这些列，它们的值默认为 0（NONE）。AdminAcl::check() 只在 `ApplicationApiRequest::authorize()` 中对 Application Key 调用。

### 7.4 ResourceLimit 是"服务器维度"而非"用户维度"

全局速率限制（`api.client`/`api.application`）是按用户 UUID 限的，但 ResourceLimit 是按服务器 UUID 限的。这意味着：
- 一个用户在 10 台服务器上各创建 2 个数据库 → 全局限制内
- 但在 1 台服务器上连续创建 3 个数据库 → ResourceLimit 会拦截第 3 个

### 7.5 TransientToken 的特殊地位

通过前端 SPA Cookie 认证的请求产生 `TransientToken`：
- 不受 `AuthenticateIPAccess` 限制
- 在 `ApplicationApiRequest::authorize()` 中直接放行
- 不受 `RequireClientApiKey` 检查（因为它不是 `ApiKey` 实例）

这意味着前端管理员 Session 在 API 层面拥有最大的自由度。

### 7.6 ApiKey::can() 始终返回 false

```php
public function can($ability) { return false; }
```

这是 Laravel Sanctum `HasAbilities` 接口的方法，原本用于按 ability 字符串校验权限。但 Pterodactyl 使用了自己的 `AdminAcl` 体系而非 Sanctum 的 ability 机制，所以这个方法未被实现，始终返回 false。实际权限校验走的是 `AdminAcl::check()` 和 `ServerPolicy`。
