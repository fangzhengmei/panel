# API Key 流程梳理：密钥签发 → 范围校验 → 频率限制

## 一、全局概览

本项目（Pterodactyl Panel）存在 **两套独立的 API 接口**，共享同一张 `api_keys` 数据表和同一个 Sanctum 认证管道，但权限模型完全不同。最容易让人困惑的是：**管理端 API Key 的签发入口（Web 后台）和使用入口（Application API）走完全不同的中间件栈**。

| 维度 | Application API（管理接口） | Client API（客户端接口） |
|------|---------------------------|------------------------|
| 接口前缀 | `/api/application` | `/api/client` |
| Key 类型 | `TYPE_APPLICATION = 2`（已标记 @deprecated） | `TYPE_ACCOUNT = 1` |
| Key 前缀 | `ptla_` | `ptlc_` |
| 权限模型 | `AdminAcl` — 列级 `r_{resource}` 位运算 | `Permission` + `ServerPolicy` — 子用户权限 |
| 签发入口 | Admin Web 后台（`/admin/api/new`，Web 路由） | Client API 自助（`/api/client/account/api-keys`，API 路由） |
| 签发中间件 | `web` 组 + `AdminAuthenticate`（Session Cookie） | `api` 组 + `auth:sanctum`（Bearer Token） |
| 使用时管理员校验 | `AuthenticateApplicationUser`（必须 `root_admin`） | `RequireClientApiKey`（禁止 Application Key 闯入） |
| 使用时权限校验 | `AdminAcl::check()`（对 Application Key） | `user()->can(permission, $server)`（ServerPolicy） |

两套 API 接口在 `RouteServiceProvider` 中被挂载到 **同一个** `api` 中间件组上，共享认证与 IP 校验逻辑，之后才分流到各自专属的中间件栈。而管理端 Web 后台（Key 签发入口）则完全独立，走 `web` 中间件组。

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

---

### 2.3 管理端 API Key 签发（Application Key）—— 完整路径逐段解析

这是最难以理解的部分，因为 **签发入口走 Web 路由，而使用时走 API 路由**。下面逐段拆解完整调用链。

#### 2.3.1 路由挂载层 — `RouteServiceProvider`

文件：`app/Providers/RouteServiceProvider.php:38-48`

```php
$this->routes(function () {
    Route::middleware('web')->group(function () {
        // 基础 Web 路由
        Route::middleware(['auth.session', RequireTwoFactorAuthentication::class])
            ->group(base_path('routes/base.php'));

        // ⭐ 管理端 Web 路由（包括 API Key 签发入口）
        Route::middleware(['auth.session', RequireTwoFactorAuthentication::class, AdminAuthenticate::class])
            ->prefix('/admin')
            ->group(base_path('routes/admin.php'));  // ← Key 签发路由在这里

        Route::middleware('guest')->prefix('/auth')->group(base_path('routes/auth.php'));
    });

    // ⭐ API 接口路由（Application API + Client API）
    Route::middleware(['api', RequireTwoFactorAuthentication::class])->group(function () {
        Route::middleware(['application-api', 'throttle:api.application'])
            ->prefix('/api/application')
            ->scopeBindings()
            ->group(base_path('routes/api-application.php'));

        Route::middleware(['client-api', 'throttle:api.client'])
            ->prefix('/api/client')
            ->scopeBindings()
            ->group(base_path('routes/api-client.php'));
    });
    // ...
});
```

**关键分流点**：管理端 Web 后台（Key 签发）走 `web` 中间件组（Session Cookie 认证），而 Application API（Key 使用）走 `api` 中间件组（Bearer Token 认证）。它们唯一的连接点是 **共享同一张 `api_keys` 数据表**。

#### 2.3.2 管理端 API Key 路由定义 — `routes/admin.php`

文件：`routes/admin.php:17-24`

```php
Route::group(['prefix' => 'api'], function () {
    Route::get('/', [Admin\ApiController::class, 'index'])->name('admin.api.index');
    Route::get('/new', [Admin\ApiController::class, 'create'])->name('admin.api.new');
    Route::post('/new', [Admin\ApiController::class, 'store']);
    Route::delete('/revoke/{identifier}', [Admin\ApiController::class, 'delete'])->name('admin.api.delete');
});
```

**路由与处理器对应关系表**：

| URL 路径 | HTTP 方法 | 路由名称 | 控制器方法 | Request 类 | 中间件 | 响应类型 |
|---------|-----------|----------|-----------|------------|--------|----------|
| `/admin/api` | GET | `admin.api.index` | `ApiController@index` | （无） | `web` + `AdminAuthenticate` | Blade 视图（Key 列表） |
| `/admin/api/new` | GET | `admin.api.new` | `ApiController@create` | （无） | `web` + `AdminAuthenticate` | Blade 视图（创建表单） |
| `/admin/api/new` | POST | （无，同路径不同方法） | `ApiController@store` | `StoreApplicationApiKeyRequest` | `web` + `AdminAuthenticate` | 重定向到列表 |
| `/admin/api/revoke/{identifier}` | DELETE | `admin.api.delete` | `ApiController@delete` | （无） | `web` + `AdminAuthenticate` | 204 No Content |

#### 2.3.3 第一步：访问创建页面 — GET `/admin/api/new`

**中间件执行顺序**：
1. `web` 全局中间件组（EncryptCookies → StartSession → VerifyCsrfToken 等）
2. `auth.session` → 通过 Session Cookie 认证用户
3. `RequireTwoFactorAuthentication` → 检查 2FA 配置
4. `AdminAuthenticate` → 检查 `user()->root_admin`

**控制器方法**：`app/Http/Controllers/Admin/ApiController.php:42-55`

```php
public function create(): View
{
    $resources = AdminAcl::getResourceList();  // 反射获取所有 RESOURCE_* 常量
    sort($resources);

    return view('admin.api.new', [
        'resources' => $resources,
        'permissions' => [
            'r'  => AdminAcl::READ,            // 1
            'rw' => AdminAcl::READ | AdminAcl::WRITE,  // 3
            'n'  => AdminAcl::NONE,            // 0
        ],
    ]);
}
```

`AdminAcl::getResourceList()` 通过反射读取类中所有 `RESOURCE_` 开头的常量，返回 9 种资源：
- `servers`, `nodes`, `allocations`, `users`, `locations`, `nests`, `eggs`, `database_hosts`, `server_databases`

#### 2.3.4 第二步：表单渲染 — `resources/views/admin/api/new.blade.php`

