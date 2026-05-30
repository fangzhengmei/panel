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

## 八、明文展示机制的代码级分析

### 8.1 两种 Key 的明文生命周期对比

结论先行：**Application Key 的明文并非"仅创建后可见"，而是创建者每次访问列表页都能看到。Client Key 的明文确实仅创建响应返回一次。** 这是一个容易误解的关键差异。

```
┌────────────────────────────────────────────────────────────────────────┐
│               Application Key (ptla_) 明文生命周期                      │
├────────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  KeyCreationService::handle()                                         │
│    └─ $this->encrypter->encrypt(str_random(ApiKey::KEY_LENGTH))       │
│       └─ 生成 32 字符随机明文 → Laravel encrypt() AES-256-CBC 加密    │
│       └─ 加密密文写入 api_keys.token 列                                │
│       └─ ⚠️ 明文变量在函数结束后丢弃，未向上层返回                      │
│                                                                        │
│  展示方式：列表页视图实时解密                                           │
│    └─ resources/views/admin/api/index.blade.php:39                     │
│       └─ {{ $key->identifier . decrypt($key->token) }}                │
│       └─ 每次渲染页面时，对 token 列执行 decrypt() 解密                │
│       └─ 条件：Auth::user()->is($key->user) → 仅创建者可见             │
│                                                                        │
│  ⚠️ 明文可反复查看：只要创建者登录管理后台访问列表页，                  │
│     decrypt($key->token) 每次都能还原出完整的 32 字符明文               │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────────┐
│               Client Key (ptlc_) 明文生命周期                           │
├────────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  HasAccessTokens::createToken()                                        │
│    └─ encrypt($plain = Str::random(ApiKey::KEY_LENGTH))               │
│       └─ 生成 32 字符随机明文 $plain → encrypt() 加密                  │
│       └─ 加密密文写入 api_keys.token 列                                │
│    └─ return new NewAccessToken($token, $plain);                       │
│       └─ ⭐ 明文 $plain 被包装进 NewAccessToken 对象向上层返回         │
│                                                                        │
│  ApiKeyController::store()                                             │
│    └─ $token = $request->user()->createToken(...)                      │
│    └─ ->addMeta(['secret_token' => $token->plainTextToken])            │
│       └─ 明文作为 JSON meta 字段返回给前端                             │
│       └─ ⭐ 这是明文最后一次出现在服务端响应中                         │
│                                                                        │
│  前端展示：ApiKeyModal 组件                                             │
│    └─ resources/scripts/components/dashboard/ApiKeyModal.tsx           │
│       └─ "Please store this in a safe location, it will not be        │
│          shown again."                                                 │
│       └─ setApiKey(`${key.identifier}${secretToken}`)                  │
│       └─ 拼接 identifier + secretToken 作为完整 Key 展示               │
│       └─ 关闭弹窗后 setApiKey('') → 明文从内存中清除                   │
│                                                                        │
│  列表查询：ApiKeyController::index()                                   │
│    └─ ApiKeyTransformer::transform()                                   │
│       └─ 只返回 identifier、description、allowed_ips、时间戳            │
│       └─ ⚠️ 不返回 token，不返回 secret_token                          │
│       └─ 没有任何端点可以再次获取明文                                   │
│                                                                        │
│  ✅ 明文确实仅可见一次：创建响应 + 弹窗展示，之后无法再获取             │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

### 8.2 为什么 Application Key 能反复查看而 Client Key 不能？

**根本原因在于 `KeyCreationService` 没有返回明文**。

对比两种签发方式的核心差异：

**Application Key — `KeyCreationService::handle()`**
```php
// app/Services/Api/KeyCreationService.php:43
'token' => $this->encrypter->encrypt(str_random(ApiKey::KEY_LENGTH)),
//         ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
// str_random() 的返回值直接传给 encrypt()，没有赋值给变量
// 明文在 encrypt() 调用后立即丢失，无法向上层传递
```

`handle()` 的返回值是 `ApiKey` 模型实例（`$this->repository->create()`），**不含明文**。控制器 `ApiController@store` 拿到 `ApiKey` 后直接重定向，明文在服务端已经不存在。

但管理端列表页通过 `decrypt($key->token)` **实时解密**恢复明文。这是因为 Laravel 的 `encrypt()` 是对称加密（AES-256-CBC），只要有加密密钥（`APP_KEY`）就能解密。管理端视图在服务端执行，天然拥有解密能力。

**Client Key — `HasAccessTokens::createToken()`**
```php
// app/Models/Traits/HasAccessTokens.php:37
'token' => encrypt($plain = Str::random(ApiKey::KEY_LENGTH)),
//                  ^^^^^ ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
//                  $plain 变量捕获了随机字符串的明文
```

这里使用了 PHP 的赋值表达式内嵌语法 `$plain = Str::random(...)`，`$plain` 捕获了明文并向上传递：

```
createToken() → NewAccessToken($token, $plain)
  → ApiKeyController@store → $token->plainTextToken
    → JSON meta.secret_token
              → 前端 ApiKeyModal 展示
