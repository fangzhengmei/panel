# 游戏面板备份创建与还原协作分析

本文档梳理 Pterodactyl 游戏面板中备份创建和还原两条路径的职责切分、失败语义和权限边界。

## 架构概览

系统由三方协作完成备份生命周期：

| 组件 | 角色 | 核心职责 |
|------|------|----------|
| **主控（Panel）** | 编排者 | API 入口、权限校验、元数据管理、事务协调 |
| **守护进程（Wings）** | 执行者 | 实际文件打包/解压、本地存储、S3 分片上传 |
| **对象存储（S3）** | 持久化 | 备份文件最终存储（可选，Wings 本地磁盘为备选） |

---

## 一、备份创建流程分析

### 1.1 流程时序

```
用户 → Panel API → InitiateBackupService → DaemonBackupRepository → Wings
                                                          ↓
Wings 回调 → BackupStatusController → Panel DB 更新
                                                          ↓
                              （S3 场景）Wings → BackupRemoteUploadController → S3 分片上传
```

### 1.2 各阶段职责切分

#### 阶段 1：客户端 API 入口
**文件**：`app/Http/Controllers/Api/Client/Servers/BackupController.php:67-96`

**职责**：
- 权限校验：`Permission::ACTION_BACKUP_CREATE`
- 参数解析：`name`、`is_locked`、`ignored`
- 活动日志记录
- 锁表防止并发超限

**关键权限规则**（`BackupController.php:76-78`）：
- 仅拥有 `backup.delete` 权限的用户才能设置 `is_locked`
- 无删除权限的用户请求锁定会被静默忽略

#### 阶段 2：业务逻辑编排
**文件**：`app/Services/Backups/InitiateBackupService.php:76-126`

**职责**：
1. **限流检查**（`backups.throttles` 配置）：默认 10 分钟内最多 2 个备份
2. **数量限制检查**：对比服务器 `backup_limit`
3. **自动轮换**：超限且 `override=true` 时删除最旧未锁定备份
4. **数据库事务**：在同一事务中创建备份记录 + 调用 Wings
5. **适配器选择**：通过 `BackupManager` 获取默认存储适配器

**失败语义**：
- 限流/超限：立即抛出 `TooManyRequestsHttpException` / `TooManyBackupsException`
- Wings 调用失败：事务回滚，备份记录不保留
- 自动轮换失败：整个操作中止

#### 阶段 3：Wings 通信
**文件**：`app/Repositories/Wings/DaemonBackupRepository.php:35-53`

**职责**：
- 构造 HTTP 请求：`POST /api/servers/{uuid}/backup`
- 节点认证：使用 `node.daemon_token` 签名
- 传递参数：`adapter`、`uuid`、`ignore`（忽略文件列表）

**失败语义**：
- 网络异常：封装为 `DaemonConnectionException` 向上抛出
- Wings 业务错误：原样传递给上层

#### 阶段 4：Wings 执行（Panel 外部）
Wings 收到请求后：
1. 暂停服务器 I/O（可选）
2. 遍历文件系统，应用忽略规则
3. 打包为 tar.gz
4. 根据 `adapter` 决定存储目的地：
   - `wings`：写入 Wings 本地磁盘
   - `s3`：向 Panel 请求分片上传 URL，逐片上传

#### 阶段 5：S3 分片上传授权（仅 S3 场景）
**文件**：`app/Http\Controllers\Api\Remote\Backups\BackupRemoteUploadController.php:34-112`

**职责**：
- Wings 认证与归属校验
- 检查备份未完成（`completed_at == null`）
- 调用 S3 `CreateMultipartUpload` 获取 UploadId
- 按分片大小生成一批 `UploadPart` 预签名 URL
- 将 `upload_id` 保存到备份记录
- 返回分片 URL 列表给 Wings

**失败语义**：
- 备份已完成：`ConflictHttpException`
- 非 S3 适配器：`BadRequestHttpException`

#### 阶段 6：回调更新状态
**文件**：`app/Http/Controllers/Api/Remote/Backups/BackupStatusController.php:32-81`