表单的核心结构：

```blade
<form method="POST" action="{{ route('admin.api.new') }}">
    <!-- 9 种资源，每种 3 个单选按钮 -->
    @foreach($resources as $resource)
        <tr>
            <td>{{ str_replace('_', ' ', title_case($resource)) }}</td>
            <td>
                <input type="radio" id="r_{{ $resource }}" 
                       name="r_{{ $resource }}" value="{{ $permissions['r'] }}">
                <label for="r_{{ $resource }}">Read</label>
            </td>
            <td>
                <input type="radio" id="rw_{{ $resource }}" 
                       name="r_{{ $resource }}" value="{{ $permissions['rw'] }}">
                <label for="rw_{{ $resource }}">Read &amp; Write</label>
            </td>
            <td>
                <input type="radio" id="n_{{ $resource }}" 
                       name="r_{{ $resource }}" value="{{ $permissions['n'] }}" checked>
                <label for="n_{{ $resource }}">None</label>
            </td>
        </tr>
    @endforeach

    <!-- 描述字段 -->
    <div class="form-group">
        <label for="memoField">Description</label>
        <input id="memoField" type="text" name="memo" class="form-control">
    </div>

    {{ csrf_field() }}
    <button type="submit" class="btn btn-success">Create Credentials</button>
</form>
```

**表单提交时的字段**：
- `memo` → 描述文本
- `r_servers` → 0/1/3（None/Read/Read&Write）
- `r_nodes` → 0/1/3
- `r_allocations` → 0/1/3
- ...（共 9 个 `r_*` 字段）
- `_token` → CSRF Token

#### 2.3.5 第三步：表单提交 — POST `/admin/api/new`

**Request 类继承链**：

```
StoreApplicationApiKeyRequest
    ↓ extends
AdminFormRequest
    ↓ extends
FormRequest (Laravel 基础类)
```

**第一层鉴权 — `AdminFormRequest::authorize()`**

文件：`app/Http/Requests/Admin/AdminFormRequest.php:18-25`

```php
public function authorize(): bool
{
    if (is_null($this->user())) {
        return false;
    }
    return (bool) $this->user()->root_admin;  // 必须是 root_admin
}
```

**第二层校验 — `StoreApplicationApiKeyRequest::rules()`**

文件：`app/Http/Requests/Admin/Api/StoreApplicationApiKeyRequest.php:15-22`

```php
public function rules(): array
{
    $modelRules = ApiKey::getRules();

    // 动态为每个资源生成验证规则
    return collect(AdminAcl::getResourceList())->mapWithKeys(function ($resource) use ($modelRules) {
        return [AdminAcl::COLUMN_IDENTIFIER . $resource => $modelRules['r_' . $resource]];
        // → 'r_servers' => 'integer|min:0|max:3'
        // → 'r_nodes'   => 'integer|min:0|max:3'
        // ... 共 9 个
    })->merge(['memo' => $modelRules['memo']])->toArray();
}
```

**提取权限字段 — `StoreApplicationApiKeyRequest::getKeyPermissions()`**

文件：`app/Http/Requests/Admin/Api/StoreApplicationApiKeyRequest.php:31-36`

```php
public function getKeyPermissions(): array
{
    return collect($this->validated())->filter(function ($value, $key) {
        return substr($key, 0, strlen(AdminAcl::COLUMN_IDENTIFIER)) === AdminAcl::COLUMN_IDENTIFIER;
        // 只保留以 'r_' 开头的字段
    })->toArray();
}
```

#### 2.3.6 第四步：控制器处理 — `ApiController@store`

文件：`app/Http/Controllers/Admin/ApiController.php:62-72`

```php
public function store(StoreApplicationApiKeyRequest $request): RedirectResponse
{
    $this->keyCreationService->setKeyType(ApiKey::TYPE_APPLICATION)->handle([
        'memo'    => $request->input('memo'),
        'user_id' => $request->user()->id,
    ], $request->getKeyPermissions());  // 传入 9 个 r_* 权限

    $this->alert->success('A new application API key has been generated for your account.')->flash();

    return redirect()->route('admin.api.index');  // 重定向回列表页
}
```

#### 2.3.7 第五步：服务层创建 — `KeyCreationService@handle`

文件：`app/Services/Api/KeyCreationService.php:38-51`

```php
public function handle(array $data, array $permissions = []): ApiKey
{
    $data = array_merge($data, [
        'key_type'   => $this->keyType,                   // TYPE_APPLICATION = 2
        'identifier' => ApiKey::generateTokenIdentifier($this->keyType),  // ptla_ 前缀
        'token'      => $this->encrypter->encrypt(str_random(ApiKey::KEY_LENGTH)),  // 加密 32 字符随机
    ]);
    
    if ($this->keyType === ApiKey::TYPE_APPLICATION) {
        $data = array_merge($data, $permissions);  // 合并 r_servers, r_nodes 等权限列
    }
    
    return $this->repository->create($data, true, true);
}
```

#### 2.3.8 第六步：明文 Key 展示 — 列表页视图

文件：`resources/views/admin/api/index.blade.php:37-43`

```blade
@foreach($keys as $key)
    <tr>
        <td><code>
            @if (Auth::user()->is($key->user))
                {{ $key->identifier . decrypt($key->token) }}
                {{-- ⭐ 只有 Key 创建者本人才能看到完整明文 --}}
            @else
                {{ $key->identifier . '****' }}
                {{-- 其他管理员只能看到 identifier --}}
            @endif
        </code></td>
        <!-- ... -->
    </tr>
@endforeach
```

**明文 Key 展示的唯一性**：完整明文 Key 仅在重定向后的列表页展示一次，刷新页面后仍然可见（因为是实时解密展示），但只有创建者本人能看到。这是与 Client API Key 最大的不同 —— Client Key 仅在创建响应的 JSON 中返回一次，之后永远无法再获取明文。

#### 2.3.9 Application Key 签发完整流程图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                Application API Key 签发完整流程                               │
└─────────────────────────────────────────────────────────────────────────────┘
        │
        ▼
GET /admin/api/new
        │
        ├─ 中间件：web → auth.session → Require2FA → AdminAuthenticate
        │
        ▼
Admin\ApiController@create()
        │
        ├─ 调用 AdminAcl::getResourceList() 获取 9 种资源
        │
        ▼
