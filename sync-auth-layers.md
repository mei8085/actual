# Actual Budget 同步服务器授权机制层级分析

## 概述

Actual Budget 自托管同步服务器的授权机制分为四个核心层级，它们协作形成完整的安全防护链，阻断越权请求：

1. **账户登录层** - 用户身份认证入口
2. **会话凭证层** - Token 管理与验证
3. **预算文件访问授权层** - 文件级权限控制
4. **API 接口鉴权层** - 同步接口的权限检查

```
┌─────────────────────────────────────────────────────────┐
│                 API 请求进入                              │
└───────────────────────────┬─────────────────────────────┘
                            │
┌───────────────────────────▼─────────────────────────────┐
│              会话凭证层 (Session Validation)             │
│  - 从 Header/Body 提取 Token                             │
│  - 验证 Token 存在性与过期时间                           │
│  - 从 DB 加载 session 信息到 res.locals                  │
└───────────────────────────┬─────────────────────────────┘
                            │
┌───────────────────────────▼─────────────────────────────┐
│              API 接口鉴权层 (Route Handlers)             │
│  - 验证文件存在 (verifyFileExists)                       │
│  - 检查文件访问权限 (requireFileAccess)                  │
│  - 验证同步参数合法性 (validateSyncedFile)               │
└───────────────────────────┬─────────────────────────────┘
                            │
┌───────────────────────────▼─────────────────────────────┐
│           预算文件访问授权层 (User Access Control)       │
│  - 文件所有者检查 (file.owner === userId)                │
│  - 管理员权限检查 (isAdmin(userId))                      │
│  - 共享权限检查 (user_access 表)                         │
└───────────────────────────┬─────────────────────────────┘
                            │
┌───────────────────────────▼─────────────────────────────┐
│                    执行业务逻辑                           │
└─────────────────────────────────────────────────────────┘
```

---

## 第一层：账户登录层

### 1.1 登录方式

系统支持三种登录方式，在 `app-account.js:74-129` 中统一分发：

| 登录方式 | 触发条件 | 处理模块 |
|---------|---------|---------|
| **密码登录** | `loginMethod === 'password'` 或默认 | `accounts/password.js` |
| **OpenID Connect** | `loginMethod === 'openid'` | `accounts/openid.ts` |
| **Header 认证** | `loginMethod === 'header'` (反向代理) | `util/validate-user.ts:validateAuthHeader` |

### 1.2 密码登录流程

**代码位置**: `accounts/password.js:35-111`

```javascript
// 关键步骤：
1. bcrypt.compareSync(password, passwordHash)  // 密码验证
2. 查询或创建用户记录 (users 表)
3. 生成 UUID Token
4. 写入 sessions 表，设置过期时间
5. 返回 Token 给客户端
```

**安全特性**:
- 使用 bcrypt 哈希存储密码 (salt rounds: 12)
- 登录限流：15分钟内最多5次失败尝试 (`app-account.js:26-33`)
- Token 过期可配置：支持永不过期、自定义分钟数、跟随 OpenID Provider

### 1.3 OpenID Connect 登录流程

**代码位置**: `accounts/openid.ts`

分为两个阶段：

**阶段一：发起认证 (`loginWithOpenIdSetup`)**
- 生成 state、code_verifier、code_challenge
- 存储到 `pending_openid_requests` 表（5分钟过期）
- 重定向到 OIDC Provider 授权页面

**阶段二：回调处理 (`loginWithOpenIdFinalize`)**
- 验证 state 有效性（防止 CSRF）
- 用 code 换取 access_token
- 获取用户信息，使用 `preferred_username`/`login`/`email`/`id`/`sub` 作为身份标识
- 自动创建用户（如果是首个用户则设为 ADMIN）
- 生成会话 Token 并返回

**安全特性**:
- PKCE (Proof Key for Code Exchange) 支持
- 回调 URL 白名单验证 (`isValidRedirectUrl`)
- 首次登录需验证现有密码（防止绕过）