**职责**：
- Wings 认证：`DaemonAuthenticate` 中间件校验节点身份
- 归属校验：确保请求节点 == 服务器所属节点
- 幂等性检查：已完成备份拒绝重复更新
- 状态更新：`is_successful`、`checksum`、`bytes`、`completed_at`
- S3 分片完成：调用 `completeMultipartUpload()` 合并或中止
- 失败备份自动解锁：便于后续清理

**失败语义**：
- 非所属节点请求：`HttpForbiddenException`
- 已完成备份重复回调：`BadRequestHttpException`
- S3 分片完成失败：事务回滚，状态不更新

---

## 二、备份还原流程分析

### 2.1 流程时序

```
用户 → Panel API → BackupController::restore() → DaemonBackupRepository → Wings
                                                                 ↓
Wings 回调 → BackupStatusController::restore() → Panel DB 更新
                                                                 ↓
                          （Wings 未回调时）Wings 重启 → /servers/reset → 状态强制重置
```

### 2.2 还原判定条件精确分析
**文件**：`app/Http/Controllers/Api/Client/Servers/BackupController.php:200-208`

```php
// 条件1：服务器状态必须为 null
if (!is_null($server->status)) {
    throw new BadRequestHttpException('...');
}

// 条件2：备份必须不是「未完成」的
if (!$backup->is_successful && is_null($backup->completed_at)) {
    throw new BadRequestHttpException('...');
}
```

**关键修正**：条件 2 是 `&&` 逻辑，意味着**失败但已完成的备份可以被还原**。

| `is_successful` | `completed_at` | 备份状态 | 是否可还原 |
|-----------------|----------------|----------|-----------|
| `false` | `null` | 进行中 / 完全失败未标记 | ❌ 拒绝 |
| `false` | 非 `null` | 明确失败（已标记完成时间） | ✅ 允许 |
| `true` | 非 `null` | 成功完成 | ✅ 允许 |
| `true` | `null` | （逻辑上不存在） | - |

> **设计意图推测**：允许还原失败备份是为了应对「备份创建部分成功但标记为失败」的边缘场景，给用户恢复数据的最后机会。但这也带来了还原不完整备份的风险。

### 2.3 各阶段职责切分

#### 阶段 1：客户端 API 入口
**文件**：`app/Http/Controllers/Api/Client/Servers/BackupController.php:198-229`

**职责**：
- 权限校验：`Permission::ACTION_BACKUP_RESTORE`
- 前置条件检查（见 2.2）
- S3 备份特殊处理：生成下载链接供 Wings 使用
- 服务器状态标记：设置为 `STATUS_RESTORING_BACKUP`
- 事务保证：状态更新 + Wings 调用在同一事务
- 审计事件：`server:backup.restore`

**失败语义**：
- 服务器状态不符：`BadRequestHttpException`
- 备份未完成（`is_successful=false && completed_at=null`）：`BadRequestHttpException`
- Wings 调用失败：事务回滚，服务器状态恢复

#### 阶段 2：Wings 通信
**文件**：`app/Repositories/Wings/DaemonBackupRepository.php:60-78`

**职责**：
- 构造 HTTP 请求：`POST /api/servers/{uuid}/backup/{backup_uuid}/restore`
- 传递参数：
  - `adapter`：备份存储类型
  - `truncate_directory`：是否清空目标目录
  - `download_url`：S3 备份的预签名下载 URL

#### 阶段 3：Wings 执行还原（Panel 外部）
Wings 收到请求后：
1. 停止服务器运行
2. 可选：清空服务器目录（`truncate=true`）
3. 从本地磁盘读取或从 S3 下载备份
4. 解压 tar.gz 到服务器目录
5. 恢复文件权限
6. 回调 Panel 报告结果（无论成败）

#### 阶段 4：回调更新状态（正常路径）
**文件**：`app/Http/Controllers/Api/Remote/Backups\BackupStatusController.php:93-111`

**职责**：
- Wings 认证与归属校验
- **无条件重置服务器状态为 `null`**（无论成功失败）
- 记录审计事件