渲染视图 admin.api.new
        │
        ├─ 表单包含：memo + 9 组 r_{resource} 单选按钮
        │  每组三个选项：None(0) / Read(1) / Read&Write(3)
        │
        ▼
用户填写表单，点击提交 → POST /admin/api/new
        │
        ├─ 中间件同上（Session Cookie 认证）
        │
        ▼
StoreApplicationApiKeyRequest 验证
        │
        ├─ authorize() → AdminFormRequest 检查 root_admin
        ├─ rules()     → 动态生成 9 个 r_* 字段规则（0-3 范围）
        ├─ getKeyPermissions() → 提取所有 r_ 开头字段
        │
        ▼
Admin\ApiController@store()
        │
        ├─ KeyCreationService->setKeyType(TYPE_APPLICATION)->handle()
        │
        ▼
KeyCreationService@handle()
        │
        ├─ 生成 identifier（ptla_ 前缀）
        ├─ 加密生成 token（32 随机字符）
        ├─ 合并 memo + user_id + 9 个 r_* 权限
        ├─ ApiKeyRepository->create() 写入数据库
        │
        ▼
重定向到 GET /admin/api
        │
        ▼
Admin\ApiController@index()
        │
        ├─ 查询所有 TYPE_APPLICATION 的 Key
        │
        ▼
渲染列表页视图
        │
        ├─ 对当前用户创建的 Key：{{ identifier + decrypt(token) }} → 显示完整明文
        ├─ 对其他用户创建的 Key：{{ identifier + '****' }} → 仅显示 identifier
```

---

### 2.4 Client API Key 签发

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
5. 返回 `secret_token`（完整明文 Key），**只出现这一次**（数据库中加密存储，之后无法再解密展示）

---

### 2.5 签发流程对比

```
┌──────────────────────────────────────────────────────────────────────┐
│                  Client API Key 签发                                  │
│  接口：POST /api/client/account/api-keys                              │
│  认证：Bearer Token（api 中间件组 + auth:sanctum）                   │
│  处理器：ApiKeyController@store                                       │
│  Request：StoreApiKeyRequest（校验 description + allowed_ips）        │
│  服务：User::createToken()（HasAccessTokens trait）                   │
│  Key 类型：TYPE_ACCOUNT，前缀 ptlc_                                    │
│  权限列：不写入 r_*（全部为 0）                                        │
│  明文展示：JSON 响应中 secret_token 字段，仅返回一次                   │
└──────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────┐
│                Application API Key 签发                               │
│  接口：POST /admin/api/new（Web 表单）                                │
│  认证：Session Cookie（web 中间件组 + AdminAuthenticate）             │
│  处理器：Admin\ApiController@store                                    │
│  Request：StoreApplicationApiKeyRequest（校验 memo + 9 个 r_*）       │
│  服务：KeyCreationService->handle()                                   │
│  Key 类型：TYPE_APPLICATION，前缀 ptla_                                │
│  权限列：写入 r_servers/r_nodes/... 等 9 个权限列                      │
│  明文展示：重定向后列表页实时解密展示，创建者每次访问都可见             │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 三、认证分流处的连接方式

这是整个体系最核心的设计。**管理端 API Key 的签发入口和使用入口走完全不同的中间件栈，但共享同一张 `api_keys` 数据表**。

### 3.1 双轨中间件架构

```
                        ┌───────────────────────────────────────────┐
                        │              HTTP 请求入口                 │
                        └───────────────────────────────────────────┘
                                       │
                    ┌──────────────────┴───────────────────┐
                    ▼                                      ▼
           ┌──────────────────────┐             ┌──────────────────────┐
           │   Web 后台路由组       │             │    API 接口路由组     │
           │  /admin/*（签发 Key）  │             │  /api/*（使用 Key）  │
           └──────────────────────┘             └──────────────────────┘
                    │                                      │
                    ▼                                      ▼
           走 web 中间件组                       走 api 中间件组
┌────────────────────────────────────┐  ┌────────────────────────────────────┐
│  EncryptCookies                    │  │  EnsureStatefulRequests            │
│  AddQueuedCookiesToResponse        │  │  auth:sanctum                     │
│  StartSession                      │  │  IsValidJson                       │
│  ShareErrorsFromSession            │  │  TrackAPIKey                       │
│  VerifyCsrfToken                   │  │  RequireTwoFactorAuthentication    │
│  SubstituteBindings                │  │  AuthenticateIPAccess              │
│  LanguageMiddleware                │  └────────────────────────────────────┘
└────────────────────────────────────┘                  │
                    │                                   ▼
                    ▼                         ┌──────────────────────┐
           admin 专属中间件                     │  分流中间件组        │
┌────────────────────────────────────┐          ├──────────────────────┤
│  auth.session                      │          │  application-api    │
│  RequireTwoFactorAuthentication    │          │    └─ AuthenticateApplicationUser (root_admin?)│
│  AdminAuthenticate (root_admin?)   │          │  throttle:api.application │
└────────────────────────────────────┘          ├──────────────────────┤
                    │                           │  client-api         │
                    ▼                           │    └─ RequireClientApiKey (!TYPE_APPLICATION?)│
           routes/admin.php                     │  throttle:api.client│
           ├─ GET  /admin/api                   └──────────────────────┘
           ├─ GET  /admin/api/new                        │
           ├─ POST /admin/api/new                        ▼
           └─ DELETE /admin/api/revoke/{id}    routes/api-application.php
                    │                           routes/api-client.php
                    ▼                                      │
           Admin\ApiController                             ▼
           ├─ index()  →  Key 列表             Api\Application\*Controller
           ├─ create() → 创建表单               Api\Client\*Controller
           ├─ store()  → 创建 Key                       │
           └─ delete() → 删除 Key                       ▼
                    │                           ApplicationApiRequest::authorize()
                    ▼                           ClientApiRequest::authorize()
           KeyCreationService                            │
                    │                                   ▼
                    └───────────────┬───────────────────┘
                                    ▼
                          ┌───────────────────┐
                          │   api_keys 表      │
                          │  共享数据存储      │
                          └───────────────────┘
```

### 3.2 `RouteServiceProvider` 中的分流点

文件：`app/Providers/RouteServiceProvider.php:38-60`