```

**Client API 的列表接口为什么不解密返回？**

文件：`app/Transformers/Api/Client/ApiKeyTransformer.php:17-25`

```php
public function transform(ApiKey $model): array
{
    return [
        'identifier' => $model->identifier,
        'description' => $model->memo,
        'allowed_ips' => $model->allowed_ips,
        'last_used_at' => $model->last_used_at ? $model->last_used_at->toAtomString() : null,
        'created_at' => $model->created_at->toAtomString(),
    ];
}
```

Transformer 返回的字段中**没有 `token`**（`ApiKey` 模型的 `$hidden = ['token']` 也阻止了序列化泄露），更没有 `decrypt($model->token)`。这是因为：

1. Client API 是无状态的 REST 接口，面向第三方应用和前端 SPA
2. 列表接口的消费者不是"Key 创建者本人"（请求可能来自任何持有有效 Token 的请求方）
3. 如果列表接口返回解密后的明文，任何有权限调用列表接口的人都能看到所有 Key 的明文，这是严重的安全风险

而管理端列表页是服务端渲染的 Blade 视图，运行在服务端，通过 `Auth::user()->is($key->user)` 严格限制只有创建者本人才能看到 `decrypt($key->token)`。

### 8.3 明文可见性的安全边界

```
┌──────────────────────────────────────────────────────────────────┐
│                    明文可见性矩阵                                 │
├───────────────────────┬──────────────┬──────────────────────────┤
│ 场景                   │ Application  │ Client                   │
│                       │ Key (ptla_)  │ Key (ptlc_)              │
├───────────────────────┼──────────────┼──────────────────────────┤
│ 创建瞬间（服务端）     │ ⚠️ 丢失      │ ✅ 保留在 NewAccessToken │
│ 创建响应               │ ❌ 不返回     │ ✅ meta.secret_token     │
│ 管理端列表页（创建者） │ ✅ 可反复看   │ ❌ 不适用                 │
│ 管理端列表页（他人）   │ ❌ identifier │ ❌ 不适用                 │
│                       │    + ****    │                          │
│ Client API 列表接口   │ ❌ 不适用     │ ❌ 不返回                 │
│ ApiKey::findToken()   │ ✅ 可解密    │ ✅ 可解密（仅用于鉴权比对）│
│ 数据库直接读取 token 列│ ✅ 可 decrypt│ ✅ 可 decrypt             │
├───────────────────────┼──────────────┼──────────────────────────┤
│ 整体安全性             │ 较弱         │ 较强                     │
│ （明文获取难度）       │              │                          │
└───────────────────────┴──────────────┴──────────────────────────┘
```

**`ApiKey::$hidden = ['token']`**（`app/Models/ApiKey.php:137`）确保了 `token` 列不会出现在 `toArray()` / `toJson()` 输出中。这防止了以下场景的意外泄露：

- Client API 的 `ApiKeyTransformer` 不会泄露加密密文
- 任何将 `ApiKey` 模型转为 JSON 的操作（如 API 响应）都不会包含 `token`

但这**不能**防止管理端 Blade 视图中的 `decrypt($key->token)` 主动解密展示。

---

## 九、Sanctum 令牌模型与 key_type 分流机制

### 9.1 Sanctum 如何接入 ApiKey 模型

文件：`app/Providers/AuthServiceProvider.php:22`

```php
Sanctum::usePersonalAccessTokenModel(ApiKey::class);
```

这一行是整个体系的连接点。Sanctum 原本使用自己的 `PersonalAccessToken` 模型，Pterodactyl 将其替换为 `ApiKey`。这意味着：

1. Sanctum Guard 在执行 `auth:sanctum` 时，会调用 `ApiKey::findToken()` 而非默认的 `PersonalAccessToken::findToken()`
2. 认证成功后，`$request->user()->currentAccessToken()` 返回的是 `ApiKey` 实例
3. `ApiKey` 模型必须实现 `Laravel\Sanctum\Contracts\HasAbilities` 接口（已实现，但 `can()` 始终返回 false）
4. `ApiKey` 模型必须有 `tokenable()` 关系方法（已实现，返回 `$this->user()`）

### 9.2 findToken() 的鉴权流程与 key_type 无关

文件：`app/Models/ApiKey.php:200-210`

```php
public static function findToken(string $token): ?self
{
    $identifier = substr($token, 0, self::IDENTIFIER_LENGTH);
    $model = static::where('identifier', $identifier)->first();
    if (!is_null($model) && decrypt($model->token) === substr($token, strlen($identifier))) {
        return $model;
    }
    return null;
}
```

**关键发现**：`findToken()` **不检查 `key_type`**。它只做两件事：
1. 用 `identifier`（前 16 字符）查表
2. 解密 `token` 列并与请求中剩余部分比对

这意味着无论是 `ptlc_`（TYPE_ACCOUNT）还是 `ptla_`（TYPE_APPLICATION）的 Key，都能通过 `auth:sanctum` 认证。**key_type 的分流不在 Sanctum 认证层，而在下游的分流中间件和 Request 层**。

### 9.3 key_type 分流的三层检查点

`key_type` 在三个不同的层级被检查，每层的作用不同：

```
请求通过 auth:sanctum（不检查 key_type）
       │
       ▼