**审计事件**：
| 结果 | 事件名 | 备注 |
|------|--------|------|
| 成功 | `server:backup.restore-complete` | 正常 |
| 失败 | `server.backup.restore-failed` | ⚠️ **BUG 1**：使用了 `.` 而非 `:`，与其他事件命名不一致 |

#### 阶段 5：强制重置状态（异常恢复路径）
**文件**：`app/Http/Controllers/Api/Remote/Servers/ServerDetailsController.php:89-134`

**触发时机**：Wings 重启时自动调用 `/api/remote/servers/reset`

**职责**：
1. 查找节点上所有 `status = restoring_backup` 的服务器
2. 尝试查找这些服务器最近的 `server:backup.restore-started` 审计事件
3. 如果找到且关联了备份主体，记录 `server:backup.restore-failed` 事件
4. 批量重置所有 `installing` 和 `restoring_backup` 状态为 `null`

> **关键设计决策**：还原失败不将服务器置于异常状态，用户可重试或使用重装功能。
> 参考 `BackupStatusController.php:84-89` 注释。

---

## 三、还原链路状态机与审计完整追踪

### 3.1 状态变更时序（完整闭环）

```
用户请求还原
    ↓
server.status = null → STATUS_RESTORING_BACKUP  ✅ 事务内
    ↓
Wings 收到请求，开始还原
    ↓
    ├─ 正常路径：Wings 回调（成功/失败）
    │       ↓
    │   server.status = STATUS_RESTORING_BACKUP → null  ✅ 无条件重置
    │
    └─ 异常路径：Wings 崩溃/网络分区，未回调
            ↓
        Wings 重启 → POST /api/remote/servers/reset
            ↓
        server.status = STATUS_RESTORING_BACKUP → null  ✅ 强制重置
```

### 3.2 状态阻塞影响
**文件**：`app/Models/Server.php:390-418`

`STATUS_RESTORING_BACKUP` 状态会阻塞以下操作：
- `validateCurrentState()`：阻塞控制台访问、电源操作、SFTP 等
- `validateTransferState()`：阻塞服务器迁移

### 3.3 审计事件完整清单

| 时机 | 事件名 | 触发位置 | 备注 |
|------|--------|----------|------|
| 用户发起还原 | `server:backup.restore` | `BackupController.php:210` | 事务内记录 |
| Wings 回调成功 | `server:backup.restore-complete` | `BackupStatusController.php:105` | 正常路径 |
| Wings 回调失败 | `server.backup.restore-failed` | `BackupStatusController.php:105` | ⚠️ BUG 1：点号分隔 |
| Wings 重启重置（找到备份） | `server:backup.restore-failed` | `ServerDetailsController.php:118` | 异常恢复路径 |

### 3.4 失败分支真实行为

**场景 1：Wings 调用失败（网络异常等）**
- 事务回滚
- 服务器状态保持 `null`
- 不产生回调事件
- 用户可立即重试

**场景 2：Wings 执行失败（文件损坏、磁盘满等）**
- Wings 回调 `successful=false`
- 服务器状态重置为 `null`
- 记录 `server.backup.restore-failed` 事件（⚠️ 命名 bug）
- 用户可立即重试或使用其他备份
- **风险**：服务器文件可能处于「部分还原」的不一致状态

**场景 3：Wings 未回调（进程崩溃、网络分区）**
- 服务器卡在 `STATUS_RESTORING_BACKUP`
- 所有需要 `validateCurrentState()` 的操作被阻塞
- **自动恢复**：Wings 重启时自动调用 `/servers/reset` 重置状态
- **手动恢复**：管理员可直接更新数据库 `servers.status = NULL`

---

## 四、还原失败恢复机制深度分析

### 4.1 /servers/reset 接口详解
**文件**：`app/Http/Controllers/Api/Remote/Servers/ServerDetailsController.php:89-134`

**触发条件**：
- Wings 服务启动时
- Wings 检测到状态不一致时