---

## 第二层：会话凭证层

### 2.1 会话验证中间件

**代码位置**: `util/middlewares.ts:33-45`

```typescript
const validateSessionMiddleware = async (req, res, next) => {
  const session = await validateSession(req, res);
  if (!session) return;  // 已在 validateSession 中发送 401
  res.locals = session;   // 会话信息注入响应上下文
  next();
};
```

### 2.2 Token 验证逻辑

**代码位置**: `util/validate-user.ts:10-42`

```typescript
export function validateSession(req: Request, res: Response) {
  // Token 提取优先级：body.token > header['x-actual-token']
  let token = req.body?.token || req.headers['x-actual-token'];
  
  // 1. 检查 Token 存在性
  const session = getSession(token);
  if (!session) {
    res.status(401).send({ status: 'error', reason: 'unauthorized', details: 'token-not-found' });
    return null;
  }

  // 2. 检查 Token 过期时间
  if (session.expires_at !== TOKEN_EXPIRATION_NEVER && 
      session.expires_at * 1000 <= Date.now()) {
    res.status(401).send({ status: 'error', reason: 'token-expired' });
    return null;
  }

  return session;
}
```

### 2.3 会话数据结构

`sessions` 表字段：
| 字段 | 类型 | 说明 |
|-----|------|------|
| `token` | VARCHAR | UUID v4 格式的会话令牌 |
| `expires_at` | INTEGER | Unix 时间戳，-1 表示永不过期 |
| `user_id` | VARCHAR | 关联用户 ID |
| `auth_method` | VARCHAR | 认证方式：`password`/`openid`/`header` |

### 2.4 会话清理

**代码位置**: `account-db.js:259-268`

- 每次登录时自动清理过期超过1小时的会话
- `DELETE FROM sessions WHERE expires_at <> -1 and expires_at < ?`

---

## 第三层：预算文件访问授权层

### 3.1 核心权限检查函数

**代码位置**: `app-sync.ts:119-129`

```typescript
function requireFileAccess(file: File, userId: string) {
  const isOwner = file.owner === userId;           // 检查文件所有者
  const isServerAdmin = isAdmin(userId);           // 检查服务器管理员
  if (isOwner || isServerAdmin) {
    return null;  // 有权限
  }
  if (UserService.countUserAccess(file.id, userId) > 0) {
    return null;  // 被共享的用户
  }
  return 'file-access-not-allowed';  // 无权限
}
```

### 3.2 权限判定优先级

权限检查按以下顺序进行，任一满足即通过：

1. **文件所有者**：`file.owner === userId`
   - 文件上传时自动设置所有者 (`app-sync.ts:374-378`)

2. **服务器管理员**：`isAdmin(userId)`
   - 用户表 `role = 'ADMIN'` 的用户
   - 可以访问所有文件

3. **共享权限**：`user_access` 表中有记录
   - 通过 `/admin/access` 接口由文件所有者或管理员授予

### 3.3 文件列表权限过滤

**代码位置**: `app-sync/services/files-service.ts:167-189`

`FilesService.find()` 方法在查询时就进行了权限过滤，防止越权看到其他用户文件：

```typescript
find({ userId, limit = 1000 }) {
  const canSeeAll = isAdmin(userId);
  
  return canSeeAll
    ? // 管理员：查看所有未删除文件
      this.accountDb.all('SELECT * FROM files WHERE deleted = 0 LIMIT ?', [limit])
    : // 普通用户：自己的文件 + 被共享的文件
      this.accountDb.all(`
        SELECT files.* FROM files WHERE files.owner = ? and deleted = 0
        UNION
        SELECT files.* FROM files
        JOIN user_access ON user_access.file_id = files.id AND user_access.user_id = ?
        WHERE files.deleted = 0 LIMIT ?`, 
        [userId, userId, limit]
      );
}
```

### 3.4 共享权限管理

**代码位置**: `app-admin.js:218-318`