```php
$this->routes(function () {
    // ═══════════════ 第一组：Web 路由 ═══════════════
    Route::middleware('web')->group(function () {
        // 基础页面路由
        Route::middleware(['auth.session', RequireTwoFactorAuthentication::class])
            ->group(base_path('routes/base.php'));

        // ⭐ 管理端 Web 路由（Key 签发入口）
        Route::middleware([
            'auth.session', 
            RequireTwoFactorAuthentication::class, 
            AdminAuthenticate::class  // ← 检查 root_admin
        ])
            ->prefix('/admin')
            ->group(base_path('routes/admin.php'));

        Route::middleware('guest')->prefix('/auth')->group(base_path('routes/auth.php'));
    });

    // ═══════════════ 第二组：API 接口路由 ═══════════════
    Route::middleware(['api', RequireTwoFactorAuthentication::class])->group(function () {
        // ⭐ Application API（Key 使用入口之一）
        Route::middleware([
            'application-api',  // ← 包含 AuthenticateApplicationUser
            'throttle:api.application'
        ])
            ->prefix('/api/application')
            ->scopeBindings()
            ->group(base_path('routes/api-application.php'));

        // ⭐ Client API（Key 使用入口之二）
        Route::middleware([
            'client-api',       // ← 包含 RequireClientApiKey
            'throttle:api.client'
        ])
            ->prefix('/api/client')
            ->scopeBindings()
            ->group(base_path('routes/api-client.php'));
    });
});
```

**分流逻辑解析**：

1. **按路径前缀分流**：
   - `/admin/*` → Web 路由，Session Cookie 认证
   - `/api/application/*` → API 路由，Bearer Token 认证
   - `/api/client/*` → API 路由，Bearer Token 认证

2. **按认证方式分流**：
   - Web 路由使用 `auth.session`（Laravel Session Guard）
   - API 路由使用 `auth:sanctum`（Sanctum Guard，调用 `ApiKey::findToken()`）

3. **管理员身份双重校验**：
   - Web 管理路由：`AdminAuthenticate` 中间件检查 `root_admin`
   - Application API：`AuthenticateApplicationUser` 中间件检查 `root_admin`

4. **Key 类型隔离**：
   - Client API：`RequireClientApiKey` 中间件拒绝 `TYPE_APPLICATION` 的 Key

### 3.3 `api` 中间件组详细解析

文件：`app/Http/Kernel.php:70-77`

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

**每一步的衔接逻辑**：

**① `EnsureStatefulRequests`** — 决定后续认证路径
```php
// 继承自 Sanctum 的 EnsureFrontendRequestsAreStateful
public static function fromFrontend($request)
{
    if (parent::fromFrontend($request)) return true;
    return $request->hasCookie(config('session.cookie'));
}
```
- 如果来自前端 SPA 域名或携带 Session Cookie → 走 Session 认证，生成 `TransientToken`
- 否则 → 走 Bearer Token 认证，生成 `ApiKey` 实例

**② `auth:sanctum`** — Sanctum Guard 认证
- 若是 Stateful 请求：通过 Session 认证，token 为 `TransientToken`
- 若是 Stateless 请求：从 `Authorization: Bearer <token>` 提取令牌 → 调用 `ApiKey::findToken()` → 查表 → 解密 → 比对 → 认证成功时 token 为 `ApiKey` 实例

**③ `IsValidJson`** — 仅对 POST/PUT/PATCH 等有请求体的方法校验 JSON 格式

**④ `TrackAPIKey`** — 记录 Key 上下文
```php
// 如果是 ApiKey 实例，记录 ID 到 LogTarget；TransientToken 则记 null
LogTarget::setApiKeyId($token instanceof ApiKey ? $token->id : null);
```

**⑤ `RequireTwoFactorAuthentication`** — 检查 2FA 配置，API 请求直接抛异常而非重定向

**⑥ `AuthenticateIPAccess`** — IP 白名单校验
```php
if ($token instanceof TransientToken) return $next($request);  // 前端请求跳过
if (empty($token->allowed_ips)) return $next($request);       // 无限制跳过
// 否则逐一匹配 CIDR，不匹配则记录日志并 403
```

### 3.4 分流后的专属中间件

**Application API 专属**（`application-api` 组，Kernel.php:78-81）：
```php
'application-api' => [
    SubstituteBindings::class,
    AuthenticateApplicationUser::class,  // 必须 root_admin
],
```

**Client API 专属**（`client-api` 组，Kernel.php:82-85）：
```php
'client-api' => [
    SubstituteClientBindings::class,
    RequireClientApiKey::class,          // 禁止 Application Key
],
```

**`AuthenticateApplicationUser` 源码**（`app/Http/Middleware/Api/Application/AuthenticateApplicationUser.php`）：
```php
public function handle(Request $request, \Closure $next): mixed
{
    $user = $request->user();
    if (!$user || !$user->root_admin) {
        throw new AccessDeniedHttpException('This account does not have permission to access the API.');
    }
    return $next($request);
}
```

**`RequireClientApiKey` 源码**（`app/Http/Middleware/Api/Client/RequireClientApiKey.php`）：
```php
public function handle(Request $request, \Closure $next): mixed
{
    $token = $request->user()->currentAccessToken();
    if ($token instanceof ApiKey && $token->key_type === ApiKey::TYPE_APPLICATION) {
        throw new AccessDeniedHttpException(
            'You are attempting to use an application API key on an endpoint that requires a client API key.'
        );
    }
    return $next($request);
}
```

---

## 四、请求认证管道（Authentication Pipeline）

### 4.1 Application API 请求完整生命周期

以一个典型的 Application API 请求为例，完整中间件执行顺序：

```
HTTP 请求: GET /api/application/servers
Authorization: Bearer ptla_xxxxxxxxxxxxxxyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyy

1. 全局中间件 (Kernel::$middleware)
   ├── TrustProxies
   ├── HandleCors
   ├── PreventRequestsDuringMaintenance
   ├── ValidatePostSize
   ├── TrimStrings
   ├── ConvertEmptyStringsToNull
   └── SetSecurityHeaders

2. api 中间件组
   ├── EnsureStatefulRequests     → 非 SPA 无 Cookie → 继续
   ├── auth:sanctum               → Bearer Token → ApiKey::findToken() → 识别为 TYPE_APPLICATION
   ├── IsValidJson                → GET 请求跳过
   ├── TrackAPIKey                → LogTarget::setApiKeyId($token->id)
   ├── RequireTwoFactorAuthentication → 检查 2FA
   └── AuthenticateIPAccess       → 检查 allowed_ips 白名单

3. application-api 中间件组
   ├── SubstituteBindings         → 路由模型绑定
   └── AuthenticateApplicationUser → 检查 root_admin → 通过

4. throttle:api.application       → 检查用户 UUID 维度速率限制（256/分钟）

5. 控制器 Request 层 — ApplicationApiRequest::authorize()
   ├── TransientToken? 否
   ├── TYPE_ACCOUNT? 否（是 TYPE_APPLICATION）
   └── AdminAcl::check($token, 'servers', AdminAcl::READ)
       → 读 $token->r_servers 的值 → 位运算 ($permission & $action) → 通过

6. 控制器
   └─ Application\Servers\ServerController@index

7. Transformer 层 — BaseTransformer::authorize()
   └─ AdminAcl::check($token, 'servers') → 二次校验（用于 include 关联资源）
```