**执行步骤**：
```php
// 步骤1：查找节点上所有 restoring_backup 状态的服务器
$servers = Server::query()
    ->with([
        'activity' => fn ($builder) => $builder
            // ⚠️ BUG 2：查找的是 'restore-started'，但实际记录的是 'restore'
            ->where('activity_logs.event', 'server:backup.restore-started')
            ->latest('timestamp'),
    ])
    ->where('node_id', $node->id)
    ->where('status', Server::STATUS_RESTORING_BACKUP)
    ->get();

// 步骤2：为每个服务器补记失败审计
foreach ($servers as $server) {
    $activity = $server->activity->first();
    if (!is_null($activity)) {
        if ($subject = $activity->subjects->where('subject_type', 'backup')->first()) {
            Activity::event('server:backup.restore-failed')
                ->subject($server, $subject->subject)
                ->property('name', $subject->subject->name)
                ->log();
        }
    }
}

// 步骤3：批量重置状态
Server::query()
    ->where('node_id', $node->id)
    ->whereIn('status', [Server::STATUS_INSTALLING, Server::STATUS_RESTORING_BACKUP])
    ->update(['status' => null]);
```

### 4.2 两条回调路径的关系与边界

| 维度 | BackupStatusController::restore() | ServerDetailsController::resetState() |
|------|-----------------------------------|----------------------------------------|
| **触发时机** | Wings 完成还原操作后主动回调 | Wings 重启时的自检恢复 |
| **针对性** | 针对单个具体备份的还原结果 | 节点级批量重置所有异常状态 |
| **事件补全** | 基于还原结果直接记录 | 需通过审计日志反向追溯备份 |
| **状态重置** | 仅重置当前服务器 | 批量重置所有 installing/restoring_backup |
| **数据一致性** | 完整的成功/失败上下文 | 仅能确认「Wings 已重启，操作已终止」 |
| **边界** | 正常流程，精确审计 | 异常兜底，尽力审计 |

### 4.3 已知缺陷分析

**BUG 1：事件命名不一致**（`BackupStatusController.php:105`）
```php
// 成功：冒号分隔（符合规范）
'server:backup.restore-complete'
// 失败：点号分隔（不符合规范）
'server.backup.restore-failed'
```
**影响**：基于事件名的监控、告警、统计逻辑可能漏掉还原失败事件。

**BUG 2：reset 查找的事件名不匹配**（`ServerDetailsController.php:102`）
```php
// 代码中记录的事件
'server:backup.restore'
// reset 接口查找的事件
'server:backup.restore-started'
```
**影响**：Wings 重启重置时**永远找不到**对应的审计事件，导致：
- 无法补记 `server:backup.restore-failed` 事件
- 审计链路断裂，无法追溯哪个备份还原失败
- 状态仍能重置，但审计信息丢失

### 4.4 失败审计完整闭环（理想 vs 实际）

**理想闭环**：
```
server:backup.restore（开始）
    ↓
server:backup.restore-complete / server:backup.restore-failed（结束）
```

**实际闭环**：
```
server:backup.restore（开始）
    ├─ 成功 → server:backup.restore-complete（正常）
    ├─ 失败回调 → server.backup.restore-failed（⚠️ 命名错误）
    └─ 未回调 → /servers/reset → 状态重置，但无失败事件（⚠️ 事件名不匹配）
```

---

## 五、对象存储集成分析

### 5.1 BackupManager 适配器模式
**文件**：`app/Extensions/Backups/BackupManager.php`

**职责**：
- 统一的存储适配器工厂
- 支持两种内置适配器：
  - `wings`：`InMemoryFilesystemAdapter`（Panel 侧不实际操作文件）
  - `s3`：`S3Filesystem`（Panel 侧管理预签名 URL）

**权限边界**：
- Panel 持有 S3 密钥（`config/backups.php`）
- Wings 从不直接持有 S3 密钥，仅通过 Panel 生成的预签名 URL 访问

### 5.2 S3 备份创建数据流
```
1. Panel 创建备份记录，disk = 's3'，upload_id = null
2. Wings 打包完成，向 Panel 请求分片上传 URL
3. Panel 调用 S3 CreateMultipartUpload，获取 UploadId
4. Panel 生成 UploadPart 预签名 URL 列表返回给 Wings
5. Panel 将 upload_id 写入备份记录
6. Wings 使用预签名 URL 分片上传
7. Wings 回调 Panel，携带 ETag 列表
8. Panel 调用 S3 CompleteMultipartUpload 合并文件
```