┌──────────────────────────────────────────────────────────────┐
│ 第一层：中间件分流（决定请求能否进入对应的 API 路径）          │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  /api/application 路径                                       │
│    └─ AuthenticateApplicationUser                           │
│       └─ 只检查 user->root_admin                             │
│       └─ ⚠️ 不检查 key_type                                 │
│       └─ TYPE_APPLICATION 和 TYPE_ACCOUNT 都能通过           │
│                                                              │
│  /api/client 路径                                            │
│    └─ RequireClientApiKey                                    │
│       └─ 检查 key_type === TYPE_APPLICATION → 403            │
│       └─ ⚠️ 这是唯一在中间件层检查 key_type 的地方           │
│       └─ TYPE_ACCOUNT 和 TransientToken 能通过               │
│                                                              │
└──────────────────────────────────────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────────────────────────────┐
│ 第二层：Request 授权（决定请求能否访问特定资源）               │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  ApplicationApiRequest::authorize()                          │
│    ├─ TransientToken     → 放行                              │
│    ├─ TYPE_ACCOUNT       → 放行（已由 root_admin 保证）       │
│    └─ TYPE_APPLICATION   → AdminAcl::check(r_{resource})     │
│       └─ ⭐ key_type 决定了是否需要 AdminAcl 细粒度校验      │
│                                                              │
│  ClientApiRequest::authorize()                               │
│    └─ 不检查 key_type（由中间件层保证了不会有 TYPE_APPLICATION）│
│    └─ 检查 ServerPolicy（Owner 或 Subuser 权限）              │
│                                                              │
└──────────────────────────────────────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────────────────────────────┐
│ 第三层：Transformer 授权（决定关联资源能否被序列化）           │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  BaseTransformer::authorize($resource)                       │
│    ├─ TYPE_ACCOUNT     → 检查 user->root_admin               │
│    └─ TYPE_APPLICATION → AdminAcl::check($token, $resource)  │
│                                                              │
│  BaseClientTransformer::authorize($ability, $server)         │
│    └─ user()->can($ability, [$server])                       │
│    └─ 不检查 key_type                                        │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### 9.4 管理端签发结果如何接入两条 API 路径

核心问题是：管理端通过 `KeyCreationService` 签发的 `TYPE_APPLICATION` Key，是如何被 Application API 和 Client API 两条路径分别接纳或拒绝的？

**接入 Application API 路径**：

```
管理端签发 KeyCreationService::handle()
  → 写入 api_keys 表，key_type = TYPE_APPLICATION
  → 写入 r_servers/r_nodes/... 等 9 个权限列

使用时：
  Bearer ptla_xxxxxxxxxxxxxxyyyyyy...
    → auth:sanctum → ApiKey::findToken() → 返回 ApiKey{key_type=2}
    → AuthenticateApplicationUser → 检查 user->root_admin → 通过
    → ApplicationApiRequest::authorize()
      → key_type === TYPE_APPLICATION → AdminAcl::check()
      → 读取 r_servers 等 9 个权限列 → 位运算校验 → 通过/拒绝
```

**被 Client API 路径拒绝**：

```
使用时：
  Bearer ptla_xxxxxxxxxxxxxxyyyyyy...
    → auth:sanctum → ApiKey::findToken() → 返回 ApiKey{key_type=2}
    → RequireClientApiKey
      → $token instanceof ApiKey → true
      → $token->key_type === TYPE_APPLICATION → true
      → 抛出 AccessDeniedHttpException → 403
```