### 4.2 Client API 请求完整生命周期

```
HTTP 请求: GET /api/client/servers/xxx/files/list
Authorization: Bearer ptlc_xxxxxxxxxxxxxxyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyy

1. 全局中间件 (同上)

2. api 中间件组 (同上)
   ├── EnsureStatefulRequests     → 继续
   ├── auth:sanctum               → 识别为 TYPE_ACCOUNT
   ├── IsValidJson                → 跳过
   ├── TrackAPIKey                → 记录 Key ID
   ├── RequireTwoFactorAuthentication → 通过
   └── AuthenticateIPAccess       → 检查 allowed_ips

3. client-api 中间件组
   ├── SubstituteClientBindings   → 路由模型绑定
   └── RequireClientApiKey        → key_type !== TYPE_APPLICATION → 通过

4. throttle:api.client            → 检查用户 UUID 速率限制

5. 路由中间件
   ├── ServerSubject              → Activity 日志绑定
   ├── AuthenticateServerAccess   → 用户是否有权访问此服务器
   └── ResourceBelongsToServer    → 资源是否属于此服务器

6. 控制器 Request 层 — ClientApiRequest::authorize()
   └─ user()->can('file.read', $server) → ServerPolicy 校验

7. Transformer 层 — BaseClientTransformer::authorize()
   └─ user()->can($ability, [$server]) → 二次鉴权
```

### 4.3 管理端 Web 请求生命周期（Key 签发）

```
HTTP 请求: POST /admin/api/new（表单提交）
Cookie: laravel_session=xxx

1. 全局中间件 (同上)

2. web 中间件组
   ├── EncryptCookies
   ├── AddQueuedCookiesToResponse
   ├── StartSession
   ├── ShareErrorsFromSession
   ├── VerifyCsrfToken            → 校验 _token 字段
   ├── SubstituteBindings
   └── LanguageMiddleware

3. admin 专属中间件
   ├── auth.session                → Session Guard 认证用户
   ├── RequireTwoFactorAuthentication → 检查 2FA
   └── AdminAuthenticate           → 检查 root_admin → 通过

4. 路由：POST /admin/api/new → Admin\ApiController@store

5. Request 层 — StoreApplicationApiKeyRequest
   ├── authorize() → AdminFormRequest 检查 root_admin
   └── rules() → 校验 memo + 9 个 r_* 字段

6. 控制器 → KeyCreationService → 写入 api_keys 表

7. 重定向到 GET /admin/api → 列表页展示明文 Key
```

---

## 五、范围校验（Scope Validation）

### 5.1 Application API — AdminAcl 位运算

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

### 5.2 Client API — Permission + ServerPolicy

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

### 5.3 Transformer 层的二次鉴权

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

---

## 六、频率限制（Rate Limiting）

### 6.1 三层频率限制体系

```
┌─────────────────────────────────────────────────────────┐
│ 第一层：全局 API 速率限制                                  │
│   由 RouteServiceProvider 注册，按用户 UUID 限制           │
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

### 6.2 全局 API 限制配置

文件：`config/http.php`

```php
'rate_limit' => [
    'client_period'      => 1,
    'client'             => env('APP_API_CLIENT_RATELIMIT', 256),
    'application_period' => 1,
    'application'        => env('APP_API_APPLICATION_RATELIMIT', 256),
],
```

### 6.3 限制器注册

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

**关键设计**：限制维度是 **用户 UUID** 而非 API Key。这意味着同一个用户的所有 Key 共享配额，切换 Key 无法绕过限制。

### 6.4 ResourceLimit — 服务器级资源限制

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

限制维度是 **服务器 UUID**，而非用户。防止单个服务器上的资源被过度创建。

---

## 七、容易混淆的设计点

### 7.1 为什么管理端 Key 签发走 Web 路由而不是 API 路由？

这是历史设计。管理端是传统的服务器端渲染（SSR）Web 应用，使用 Session Cookie 认证。而 Application API 是为第三方应用设计的，使用 Bearer Token 认证。**它们唯一的连接点是共享 `api_keys` 数据表**。

### 7.2 Application Key 明文为什么能反复查看？

管理端列表页的 `decrypt($key->token)` 是实时解密。因为管理端走 Session 认证，知道当前用户身份，可以判断 `Auth::user()->is($key->user)`。而 Client API 是无状态的，无法在列表查询时安全地解密返回明文。

### 7.3 Account Key 为什么能通过 Application API 的 authorize()？

`ApplicationApiRequest::authorize()` 中有：
```php
if ($token->key_type === ApiKey::TYPE_ACCOUNT) return true;
```

这看起来是"Account Key 拥有全部 Application 权限"，但实际上 `AuthenticateApplicationUser` 中间件已经确保了只有 `root_admin` 用户才能到达这里。Account Key 本身就属于管理员，所以无需 AdminAcl 细分权限。

### 7.4 TYPE_APPLICATION 已标记 @deprecated

`ApiKey` 模型中 `TYPE_APPLICATION = 2` 已被标记为废弃。当前系统仍然支持签发和使用 Application Key，但未来可能移除。Account Key + root_admin 的组合正在成为访问管理 API 的推荐方式。

### 7.5 AdminAcl 的权限列只对 Application Key 有意义

`r_servers`、`r_nodes` 等列只在使用 `KeyCreationService` 签发 Application Key 时写入。Client Account Key 签发时不涉及这些列，它们的值默认为 0（NONE）。AdminAcl::check() 只在 `ApplicationApiRequest::authorize()` 中对 Application Key 调用。

### 7.6 TransientToken 的特殊地位

通过前端 SPA Cookie 认证的请求产生 `TransientToken`：
- 不受 `AuthenticateIPAccess` 限制
- 在 `ApplicationApiRequest::authorize()` 中直接放行
- 不受 `RequireClientApiKey` 检查（因为它不是 `ApiKey` 实例）

这意味着前端管理员 Session 在 API 层面拥有最大的自由度。

### 7.7 ApiKey::can() 始终返回 false

```php
public function can($ability) { return false; }
```

这是 Laravel Sanctum `HasAbilities` 接口的方法，原本用于按 ability 字符串校验权限。但 Pterodactyl 使用了自己的 `AdminAcl` 体系而非 Sanctum 的 ability 机制，所以这个方法未被实现，始终返回 false。实际权限校验走的是 `AdminAcl::check()` 和 `ServerPolicy`。

### 7.8 Key 类型隔离是单向的

- Application Key → Client API：`RequireClientApiKey` 中间件直接 403
- Account Key → Application API：`AuthenticateApplicationUser` 检查 root_admin，通过后 `ApplicationApiRequest::authorize()` 直接放行

这是**单向隔离**。

---

## 八、明文可见性的最终结论（代码级验证）

### 8.1 管理端列表页显示完整 Key 的精确条件

**经过逐段代码验证，Application Key 的完整明文在管理端列表页的显示条件如下：**

**第一步：列表查询 — `ApiController@index()`**（`app/Http/Controllers/Admin/ApiController.php:30-35`）
```php
public function index(Request $request): View
{
    return view('admin.api.index', [
        'keys' => ApiKey::query()->where('key_type', ApiKey::TYPE_APPLICATION)->get(),
    ]);
}
```
→ 查询 `api_keys` 表中 **所有** `key_type = TYPE_APPLICATION` 的记录，**不按 user_id 过滤**。所有管理员都能看到所有 Application Key 的列表行。

**第二步：视图渲染 — Blade 条件判断**（`resources/views/admin/api/index.blade.php:38-42`）
```blade
@if (Auth::user()->is($key->user))
    {{ $key->identifier . decrypt($key->token) }}
    {{-- 完整明文：identifier + 解密后的 token --}}
