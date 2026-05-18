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
用户 → Panel API → InitiateBackupService → DaemonBackupRepository → Wings → S3（可选）
                                                          ↓
Wings 回调 → BackupStatusController → Panel DB 更新
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

#### 阶段 5：回调更新状态
**文件**：`app/Http/Controllers/Api/Remote/Backups/BackupStatusController.php:32-81`

**职责**：
- Wings 认证：`DaemonAuthenticate` 中间件校验节点身份
- 归属校验：确保请求节点 == 服务器所属节点
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
```

### 2.2 各阶段职责切分

#### 阶段 1：客户端 API 入口
**文件**：`app/Http/Controllers/Api/Client/Servers/BackupController.php:198-229`

**职责**：
- 权限校验：`Permission::ACTION_BACKUP_RESTORE`
- 前置条件检查：
  - 服务器状态必须为 `null`（正常运行状态）
  - 备份必须 `is_successful=true` 且 `completed_at != null`
- S3 备份特殊处理：生成下载链接供 Wings 使用
- 服务器状态标记：设置为 `STATUS_RESTORING_BACKUP`
- 事务保证：状态更新 + Wings 调用在同一事务

**失败语义**：
- 服务器状态不符：`BadRequestHttpException`
- 备份未完成/失败：`BadRequestHttpException`
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
6. 回调 Panel 报告结果

#### 阶段 4：回调更新状态
**文件**：`app/Http/Controllers/Api/Remote/Backups/BackupStatusController.php:93-111`

**职责**：
- Wings 认证与归属校验
- 重置服务器状态为 `null`（无论成功失败）
- 记录活动日志：`server:backup.restore-complete` 或 `server:backup.restore-failed`

**关键设计决策**：
> 还原失败不将服务器置于异常状态，用户可重试或使用重装功能。
> 参考 `BackupStatusController.php:86-89` 注释。

---

## 三、对象存储集成分析

### 3.1 BackupManager 适配器模式
**文件**：`app/Extensions/Backups/BackupManager.php`

**职责**：
- 统一的存储适配器工厂
- 支持两种内置适配器：
  - `wings`：`InMemoryFilesystemAdapter`（Panel 侧不实际操作文件）
  - `s3`：`S3Filesystem`（Panel 侧管理预签名 URL）

**权限边界**：
- Panel 持有 S3 密钥（`config/backups.php`）
- Wings 从不直接持有 S3 密钥，仅通过 Panel 生成的预签名 URL 访问

### 3.2 S3 备份创建数据流
```
1. Panel 创建备份记录，disk = 's3'
2. Wings 请求 Panel 获取分片上传 URL
3. Wings 分片上传到 S3（使用预签名 URL）
4. Wings 回调 Panel，携带分片 ETag 列表
5. Panel 调用 S3 CompleteMultipartUpload 合并文件
```

### 3.3 S3 备份还原数据流
```
1. 用户请求还原
2. Panel 生成 S3 GetObject 预签名 URL（有效期 5 分钟）
3. Panel 将 URL 传递给 Wings
4. Wings 使用 URL 下载备份文件
5. Wings 解压还原
```

### 3.4 S3 备份删除数据流
**文件**：`app/Services/Backups/DeleteBackupService.php:68-82`

**执行顺序**：
1. 数据库事务内先删除备份记录
2. 再调用 S3 `DeleteObject` API
3. 即使 S3 删除失败，记录已删除（Panel 侧不再可见）

> **注意**：存在最终一致性风险，S3 删除失败可能产生孤立文件。

---

## 四、权限边界详解

### 4.1 用户权限模型
**文件**：`app/Models/Permission.php:142-151`

| 权限 | 描述 | 风险等级 |
|------|------|----------|
| `backup.read` | 查看备份列表 | 低 |
| `backup.create` | 创建备份 | 中 |
| `backup.delete` | 删除备份、锁定/解锁备份 | 中 |
| `backup.download` | 下载备份（可获取所有服务器文件） | 高 |
| `backup.restore` | 还原备份（可覆盖所有服务器文件） | 高 |

### 4.2 权限判定逻辑
**文件**：`app/Policies/ServerPolicy.php:26-33`

权限检查优先级：
1. **Root Admin**：无条件通过
2. **Server Owner**：无条件通过
3. **Subuser**：检查 `subuser.permissions` 数组

### 4.3 Wings 认证体系
**文件**：`app/Http/Middleware/Api/Daemon/DaemonAuthenticate.php`

**认证流程**：
1. 请求头：`Authorization: Bearer {token_id}.{token_secret}`
2. Panel 通过 `token_id` 查找 Node
3. 解密 `node.daemon_token` 并与 `token_secret` 常量比较
4. 认证通过后将 `node` 注入请求属性

**额外防护**（`BackupStatusController.php:45-49`）：
- 备份归属校验：确保回调节点 == 备份所属服务器的节点
- 防止恶意节点修改其他节点的备份状态

### 4.4 下载链接安全
**文件**：`app/Services/Backups/DownloadLinkService.php`

| 存储类型 | URL 类型 | 有效期 | 认证方式 |
|----------|----------|--------|----------|
| Wings 本地 | Panel 代理 URL | 15 分钟 | JWT 签名（含 user_id, backup_uuid, server_uuid） |
| S3 | 预签名 URL | 5 分钟 | AWS Signature v4 |

---

## 五、失败语义汇总

### 5.1 备份创建失败场景

| 失败点 | 处理方式 | 数据状态 |
|--------|----------|----------|
| 限流/超限 | 立即抛异常 | 无记录 |
| 自动轮换删除失败 | 中止操作 | 无记录 |
| Wings 连接失败 | 事务回滚 | 无记录 |
| Wings 打包失败 | Wings 回调标记失败 | 记录存在，`is_successful=false`，自动解锁 |
| S3 上传失败 | Wings 回调，Panel 中止分片 | 记录存在，`is_successful=false` |
| S3 合并分片失败 | 事务回滚 | 状态不更新 |

### 5.2 备份还原失败场景

| 失败点 | 处理方式 | 数据状态 |
|--------|----------|----------|
| 服务器状态不符 | 立即拒绝 | 无变化 |
| 备份未完成/失败 | 立即拒绝 | 无变化 |
| Wings 连接失败 | 事务回滚 | 服务器状态恢复 |
| Wings 解压失败 | Wings 回调 | 服务器状态重置为 `null`，记录失败日志 |

### 5.3 备份删除失败场景

| 失败点 | 处理方式 | 数据状态 |
|--------|----------|----------|
| 备份已锁定 | `BackupLockedException` | 无变化 |
| Wings 返回 404 | 忽略，继续删除 Panel 记录 | 记录删除，Wings 侧可能残留 |
| Wings 其他错误 | 中止操作 | 记录保留 |
| S3 删除失败 | 记录已删除，S3 文件可能残留 | 记录删除 |

---

## 六、关键设计决策

### 6.1 事务边界设计
- **创建备份**：DB 记录 + Wings 调用在同一事务
  - 保证 Wings 收到请求时记录已存在
  - 失败则无残留记录
  
- **删除 S3 备份**：先删 DB 记录，再删 S3 文件
  - 优先保证 Panel 侧状态一致
  - 接受 S3 可能存在孤立文件的风险

### 6.2 锁定机制
- 成功备份可锁定，防止自动轮换或误删
- 失败备份自动解锁，便于清理
- 锁定操作需要 `backup.delete` 权限

### 6.3 状态机设计
- 服务器状态：`null` → `restoring_backup` → `null`
- 还原失败不设特殊失败状态，降低用户理解成本
- 还原期间禁止其他操作（通过状态检查实现）

---

## 七、核心文件索引

| 模块 | 文件路径 |
|------|----------|
| 备份创建服务 | `app/Services/Backups/InitiateBackupService.php` |
| 备份删除服务 | `app/Services/Backups/DeleteBackupService.php` |
| 下载链接服务 | `app/Services/Backups/DownloadLinkService.php` |
| Wings 通信 | `app/Repositories/Wings/DaemonBackupRepository.php` |
| 客户端 API | `app/Http/Controllers/Api/Client/Servers/BackupController.php` |
| 远程回调 API | `app/Http/Controllers/Api/Remote/Backups/BackupStatusController.php` |
| 存储适配器 | `app/Extensions/Backups/BackupManager.php` |
| 权限定义 | `app/Models/Permission.php` |
| 权限策略 | `app/Policies/ServerPolicy.php` |
| Wings 认证 | `app/Http/Middleware/Api/Daemon/DaemonAuthenticate.php` |
| 备份模型 | `app/Models/Backup.php` |
| 配置文件 | `config/backups.php` |