| 接口 | 功能 | 权限要求 |
|-----|------|---------|
| `GET /admin/access?fileId=` | 查看文件的共享用户 | 文件所有者或管理员 |
| `POST /admin/access` | 授予用户文件访问权 | 文件所有者或管理员 |
| `DELETE /admin/access?fileId=` | 撤销用户文件访问权 | 文件所有者或管理员 |
| `POST /admin/access/transfer-ownership` | 转移文件所有权 | 文件所有者或管理员 |

---

## 第四层：API 接口鉴权层

### 4.1 同步接口统一防护

**代码位置**: `app-sync.ts:44-47`

所有 `/sync/*` 接口都经过 `validateSessionMiddleware`，确保：
- 请求必须携带有效的 Token
- 会话信息自动注入 `res.locals`

### 4.2 典型同步接口鉴权流程

以 `/sync/sync` 接口为例 (`app-sync.ts:131-194`)：

```typescript
app.post('/sync', async (req, res) => {
  // 1. 解析 protobuf 格式的同步请求
  const requestPb = fromBinary(SyncRequestSchema, req.body);
  
  // 2. 参数校验：since 字段必填
  if (!since) { res.status(422).send(...); return; }
  
  // 3. 验证文件存在
  const currentFile = verifyFileExists(fileId, filesService, res, 'file-not-found');
  if (!currentFile) return;
  
  // 4. 检查文件访问权限 ⭐ 越权阻断点
  const fileAccessError = requireFileAccess(currentFile, res.locals.user_id);
  if (fileAccessError) {
    res.status(403).send(fileAccessError);
    return;
  }
  
  // 5. 验证同步参数 (groupId, keyId 一致性)
  const errorMessage = validateSyncedFile(groupId, keyId, currentFile);
  if (errorMessage) { res.status(400).send(errorMessage); return; }
  
  // 6. 执行业务逻辑
  const { trie, newMessages } = simpleSync.sync(messages, since, groupId);
  // ... 返回响应
});
```

### 4.3 各同步接口鉴权点

| 接口 | 文件存在检查 | 权限检查 | 参数验证 |
|-----|------------|---------|---------|
| `POST /sync/sync` | ✅ | ✅ | ✅ |
| `POST /sync/user-get-key` | ✅ | ✅ | - |
| `POST /sync/user-create-key` | ✅ | ✅ | - |
| `POST /sync/reset-user-file` | ✅ | ✅ | - |
| `POST /sync/upload-user-file` | ⚠️ 新建时跳过 | ⚠️ 新建时跳过 | ✅ |
| `GET /sync/download-user-file` | ✅ | ✅ | ✅ (路径遍历防护) |
| `POST /sync/update-user-filename` | ✅ | ✅ | - |
| `GET /sync/list-user-files` | - (查询时过滤) | - (查询时过滤) | - |
| `GET /sync/get-user-file-info` | ✅ | ✅ | - |
| `POST /sync/delete-user-file` | ✅ | ✅ | - |

### 4.4 特殊安全防护

#### 路径遍历防护
**代码位置**: `app-sync.ts:440-444`

```typescript
const path = getPathForUserFile(fileId);
if (!path.startsWith(resolve(config.get('userFiles')))) {
  res.status(403).send('Access denied');
  return;
}
```

防止通过构造恶意 `fileId` 访问服务器其他目录。

#### 文件 ID 格式验证
**代码位置**: `util/paths.ts` (isValidFileId, isValidGroupId)

确保 fileId 和 groupId 符合 UUID v4 格式，防止 SQL 注入和路径遍历。

---

## 越权攻击阻断场景分析

### 场景 1：尝试访问他人预算文件

**攻击路径**：
1. 攻击者登录获取自己的 Token
2. 调用 `GET /sync/download-user-file`，Header 中传入他人的 `x-actual-file-id`