@else
    {{ $key->identifier . '****' }}
    {{-- 只显示 identifier，token 部分用 **** 掩盖 --}}
@endif
```

**精确条件总结**：

| 条件 | 说明 | 结果 |
|------|------|------|
| `Auth::check()` | 必须已登录（由 `AdminAuthenticate` 中间件保证） | 基础前提 |
| `$key->key_type === TYPE_APPLICATION` | 是 Application Key（由 index() 查询保证） | 基础前提 |
| `Auth::user()->is($key->user)` | **当前登录用户 == Key 的创建者** | 显示完整明文 |
| 否则 | 其他管理员查看 | 只显示 `identifier****` |

**代码执行路径**：
```
GET /admin/api
  → web 中间件组 + AdminAuthenticate
  → ApiController@index()
    → ApiKey::where('key_type', TYPE_APPLICATION)->get()
    → 视图 admin.api.index
      → 循环渲染每一行
        → @if (Auth::user()->is($key->user))
          → decrypt($key->token) 实时解密
          → 拼接 identifier + 明文
        → @else
          → identifier + '****'
```

**最终结论**：Application Key 的完整明文 **并非所有人都能反复查看**。只有 Key 的创建者本人每次访问列表页时能看到完整明文。其他管理员只能看到 identifier。

### 8.2 两种 Key 的明文生命周期对比（修正版）

```
┌────────────────────────────────────────────────────────────────────────┐
│           Application Key (ptla_) 明文生命周期 — 修正版                  │
├────────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  签发阶段：                                                             │
│  KeyCreationService::handle()                                          │
│    └─ $this->encrypter->encrypt(str_random(ApiKey::KEY_LENGTH))        │
│       └─ str_random() 的返回值直接传入 encrypt()，未赋值给变量         │
│       └─ 明文在 encrypt() 调用后立即丢失，未向上层返回                  │
│       └─ ApiController@store() 重定向，不含明文                        │
│                                                                        │
│  展示阶段：                                                             │
│  管理端列表页 GET /admin/api                                            │
│    └─ 查询所有 TYPE_APPLICATION 的 Key                                  │
│    └─ 条件 Auth::user()->is($key->user)                                │
│       ├─ true → decrypt($key->token) → 显示完整明文（创建者本人）       │
│       └─ false → identifier + '****'（其他管理员）                     │
│                                                                        │
│  ✅ 最终结论：                                                          │
│     创建者每次访问列表页都能看到完整明文（实时解密）                     │
│     其他管理员只能看到 identifier                                       │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────────┐
│           Client Key (ptlc_) 明文生命周期 — 确认版                       │
├────────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  签发阶段：                                                             │
│  HasAccessTokens::createToken()                                         │
│    └─ encrypt($plain = Str::random(ApiKey::KEY_LENGTH))                │
│       └─ $plain = 捕获明文（赋值表达式语法）                            │
│    └─ return new NewAccessToken($token, $plain)                         │
│       └─ 明文向上层传递                                                 │
│  ApiKeyController@store()                                               │
│    └─ addMeta(['secret_token' => $token->plainTextToken])              │
│       └─ JSON 响应返回明文                                              │
│  前端 ApiKeyModal                                                       │
│    └─ 弹窗展示明文 → 关闭后 setApiKey('') → 内存清除                    │
│                                                                        │
│  列表查询：                                                             │
│  GET /api/client/account/api-keys                                       │
│    └─ ApiKeyTransformer::transform()                                    │
│       └─ 返回 identifier、description、allowed_ips、时间戳              │
│       └─ ⚠️ 不返回 token，不解密                                        │
│       └─ 没有任何端点可以再次获取明文                                   │
│                                                                        │
│  ✅ 最终结论：                                                          │
│     明文仅在创建响应的 JSON 中出现一次，之后无法再获取                   │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

### 8.3 明文可见性矩阵（最终版）