### 5.3 S3 备份还原数据流
```
1. 用户请求还原
2. Panel 生成 S3 GetObject 预签名 URL（有效期 5 分钟）
3. Panel 将 URL 传递给 Wings
4. Wings 使用 URL 下载备份文件
5. Wings 解压还原
6. Wings 回调 Panel 报告结果
```

### 5.4 S3 备份删除数据流
**文件**：`app/Services/Backups/DeleteBackupService.php:68-82`

**执行顺序**：
1. 数据库事务内先删除备份记录
2. 再调用 S3 `DeleteObject` API
3. 即使 S3 删除失败，记录已删除（Panel 侧不再可见）

> **注意**：存在最终一致性风险，S3 删除失败可能产生孤立文件。

---

## 六、权限边界详解

### 6.1 用户权限模型
**文件**：`app/Models/Permission.php:142-151`

| 权限 | 描述 | 风险等级 |
|------|------|----------|
| `backup.read` | 查看备份列表 | 低 |
| `backup.create` | 创建备份 | 中 |
| `backup.delete` | 删除备份、锁定/解锁备份 | 中 |
| `backup.download` | 下载备份（可获取所有服务器文件） | 高 |
| `backup.restore` | 还原备份（可覆盖所有服务器文件） | 高 |

### 6.2 权限判定逻辑
**文件**：`app/Policies/ServerPolicy.php:26-33`

权限检查优先级：
1. **Root Admin**：无条件通过
2. **Server Owner**：无条件通过
3. **Subuser**：检查 `subuser.permissions` 数组

### 6.3 Wings 认证体系
**文件**：`app/Http/Middleware/Api/Daemon/DaemonAuthenticate.php`

**认证流程**：
1. 请求头：`Authorization: Bearer {token_id}.{token_secret}`
2. Panel 通过 `token_id` 查找 Node
3. 解密 `node.daemon_token` 并与 `token_secret` 常量比较
4. 认证通过后将 `node` 注入请求属性

**额外防护**（`BackupStatusController.php:45-49`）：
- 备份归属校验：确保回调节点 == 备份所属服务器的节点
- 防止恶意节点修改其他节点的备份状态

### 6.4 下载链接安全
**文件**：`app/Services/Backups/DownloadLinkService.php`

| 存储类型 | URL 类型 | 有效期 | 认证方式 |
|----------|----------|--------|----------|
| Wings 本地 | Panel 代理 URL | 15 分钟 | JWT 签名（含 user_id, backup_uuid, server_uuid） |
| S3 | 预签名 URL | 5 分钟 | AWS Signature v4 |

---

## 七、失败语义汇总

### 7.1 备份创建失败场景

| 失败点 | 处理方式 | 数据状态 |
|--------|----------|----------|
| 限流/超限 | 立即抛异常 | 无记录 |
| 自动轮换删除失败 | 中止操作 | 无记录 |
| Wings 连接失败 | 事务回滚 | 无记录 |
| Wings 打包失败 | Wings 回调标记失败 | 记录存在，`is_successful=false`，自动解锁 |
| S3 上传失败 | Wings 回调，Panel 中止分片 | 记录存在，`is_successful=false` |
| S3 合并分片失败 | 事务回滚 | 状态不更新 |

### 7.2 备份还原失败场景

| 失败点 | 处理方式 | 数据状态 | 风险 |
|--------|----------|----------|------|
| 服务器状态不符 | 立即拒绝 | 无变化 | 无 |
| 备份未完成（is_successful=false && completed_at=null） | 立即拒绝 | 无变化 | 无 |
| Wings 连接失败 | 事务回滚 | 服务器状态恢复 | 无 |
| Wings 执行失败（文件损坏等） | Wings 回调，状态重置为 null | 服务器状态正常，记录失败日志 | ⚠️ 文件可能部分还原 |
| Wings 未回调 | Wings 重启时自动调用 /servers/reset | 状态重置，但审计事件丢失 | ⚠️ BUG：事件名不匹配 |

### 7.3 备份删除失败场景

