# 数据库凭据管理：主机选择、密码下发与节点回写的关联流程

## 1. 核心数据模型与关系

### 1.1 三张关键表

| 表名 | 关键字段 | 说明 |
|---|---|---|
| `nodes` | `id`, `fqdn`, `daemonListen` | 运行服务器实例的物理/虚拟节点 |
| `database_hosts` | `id`, `host`, `port`, `username`, `password`, `node_id` | 数据库主机（MySQL 实例），通过 `node_id` 关联到节点 |
| `databases` | `id`, `server_id`, `database_host_id`, `database`, `username`, `password`, `remote` | 具体的数据库及访问凭据，通过 `database_host_id` 关联到数据库主机 |

### 1.2 关系链路

```
Node (1) ←── (N) DatabaseHost (1) ←── (N) Database (N) → (1) Server
```

- `DatabaseHost.node_id` → `Node.id`（一个数据库主机可选地绑定到一个节点，外键 `ON DELETE SET NULL`）
- `Database.database_host_id` → `DatabaseHost.id`（一个数据库必须属于一个数据库主机）
- `Database.server_id` → `Server.id`（一个数据库属于一个服务器）

## 2. 主机选择（Host Selection）

### 2.1 入口

客户端通过 API `POST /api/client/servers/{server}/databases` 创建数据库时，由 `DatabaseController::store()` 调用 `DeployServerDatabaseService::handle()`。

### 2.2 选择逻辑

文件：`app/Services/Databases/DeployServerDatabaseService.php`

```
1. 获取所有 DatabaseHost 记录
2. 如果没有任何主机 → 抛出 NoSuitableDatabaseHostException
3. 从所有主机中筛选 node_id == server.node_id 的主机（同节点主机）
4. 如果同节点主机不为空 → 从中随机选择一台
5. 如果同节点主机为空：
   a. 若配置 allow_random = true → 从所有主机中随机选择
   b. 若配置 allow_random = false → 抛出 NoSuitableDatabaseHostException
```

### 2.3 配置项

文件：`config/pterodactyl.php`

- `client_features.databases.enabled`：是否允许客户端创建数据库
- `client_features.databases.allow_random`：当服务器所在节点没有数据库主机时，是否允许随机分配其他节点的主机

### 2.4 核心代码

```php
// DeployServerDatabaseService.php:30-44
$hosts = DatabaseHost::query()->get()->toBase();
$nodeHosts = $hosts->where('node_id', $server->node_id)->toBase();

'database_host_id' => $nodeHosts->isEmpty()
    ? $hosts->random()->id      // 随机选择任意主机
    : $nodeHosts->random()->id, // 优先选择同节点主机
```

## 3. 密码下发（Password Provisioning）

密码下发分为两个场景：**创建数据库时自动生成** 和 **轮换密码时重新生成**。

### 3.1 创建数据库时的密码生成

文件：`app/Services/Databases/DatabaseManagementService.php`

当 `DeployServerDatabaseService` 选定主机后，调用 `DatabaseManagementService::create()`，流程如下：

```
1. 检查客户端数据库功能是否启用
2. 检查服务器数据库数量是否超过限制
3. 生成数据库用户名：u{server_id}_{random_10_chars}
4. 生成随机密码（24位含特殊字符）→ 加密后存入 databases 表
5. 建立 DynamicDatabaseConnection（动态连接到选定的数据库主机）
6. 在远程 MySQL 上执行：
   a. CREATE DATABASE
   b. CREATE USER（使用明文密码）
   c. GRANT 权限
   d. FLUSH PRIVILEGES
```

关键代码：

```php
// DatabaseManagementService.php:90-114
$data = array_merge($data, [
    'server_id' => $server->id,
    'username' => sprintf('u%d_%s', $server->id, str_random(10)),
    'password' => $this->encrypter->encrypt(
        Utilities::randomStringWithSpecialCharacters(24)
    ),
]);

$this->dynamic->set('dynamic', $data['database_host_id']);
$this->repository->createDatabase($database->database);
$this->repository->createUser(
    $database->username,
    $database->remote,
    $this->encrypter->decrypt($database->password),  // 解密后明文传给 MySQL
    $database->max_connections
);
$this->repository->assignUserToDatabase(...);
$this->repository->flush();
```