```
┌──────────────────────────────────────────────────────────────────────┐
│                      明文可见性矩阵（最终版）                         │
├───────────────────────┬──────────────────────┬──────────────────────┤
│ 场景                   │ Application Key      │ Client Key           │
│                       │ (ptla_)              │ (ptlc_)              │
├───────────────────────┼──────────────────────┼──────────────────────┤
│ 创建瞬间（服务端）     │ ⚠️ 未捕获，立即丢失   │ ✅ $plain 变量保留    │
│ 创建响应               │ ❌ 不返回（重定向）   │ ✅ meta.secret_token  │
│ 管理端列表页（创建者） │ ✅ 每次访问都可见     │ ❌ 不适用             │
│                       │   (decrypt 实时解密)  │                      │
│ 管理端列表页（他人）   │ ❌ identifier+****   │ ❌ 不适用             │
│ Client API 列表接口   │ ❌ 不适用             │ ❌ 不返回             │
│ ApiKey::findToken()   │ ✅ 解密比对（内部）   │ ✅ 解密比对（内部）   │
│ 数据库 token 列        │ ✅ 可 decrypt         │ ✅ 可 decrypt         │
│                       │ （需知道 APP_KEY）    │ （需知道 APP_KEY）    │
├───────────────────────┼──────────────────────┼──────────────────────┤
│ 明文获取难度           │ 中等（需创建者身份） │ 高（仅一次机会）      │
└───────────────────────┴──────────────────────┴──────────────────────┘
```

### 8.4 `ApiKey::$hidden` 的保护范围

文件：`app/Models/ApiKey.php:137`
```php
protected $hidden = ['token'];
```

`$hidden` 只在 `toArray()` / `toJson()` 序列化时生效：
- ✅ 保护 `ApiKeyTransformer` 等 API 响应不泄露加密后的 `token` 字符串
- ❌ **不保护** Blade 视图中的 `decrypt($key->token)` 主动解密
- ❌ **不保护** `$key->token` 直接属性访问（PHP 代码内）

---

## 九、三者连接关系：共享表 → Sanctum 绑定 → key_type 分流

### 9.1 三者的层级关系

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    API Key 协作链路的三层架构                             │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  第一层：数据层 — 共享 api_keys 表                                       │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  api_keys 表                                                      │   │
│  │  ├── id, user_id                                                  │   │
│  │  ├── key_type         → 1=ACCOUNT, 2=APPLICATION                │   │
│  │  ├── identifier       → ptlc_xxxxxx / ptla_xxxxxx               │   │
│  │  ├── token            → AES-256 加密的 32 字符                   │   │
│  │  ├── r_* 列           → 仅 TYPE_APPLICATION 使用                 │   │
│  │  └── allowed_ips/memo/时间戳 → 两者通用                           │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                              ▲                                          │
│                              │                                          │
│  第二层：认证层 — Sanctum 模型绑定                                       │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  AuthServiceProvider::boot()                                     │   │
│  │    └─ Sanctum::usePersonalAccessTokenModel(ApiKey::class)        │   │
│  │       ↓ 告诉 Sanctum 使用 ApiKey 而非默认的 PersonalAccessToken   │   │
│  │  HasAccessTokens trait (User 模型)                               │   │
│  │    ├─ tokens() → 关联 ApiKey 模型（无 key_type 过滤）             │   │
│  │    └─ createToken() → 封装 TYPE_ACCOUNT Key 的签发                │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                              ▲                                          │
│                              │                                          │
│  第三层：分流层 — key_type 授权检查                                      │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  /api/application 路径                                           │   │
│  │    ├─ AuthenticateApplicationUser → 检查 root_admin             │   │
│  │    └─ ApplicationApiRequest::authorize()                        │   │
│  │       ├─ TYPE_ACCOUNT → 放行（已由 root_admin 保证）             │   │
│  │       └─ TYPE_APPLICATION → AdminAcl::check(r_*)               │   │
│  │                                                                   │   │
│  │  /api/client 路径                                                │   │
│  │    └─ RequireClientApiKey                                        │   │
│  │       └─ TYPE_APPLICATION → 403 拒绝                             │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 9.2 第一层：共享 api_keys 表的连接细节

**User 模型的两个关联，访问同一张表但过滤不同**：

| 关联方法 | 代码位置 | 过滤条件 | 用途 |
|---------|---------|---------|------|
| `$user->apiKeys()` | `app/Models/User.php:250-254` | `where('key_type', TYPE_ACCOUNT)` | Client API 只查自己的 Account Key |
| `$user->tokens()` | `app/Models/Traits/HasAccessTokens.php:25-28` | **无过滤** | Sanctum 内部使用，查所有类型 |

```php
// User::apiKeys() — 只返回 Account Key
public function apiKeys(): HasMany
{
    return $this->hasMany(ApiKey::class)
        ->where('key_type', ApiKey::TYPE_ACCOUNT);  // ⭐ 过滤 TYPE_ACCOUNT
}

// HasAccessTokens::tokens() — 返回所有类型
public function tokens(): HasMany
{
    return $this->hasMany(Sanctum::$personalAccessTokenModel);  // ⚠️ 无过滤
    // Sanctum::$personalAccessTokenModel === ApiKey::class（由 AuthServiceProvider 设置）
}
```

**两种签发方式写入同一张表**：

| 签发方式 | 写入的 key_type | 写入 r_* 权限列 |
|---------|----------------|----------------|
| 管理端 `KeyCreationService` | `TYPE_APPLICATION` | ✅ 写入 9 个权限列 |
| Client API `HasAccessTokens::createToken()` | `TYPE_ACCOUNT` | ❌ 不写入（全部 0） |

### 9.3 第二层：Sanctum 令牌模型绑定的完整链路

**绑定入口 — `AuthServiceProvider::boot()`**（`app/Providers/AuthServiceProvider.php:22`）
```php
Sanctum::usePersonalAccessTokenModel(ApiKey::class);
```

这一行改变了 Sanctum 的以下行为：

1. **Sanctum Guard 调用 `ApiKey::findToken()`** 而非默认的 `PersonalAccessToken::findToken()`
2. **`$user->tokens()` 关联 `ApiKey` 模型**（因为 `Sanctum::$personalAccessTokenModel` 被设置为 `ApiKey`）
3. **`$request->user()->currentAccessToken()` 返回 `ApiKey` 实例**

**HasAccessTokens trait 对 Sanctum 的覆盖**（`app/Models/Traits/HasAccessTokens.php:17-43`）：

```php
trait HasAccessTokens
{
    use HasApiTokens {
        tokens as private _tokens;        // 别名 Sanctum 原始实现
        createToken as private _createToken;
    }

    // ⭐ 覆盖 tokens() — 直接关联 ApiKey（与 Sanctum 一致）
    public function tokens(): HasMany
    {
        return $this->hasMany(Sanctum::$personalAccessTokenModel);
    }

    // ⭐ 覆盖 createToken() — 强制 TYPE_ACCOUNT，自定义字段
    public function createToken(?string $memo, ?array $ips): NewAccessToken
    {
        $token = $this->tokens()->forceCreate([
            'user_id' => $this->id,
            'key_type' => ApiKey::TYPE_ACCOUNT,  // ⚠️ 硬编码 TYPE_ACCOUNT
            'identifier' => ApiKey::generateTokenIdentifier(ApiKey::TYPE_ACCOUNT),
            'token' => encrypt($plain = Str::random(ApiKey::KEY_LENGTH)),
            'memo' => $memo ?? '',
            'allowed_ips' => $ips ?? [],
        ]);
        return new NewAccessToken($token, $plain);
    }
}
```