| 失败点 | 处理方式 | 数据状态 |
|--------|----------|----------|
| 备份已锁定 | `BackupLockedException` | 无变化 |
| Wings 返回 404 | 忽略，继续删除 Panel 记录 | 记录删除，Wings 侧可能残留 |
| Wings 其他错误 | 中止操作 | 记录保留 |
| S3 删除失败 | 记录已删除，S3 文件可能残留 | 记录删除 |

---

## 八、已知问题与风险

### 8.1 事件命名不一致
**文件**：`BackupStatusController.php:105`

```php
// 成功使用冒号分隔
'server:backup.restore-complete'
// 失败使用点号分隔（BUG）
'server.backup.restore-failed'
```

**影响**：基于事件名的监控、告警、统计逻辑可能漏掉还原失败事件。

### 8.2 reset 接口事件名不匹配
**文件**：`ServerDetailsController.php:102`

```php
// 实际记录的事件
'server:backup.restore'
// reset 接口查找的事件
'server:backup.restore-started'
```

**影响**：Wings 重启重置时无法补记失败审计事件，审计链路断裂。

### 8.3 失败备份可还原
**文件**：`BackupController.php:206`

**风险**：用户可能还原一个不完整、损坏的备份，导致服务器无法正常运行。

### 8.4 S3 删除最终一致性
**文件**：`DeleteBackupService.php:68-82`

**风险**：先删 DB 记录再删 S3 文件，若 S3 删除失败产生孤立文件，占用存储成本。

---

## 九、关键设计决策

### 9.1 事务边界设计
- **创建备份**：DB 记录 + Wings 调用在同一事务
  - 保证 Wings 收到请求时记录已存在
  - 失败则无残留记录
  
- **删除 S3 备份**：先删 DB 记录，再删 S3 文件
  - 优先保证 Panel 侧状态一致
  - 接受 S3 可能存在孤立文件的风险

### 9.2 锁定机制
- 成功备份可锁定，防止自动轮换或误删
- 失败备份自动解锁，便于清理
- 锁定操作需要 `backup.delete` 权限

### 9.3 状态机设计
- 服务器状态：`null` → `restoring_backup` → `null`
- 还原失败不设特殊失败状态，降低用户理解成本
- 还原期间禁止其他操作（通过状态检查实现）
- Wings 重启时提供兜底重置机制，防止永久卡住

### 9.4 失败备份可还原
- 设计意图：给用户恢复部分数据的机会
- 权衡：可用性 > 数据完整性保证
- 建议：UI 应明确提示「失败备份可能不完整」

### 9.5 双路径恢复设计
- **正常路径**：Wings 回调，精确审计
- **异常路径**：Wings 重启重置，尽力审计
- 设计原则：状态可恢复优先，审计尽力而为

---

## 十、核心文件索引

| 模块 | 文件路径 |
|------|----------|
| 备份创建服务 | `app/Services/Backups/InitiateBackupService.php` |
| 备份删除服务 | `app/Services/Backups/DeleteBackupService.php` |
| 下载链接服务 | `app/Services/Backups/DownloadLinkService.php` |
| Wings 通信 | `app/Repositories/Wings/DaemonBackupRepository.php` |
| 客户端 API | `app/Http/Controllers/Api/Client/Servers/BackupController.php` |
| 远程回调 API | `app/Http/Controllers/Api/Remote/Backups/BackupStatusController.php` |
| 状态重置 API | `app/Http/Controllers/Api/Remote/Servers/ServerDetailsController.php:89-134` |
| S3 分片上传授权 | `app/Http/Controllers/Api/Remote/Backups/BackupRemoteUploadController.php` |
| 存储适配器 | `app/Extensions/Backups/BackupManager.php` |
| 权限定义 | `app/Models/Permission.php` |
| 权限策略 | `app/Policies/ServerPolicy.php` |
| Wings 认证 | `app/Http/Middleware/Api/Daemon/DaemonAuthenticate.php` |
| 备份模型 | `app/Models/Backup.php` |
| 服务器状态校验 | `app/Models/Server.php:390-418` |
| 活动日志模型 | `app/Models/ActivityLog.php` |
| 配置文件 | `config/backups.php` |