### 3.2 密码轮换

文件：`app/Services/Databases/DatabasePasswordService.php`

客户端通过 API `POST /api/client/servers/{server}/databases/{database}/rotate-password` 触发：

```
1. 生成新的 24 位随机密码
2. 加密后更新 databases 表中的 password 字段
3. 建立 DynamicDatabaseConnection 连接到数据库主机
4. 在远程 MySQL 上：
   a. DROP USER 旧用户
   b. CREATE USER 新用户（使用新密码）
   c. GRANT 权限
   d. FLUSH PRIVILEGES
5. 返回明文密码给客户端
```

### 3.3 DynamicDatabaseConnection 的作用

文件：`app/Extensions/DynamicDatabaseConnection.php`

这是连接 Panel 数据库和远程 MySQL 主机的桥梁：

```php
public function set(string $connection, DatabaseHost|int $host, string $database = 'mysql'): void
{
    $this->config->set('database.connections.' . $connection, [
        'driver'    => 'mysql',
        'host'      => $host->host,
        'port'      => $host->port,
        'database'  => 'mysql',
        'username'  => $host->username,       // DatabaseHost 的管理账号
        'password'  => $this->encrypter->decrypt($host->password),  // 解密管理密码
        'charset'   => 'utf8',
        'collation' => 'utf8_unicode_ci',
    ]);
}
```

它使用 `DatabaseHost` 自身存储的管理员凭据（username/password）连接到远程 MySQL，然后通过该连接执行 CREATE USER / GRANT 等 DDL 操作。

### 3.4 密码如何返回给客户端

文件：`app/Transformers/Api/Client/DatabaseTransformer.php`

密码不会在默认的 `transform()` 中返回，只有当 API 请求显式 `include=password` 且用户拥有 `database.view-password` 权限时才返回：

```php
public function includePassword(Database $database): Item|NullResource
{
    if (!$this->request->user()->can(Permission::ACTION_DATABASE_VIEW_PASSWORD, $database->server)) {
        return $this->null();
    }
    return $this->item($database, function (Database $model) {
        return ['password' => $this->encrypter->decrypt($model->password)];
    }, 'database_password');
}
```

## 4. 节点回写（Node Write-back）

"节点回写"指的是 `DatabaseHost.node_id` 字段的设置。这个字段将数据库主机与运行服务器的工作节点关联起来。

### 4.1 创建 DatabaseHost 时设置 node_id

文件：`app/Services/Databases/Hosts/HostCreationService.php`

管理员在后台创建数据库主机时，通过表单提交 `node_id`，该值被直接写入 `database_hosts` 表：

```php
$host = $this->repository->create([
    'password' => $this->encrypter->encrypt(array_get($data, 'password')),
    'name'     => array_get($data, 'name'),
    'host'     => array_get($data, 'host'),
    'port'     => array_get($data, 'port'),
    'username' => array_get($data, 'username'),
    'max_databases' => null,
    'node_id'  => array_get($data, 'node_id'),  // 节点回写
]);
```

表单请求 `DatabaseHostFormRequest` 中，如果 `node_id` 未填写，会被设为 `null`：

```php
if (!$this->filled('node_id')) {
    $this->merge(['node_id' => null]);
}
```

### 4.2 更新 DatabaseHost 时设置 node_id

文件：`app/Services/Databases/Hosts/HostUpdateService.php`

管理员修改数据库主机信息时，`node_id` 可被更新。更新后同样会验证连接可用性。

### 4.3 node_id 的设计意图

- `node_id` 是 **可选字段**（nullable），表示该数据库主机"偏好"绑定到某个节点
- 数据库主机的 MySQL 实例不一定与节点在同一台物理机上
- 绑定 `node_id` 的目的是在主机选择时提供优先匹配依据（同节点优先）
- 数据库 schema 中，外键约束为 `ON DELETE SET NULL`，即节点被删除时，`node_id` 被置空而不影响数据库主机记录

## 5. 三者配合的完整流程

### 5.1 管理员配置阶段