**`ApiKey::tokenable()` 关系 — 反向关联**（`app/Models/ApiKey.php:192-195`）：
```php
public function tokenable(): BelongsTo
{
    return $this->user();  // Sanctum 要求 tokenable 关系，直接返回 user()
}
```

Sanctum 通过 `$token->tokenable` 获取 Token 所属的用户模型，完成认证流程。

### 9.4 第三层：key_type 分流机制的完整执行路径

**`findToken()` 不检查 key_type**（`app/Models/ApiKey.php:200-210`）：
```php
public static function findToken(string $token): ?self
{
    $identifier = substr($token, 0, self::IDENTIFIER_LENGTH);
    $model = static::where('identifier', $identifier)->first();
    if (!is_null($model) && decrypt($model->token) === substr($token, strlen($identifier))) {
        return $model;  // ⚠️ 返回 ApiKey 实例，key_type 可以是任何有效值
    }
    return null;
}
```

**key_type 检查全部在下游执行**：

```
Bearer Token 到达
    │
    ▼
auth:sanctum → ApiKey::findToken() → 返回 ApiKey（不检查 key_type）
    │
    ├─ /api/application 路径
    │    │
    │    ▼
    │  AuthenticateApplicationUser
    │    └─ 只检查 user->root_admin → ⚠️ 不检查 key_type
    │    │
    │    ▼
    │  ApplicationApiRequest::authorize()
    │    ├─ TransientToken → 放行
    │    ├─ key_type === TYPE_ACCOUNT → 放行（已由 root_admin 保证）
    │    └─ key_type === TYPE_APPLICATION → AdminAcl::check(r_{resource})
    │
    └─ /api/client 路径
         │
         ▼
       RequireClientApiKey
         └─ key_type === TYPE_APPLICATION → 403 拒绝 ❌
         └─ key_type === TYPE_ACCOUNT → 放行 ✅
         └─ TransientToken → 放行 ✅
         │
         ▼
       ClientApiRequest::authorize()
         └─ ServerPolicy 校验（不检查 key_type）
```

### 9.5 管理端签发结果接入两条 API 路径的完整流程

**管理端签发 Application Key**：
```
POST /admin/api/new（Web 表单）
    → web 中间件组 + AdminAuthenticate
    → KeyCreationService::handle()
        → setKeyType(TYPE_APPLICATION)
        → 写入 api_keys 表：key_type=2, r_servers=*, r_nodes=*, ...
    → 重定向到列表页
        → 视图 decrypt($key->token) 展示完整明文（仅创建者）
```

**Application Key 接入 Application API**：
```
Bearer ptla_xxxxxxxxxxxxxxyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyy
    → auth:sanctum → ApiKey::findToken()
        → identifier 查表 → 找到记录
        → decrypt(token) 比对 → 匹配成功
        → 返回 ApiKey{key_type=2, r_servers=3, ...}
    → AuthenticateApplicationUser
        → 检查 user->root_admin → 通过
    → ApplicationApiRequest::authorize()
        → key_type === TYPE_APPLICATION → 是
        → AdminAcl::check($token, 'servers', READ)
            → $permission = $token->r_servers = 3
            → $action = 1
            → ($permission & $action) = 1 → 非零 → 通过
    → 控制器执行
```

**Application Key 被 Client API 拒绝**：
```
Bearer ptla_xxxxxxxxxxxxxxyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyy
    → auth:sanctum → ApiKey::findToken() → 返回 ApiKey{key_type=2}
    → RequireClientApiKey
        → $token instanceof ApiKey → true
        → $token->key_type === TYPE_APPLICATION → true
        → throw AccessDeniedHttpException → 403
```

**Account Key 接入 Application API**（单向渗透）：
```
Bearer ptlc_xxxxxxxxxxxxxxyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyy
    → auth:sanctum → ApiKey::findToken() → 返回 ApiKey{key_type=1}
    → AuthenticateApplicationUser
        → 检查 user->root_admin → 通过
    → ApplicationApiRequest::authorize()
        → key_type === TYPE_ACCOUNT → 是
        → 直接 return true → 放行（不检查 AdminAcl）
        → ⚠️ 即使 r_* 权限列全为 0，也能访问所有资源
```

### 9.6 为什么不把 key_type 检查放在 findToken()？

这是**关注点分离**的设计原则：

- **认证（Authentication）** = "这个 Token 有效吗？" → `findToken()` 的职责
  - 只验证 Token 的有效性（查表、解密、比对）
  - 不关心 Token 能访问什么

- **授权（Authorization）** = "这个 Token 能访问什么？" → 中间件 + Request 层的职责
  - 检查 key_type
  - 检查 root_admin
  - 检查 AdminAcl 或 ServerPolicy
  - 可以根据路径灵活调整规则

如果 `findToken()` 检查 `key_type`，就会把授权逻辑混入认证层，导致：
1. 同一个 Token 无法在不同路径获得不同授权结果
2. 认证层与业务逻辑强耦合
3. 难以调整授权规则而不影响认证逻辑

### 9.7 identifier 前缀的作用

`ptlc_` / `ptla_` 前缀（`app/Models/ApiKey.php:215-230`）：

```php
public static function generateTokenIdentifier(int $type): string
{
    $prefix = $type === self::TYPE_ACCOUNT ? 'ptlc_' : 'ptla_';
    return $prefix . Str::random(self::IDENTIFIER_LENGTH - strlen($prefix));
}
```

**前缀不参与任何分流逻辑**。系统在所有检查点判断的是 `key_type` 数值字段（1 或 2），而非字符串前缀。

前缀的实际作用：
1. **人类可读性**：在日志、数据库、错误消息中一眼识别 Key 类型
2. **查表加速**：前缀是 identifier 的一部分，用于快速定位记录
3. **错误提示**：`RequireClientApiKey` 的错误消息明确指出 "application API key on an endpoint that requires a client API key"，用户能立刻理解问题所在