**阻断点**：
- `requireFileAccess()` 检查发现 `file.owner !== userId`
- 检查 `isAdmin(userId)` → 不是管理员
- 检查 `user_access` 表 → 没有共享记录
- **返回 403: file-access-not-allowed**

### 场景 2：尝试列出所有用户文件

**攻击路径**：
1. 攻击者登录获取 Token
2. 调用 `GET /sync/list-user-files`

**阻断点**：
- `FilesService.find()` 在 SQL 层面就进行了权限过滤
- UNION 查询只返回 `owner = ?` 或 `user_access.user_id = ?` 的文件
- **攻击者只能看到自己和被共享的文件**

### 场景 3：使用过期 Token

**攻击路径**：
1. 攻击者获取一个过期的 Token
2. 使用该 Token 调用任意同步接口

**阻断点**：
- `validateSession()` 检查 `session.expires_at * 1000 <= Date.now()`
- **返回 401: token-expired**

### 场景 4：使用伪造 Token

**攻击路径**：
1. 攻击者构造一个随机 UUID 作为 Token
2. 调用同步接口

**阻断点**：
- `getSession(token)` 查询 `sessions` 表无记录
- **返回 401: token-not-found**

### 场景 5：通过 Header 认证绕过

**攻击路径**：
1. 攻击者直接访问服务，在 Header 中设置 `x-actual-password`

**阻断点**：
- `validateAuthHeader()` 检查请求来源 IP
- 只有配置的 `trustedAuthProxies` 列表中的 IP 才能使用 Header 认证
- **非信任代理返回 proxy-not-trusted**

---

## 数据库表结构关系

```
┌─────────────┐       ┌──────────────┐       ┌─────────────┐
│   users     │       │   sessions   │       │    files    │
├─────────────┤       ├──────────────┤       ├─────────────┤
│ id (PK)     │◄──────│ user_id (FK) │       │ id (PK)     │
│ user_name   │       │ token        │       │ group_id    │
│ display_name│       │ expires_at   │       │ name        │
│ enabled     │       │ auth_method  │       │ encrypt_*   │
│ owner       │       └──────────────┘       │ deleted     │
│ role        │                              │ owner (FK)  │
└─────────────┘                              └──────┬──────┘
         ▲                                          │
         │                                          │
         │                                          ▼
         │                                ┌──────────────┐
         │                                │ user_access  │
         │                                ├──────────────┤
         └────────────────────────────────│ user_id (FK) │
                                          │ file_id (FK) │
                                          └──────────────┘
```

---

## 关键安全模块索引

| 功能 | 文件位置 | 关键函数/类 |
|-----|---------|------------|
| 会话验证 | `util/validate-user.ts` | `validateSession`, `validateAuthHeader` |
| 会话中间件 | `util/middlewares.ts` | `validateSessionMiddleware` |
| 密码登录 | `accounts/password.js` | `loginWithPassword`, `bootstrapPassword` |
| OIDC 登录 | `accounts/openid.ts` | `loginWithOpenIdSetup`, `loginWithOpenIdFinalize` |
| 文件权限检查 | `app-sync.ts` | `requireFileAccess`, `verifyFileExists` |
| 文件服务 | `app-sync/services/files-service.ts` | `FilesService`, `File`, `FileUpdate` |
| 用户权限服务 | `services/user-service.ts` | `countUserAccess`, `addUserAccess`, `checkFilePermission` |
| 账户数据库操作 | `account-db.js` | `getSession`, `isAdmin`, `hasPermission` |

---

## 总结

Actual Budget 的授权机制采用**纵深防御**策略：

1. **入口把关**：所有同步接口必须先通过会话验证
2. **边界检查**：每个接口都显式验证文件存在性和访问权限
3. **数据隔离**：文件列表查询在 SQL 层面就进行权限过滤
4. **最小权限**：普通用户默认只能访问自己的文件
5. **灵活共享**：通过 `user_access` 表实现受控的文件共享

这种设计确保了即使某一层被绕过，下一层仍然能够提供防护，有效阻断越权访问。