```
管理员创建 Node（节点）
       │
       ▼
管理员创建 DatabaseHost，指定 node_id（节点回写）
       │  → 密码加密存储到 database_hosts.password
       │  → 使用 DynamicDatabaseConnection 验证连接
       │
       ▼
管理员创建 Server，绑定到某个 Node
```

### 5.2 客户端创建数据库阶段

```
客户端请求创建数据库
       │
       ▼
DatabaseController::store()
       │
       ▼
DeployServerDatabaseService::handle($server, $data)
       │
       ├─① 主机选择：根据 server.node_id 查找同节点的 DatabaseHost
       │   ├─ 找到同节点主机 → 随机选一台
       │   └─ 未找到同节点主机：
       │       ├─ allow_random=true → 从所有主机随机选
       │       └─ allow_random=false → 抛异常
       │
       ▼
DatabaseManagementService::create($server, $data)
       │
       ├─② 密码下发：
       │   ├─ 生成随机密码 → 加密存入 databases.password
       │   ├─ DynamicDatabaseConnection 使用 DatabaseHost 管理凭据连接远程 MySQL
       │   ├─ 在远程 MySQL 执行：CREATE DATABASE + CREATE USER + GRANT + FLUSH
       │   └─ 密码通过 Transformer 解密后返回给客户端
       │
       ▼
Database 创建完成
```

### 5.3 客户端轮换密码阶段

```
客户端请求轮换密码
       │
       ▼
DatabaseController::rotatePassword()
       │
       ▼
DatabasePasswordService::handle($database)
       │
       ├─ 生成新随机密码
       ├─ 加密后更新 databases.password
       ├─ DynamicDatabaseConnection 连接到 database.database_host_id 对应的 DatabaseHost
       ├─ 在远程 MySQL：DROP USER + CREATE USER + GRANT + FLUSH
       └─ 返回明文新密码给客户端
```

## 6. 关键协作机制总结

| 机制 | 参与组件 | 说明 |
|---|---|---|
| 主机选择 | `DeployServerDatabaseService` + `DatabaseHost.node_id` + `Server.node_id` | 通过 node_id 匹配优先选择同节点数据库主机 |
| 密码下发 | `DatabaseManagementService` / `DatabasePasswordService` + `DynamicDatabaseConnection` + `DatabaseHost` 管理凭据 | 使用 DatabaseHost 的管理员账号连接远程 MySQL，创建/修改数据库用户并设置密码 |
| 节点回写 | `HostCreationService` / `HostUpdateService` + `DatabaseHostFormRequest` | 管理员在创建/编辑数据库主机时设置 node_id，为主机选择提供匹配依据 |
| 凭据加密 | `Encrypter`（Laravel） | 所有密码在数据库中加密存储，使用时解密；DatabaseHost.password 是管理凭据，Database.password 是用户凭据 |

### 6.1 核心关联

**node_id 是串联主机选择与节点回写的桥梁**：

1. **写入方向**（节点回写）：管理员创建 DatabaseHost 时设置 `node_id` → 标记该数据库主机属于哪个节点
2. **读取方向**（主机选择）：客户端创建数据库时，`DeployServerDatabaseService` 用 `server.node_id` 匹配 `DatabaseHost.node_id` 来优先选择同节点主机

**DynamicDatabaseConnection 是串联主机选择与密码下发的桥梁**：

1. 主机选择确定 `database_host_id` 后
2. `DynamicDatabaseConnection` 读取该 DatabaseHost 的管理凭据（host/port/username/password）
3. 建立到远程 MySQL 的动态连接
4. 通过该连接执行密码下发操作（CREATE USER / GRANT 等）

### 6.2 两类密码的区别

| | DatabaseHost.password | Database.password |
|---|---|---|
| 用途 | 连接远程 MySQL 的管理员凭据 | 数据库最终用户的访问密码 |
| 设置者 | 管理员创建/编辑主机时手动输入 | 系统自动生成（24位随机字符串） |
| 使用场景 | DynamicDatabaseConnection 建立管理连接 | 远程 MySQL CREATE USER，客户端连接数据库 |
| 存储位置 | `database_hosts.password` | `databases.password` |
| 加密方式 | Laravel Encrypter 加密 | Laravel Encrypter 加密 |