**反向：Client Key 接入 Application API 路径**：

```
客户端签发 HasAccessTokens::createToken()
  → 写入 api_keys 表，key_type = TYPE_ACCOUNT
  → r_* 权限列全部为 0（NONE）

使用时：
  Bearer ptlc_xxxxxxxxxxxxxxyyyyyy...
    → auth:sanctum → ApiKey::findToken() → 返回 ApiKey{key_type=1}
    → AuthenticateApplicationUser → 检查 user->root_admin → 通过
    → ApplicationApiRequest::authorize()
      → key_type === TYPE_ACCOUNT → 直接放行（不检查 AdminAcl）
      → ⚠️ 即使 r_* 权限列全为 0，也能访问所有 Application API 资源
```

### 9.5 key_type 分流的完整决策树

```
ApiKey::findToken($bearerToken)
  │
  ├─ 返回 null → 401 Unauthorized
  │
  └─ 返回 ApiKey 实例
       │
       ├─ key_type === TYPE_ACCOUNT (1, ptlc_)
       │     │
       │     ├─ /api/application 路径
       │     │     ├─ AuthenticateApplicationUser: root_admin? → 否则 403
       │     │     └─ ApplicationApiRequest::authorize()
       │     │           └─ TYPE_ACCOUNT → 放行（不查 AdminAcl）
       │     │                 → ⚠️ 拥有全部 Application API 权限
       │     │
       │     └─ /api/client 路径
       │           ├─ RequireClientApiKey: 不是 TYPE_APPLICATION → 放行
       │           └─ ClientApiRequest::authorize()
       │                 └─ ServerPolicy 校验
       │
       ├─ key_type === TYPE_APPLICATION (2, ptla_)
       │     │
       │     ├─ /api/application 路径
       │     │     ├─ AuthenticateApplicationUser: root_admin? → 否则 403
       │     │     └─ ApplicationApiRequest::authorize()
       │     │           └─ TYPE_APPLICATION → AdminAcl::check()
       │     │                 → 读取 r_{resource} → 位运算校验
       │     │
       │     └─ /api/client 路径
       │           └─ RequireClientApiKey: TYPE_APPLICATION → 403 ❌
       │
       └─ TransientToken（前端 SPA Cookie 认证）
             │
             ├─ /api/application 路径
             │     ├─ AuthenticateApplicationUser: root_admin? → 否则 403
             │     └─ ApplicationApiRequest::authorize()
             │           └─ TransientToken → 放行
             │
             └─ /api/client 路径
                   ├─ RequireClientApiKey: 不是 ApiKey 实例 → 放行
                   └─ ClientApiRequest::authorize()
                         └─ ServerPolicy 校验
```

### 9.6 identifier 前缀在分流中的作用

文件：`app/Models/ApiKey.php:215-230`

```php
public static function getPrefixForType(int $type): string
{
    Assert::oneOf($type, [self::TYPE_ACCOUNT, self::TYPE_APPLICATION]);
    return $type === self::TYPE_ACCOUNT ? 'ptlc_' : 'ptla_';
}

public static function generateTokenIdentifier(int $type): string
{
    $prefix = self::getPrefixForType($type);
    return $prefix . Str::random(self::IDENTIFIER_LENGTH - strlen($prefix));
}
```

`ptlc_` 和 `ptla_` 前缀占据了 `identifier` 的前 5 个字符（identifier 总长 16 字符）。**前缀仅用于人类可读性，不参与任何分流逻辑**。系统在所有分流点检查的是 `key_type` 数值字段（1 或 2），而非 identifier 的前缀字符串。

但前缀有间接的安全价值：管理员在日志或数据库中看到 `ptla_` 前缀就能立刻识别这是 Application Key，不应在 Client API 中使用。`findToken()` 虽然不检查前缀，但前缀是 identifier 的一部分，用于查表定位。

### 9.7 findToken() 为什么不检查 key_type？

这是有意的设计选择。`findToken()` 的职责是**认证**（"这个 Token 有效吗？"），而非**授权**（"这个 Token 能访问什么？"）。将授权逻辑分离到中间件和 Request 层，使得：

1. 认证层保持简洁，只做 Token 有效性验证
2. 授权层可以根据路径（`/api/application` vs `/api/client`）灵活决定是否允许
3. 同一个 Token 可以在不同路径上得到不同的授权结果（如 TYPE_ACCOUNT 在两条路径上都可使用）

如果 `findToken()` 检查了 `key_type`，就会把授权逻辑混入认证层，违反关注点分离原则。
