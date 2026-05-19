# Actual Budget 同步服务器授权机制层级分析

> **修订说明**：本文档基于代码深度审计完成，所有结论均标注证据来源。明确区分「代码事实」与「风险推断」。本次修订重点核实了：sessions.auth_method 写入路径、user_name 为空账户的边界、多用户 session 接管声明的有效性。

---

## 证据链说明

本文档中所有结论分为两类：

| 类别 | 标记 | 含义 |
|-----|------|------|
| ✅ **代码事实** | 有明确代码证据支撑的客观事实 |
| ⚠️ **风险推断** | 基于代码逻辑推导的潜在问题/设计意图 |
| ❌ **已修正结论** | 之前版本中不准确的结论 |

---

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

### 1.1 登录方式决策链

✅ **代码事实**：`account-db.js:56-79`

登录方式的选择遵循严格的优先级顺序：

```javascript
export function getLoginMethod(req) {
  // 优先级 1: 请求体中明确指定的 loginMethod
  // 条件：在 allowedLoginMethods 白名单中 且 数据库中有该方法记录
  if (typeof req !== 'undefined' &&
      (req.body || { loginMethod: null }).loginMethod &&
      config.get('allowedLoginMethods').includes(req.body.loginMethod)) {
    const accountDb = getAccountDb();
    const row = accountDb.first('SELECT method FROM auth WHERE method = ?', 
                                [req.body.loginMethod]);
    if (row) return req.body.loginMethod;
  }

  // 优先级 2: Header 认证（如果配置则强制使用，绕过其他所有配置）
  // BY-PASS ANY OTHER CONFIGURATION TO ENSURE HEADER AUTH
  if (config.get('loginMethod') === 'header' &&
      config.get('allowedLoginMethods').includes('header')) {
    return config.get('loginMethod');
  }

  // 优先级 3: 数据库中标记为 active 的登录方式
  const activeMethod = getActiveLoginMethod();
  
  // 优先级 4: 配置文件中的 loginMethod
  // 优先级 5: 回退到 password（activeMethod 为 undefined 时）
  return activeMethod || config.get('loginMethod');
}
```

**决策链流程图**：
```
请求到达
    │
    ▼
┌────────────────────────────────────────────┐
│ 请求体有 loginMethod？                      │
│  且在 allowedLoginMethods 中？              │
│  且数据库 auth 表中有该方法记录？           │
└─────────────────┬──────────────────────────┘
                  │是
                  ▼
            使用该方法 ←─── 优先级最高
                  │否
                  ▼
┌────────────────────────────────────────────┐
│ 配置 loginMethod === 'header'              │
│ 且在 allowedLoginMethods 中？               │
└─────────────────┬──────────────────────────┘
                  │是
                  ▼
            使用 header 认证 ←─── 强制绕过
                  │否
                  ▼
┌────────────────────────────────────────────┐
│ 数据库有 active=1 的 auth 记录？           │
└─────────────────┬──────────────────────────┘
                  │是
                  ▼
            使用 active 方法
                  │否
                  ▼
┌────────────────────────────────────────────┐
│ 使用 config.loginMethod                    │
│ （默认: password）                          │
└────────────────────────────────────────────┘
```

### 1.2 三种登录方式对比

| 登录方式 | 触发条件 | 处理模块 | auth_method 写入值 |
|---------|---------|---------|-------------------|
| **密码登录** | 默认方式，或 loginMethod='password' | `accounts/password.js` | `'password'` |
| **OpenID Connect** | loginMethod='openid' | `accounts/openid.ts` | `'openid'` |
| **Header 认证** | 配置 loginMethod='header' | `util/validate-user.ts` + `loginWithPassword()` | `'password'`（复用密码登录） |

✅ **代码事实**：`app-account.js:89` - Header 认证通过后直接调用 `loginWithPassword(headerVal)`，因此 `auth_method` 被写入为 `'password'`。

### 1.3 sessions.auth_method 写入路径汇总

**生产代码中所有写入 `sessions.auth_method` 的位置**：

| 代码位置 | 写入值 | 场景 |
|---------|--------|------|
| `accounts/password.js:98` | `'password'` | 密码登录新建 session |
| `accounts/openid.ts:330` | `'openid'` | OpenID 登录新建 session |
| `migrations/1719409568000-multiuser.js:48` | `'password'` | 数据库迁移，更新旧数据（auth_method IS NULL） |

✅ **代码事实**：`password.js:103` 更新 session 时的 SQL 为：
```sql
UPDATE sessions SET user_id = ?, expires_at = ? WHERE token = ?
```
**不更新 `auth_method` 字段**，只更新 `user_id` 和 `expires_at`。

---

### 1.4 密码登录流程的 Session 复用机制

✅ **代码事实**：`accounts/password.js:35-111`

```javascript
export function loginWithPassword(password) {
  // ... 密码验证（bcrypt.compareSync）...
  
  // ⚠️ 关键：查找已存在的 password 类型 session
  const sessionRow = accountDb.first(
    'SELECT * FROM sessions WHERE auth_method = ?',
    ['password'],
  );

  // 如果已存在，复用同一个 token！
  const token = sessionRow ? sessionRow.token : uuidv4();
  
  // ⚠️ 关键：用户查找逻辑
  const { totalOfUsers } = accountDb.first(
    'SELECT count(*) as totalOfUsers FROM users',
  );
  let userId = null;
  if (totalOfUsers === 0) {
    // 首次登录：创建 user_name 为空的用户
    userId = uuidv4();
    accountDb.mutate(
      'INSERT INTO users (id, user_name, display_name, enabled, owner, role) VALUES (?, ?, ?, 1, 1, ?)',
      [userId, '', '', 'ADMIN'],
    );
  } else {
    // ⚠️ 非首次登录：总是查询 user_name = '' 的用户！
    const { id: userIdFromDb } = accountDb.first(
      'SELECT id FROM users WHERE user_name = ?',
      [''],
    );
    userId = userIdFromDb;
    if (!userId) {
      return { error: 'user-not-found' };
    }
  }
  // ... 计算过期时间 ...
  
  // 新建或更新 session
  if (!sessionRow) {
    accountDb.mutate(
      'INSERT INTO sessions (token, expires_at, user_id, auth_method) VALUES (?, ?, ?, ?)',
      [token, expiration, userId, 'password'],
    );
  } else {
    accountDb.mutate(
      'UPDATE sessions SET user_id = ?, expires_at = ? WHERE token = ?',
      [userId, expiration, token],
    );
  }
  
  return { token };
}
```

#### ❌ 已修正结论：密码登录的多用户 session 接管

**之前的结论**：在多用户场景下，用户 B 用密码登录后，用户 A 的会话会被"接管"。

**修正后的准确结论**：
- ✅ **代码事实**：密码登录时总是查询 `user_name = ''` 的用户（`password.js:74-77`）
- ✅ **代码事实**：管理员通过 `/admin/users` 创建的用户 `userName` 不能为空（`app-admin.js:52`）
- ✅ **代码事实**：`getUserByUsername('')` 返回 null（`user-service.ts:5-6`：`if (!userName) return null`）
- **结论**：密码登录模式下系统设计为**单用户模式**，只有 `user_name = ''` 的用户可以通过密码登录。管理员创建的其他用户无法通过密码登录，因此不存在"多用户 session 接管"的场景。

#### Password Session 复用的边界条件

| 条件 | 结果 |
|-----|------|
| 首次登录（users 表为空） | 创建 user_name='' 的 ADMIN 用户，新建 session |
| 非首次登录，存在 user_name='' 的用户 | 复用该用户的 session，更新 expires_at |
| 非首次登录，不存在 user_name='' 的用户 | 返回 'user-not-found' 错误 |
| 管理员创建的 user_name≠'' 的用户 | 无法通过密码登录 |

---

### 1.5 Token 过期配置语义与单位口径

#### 配置定义
✅ **代码事实**：`load-config.js:258-263`
```javascript
token_expiration: {
  doc: 'Token expiration time.',
  format: 'tokenExpiration',
  default: 'never',
  env: 'ACTUAL_TOKEN_EXPIRATION',
},
```

✅ **代码事实**：`load-config.js:31-59` - `tokenExpiration` 格式定义：
- 允许值：`'never'`、`'openid-provider'`、非负数字

✅ **文档证据**：`packages/docs/docs/config/oauth-auth.md:174-180`
> **Possible Values:**
> - `"never"` (tokens never expire - current default)
> - `"openid-provider"` (tokens follow the expiration time from the OpenID provider)
> - A numeric value in seconds (e.g., `3600` for 1 hour)

**结论**：`token_expiration` 数值配置的单位是**秒**。

#### 密码登录分支实现
✅ **代码事实**：`accounts/password.js:86-94`
```javascript
let expiration = TOKEN_EXPIRATION_NEVER;  // -1
if (config.get('token_expiration') !== 'never' &&
    config.get('token_expiration') !== 'openid-provider' &&
    typeof config.get('token_expiration') === 'number') {
  // ⚠️ BUG: 文档规定单位是秒，但代码乘以了 60！
  expiration = Math.floor(Date.now() / 1000) + config.get('token_expiration') * 60;
}
```

#### OpenID 登录分支实现
✅ **代码事实**：`accounts/openid.ts:317-327`
```javascript
let expiration;
if (config.get('token_expiration') === 'openid-provider') {
  expiration = tokenSet.expires_at ?? TOKEN_EXPIRATION_NEVER;
} else if (config.get('token_expiration') === 'never') {
  expiration = TOKEN_EXPIRATION_NEVER;
} else if (typeof config.get('token_expiration') === 'number') {
  // ✅ 正确：直接加秒数，符合文档
  expiration = Math.floor(Date.now() / 1000) + config.get('token_expiration');
} else {
  expiration = Math.floor(Date.now() / 1000) + 10 * 60; // 默认 10 分钟
}
```

**不一致性总结**：

| 登录方式 | 代码实现 | 与文档一致性 | 实际效果（配置 3600 秒） |
|---------|---------|-------------|------------------------|
| 密码登录 | `config * 60` | ❌ 不一致 | Token 在 60 小时后过期（216000 秒） |
| OpenID 登录 | `config`（无乘法） | ✅ 一致 | Token 在 1 小时后过期（3600 秒） |

⚠️ **风险推断**：这是一个 Bug。密码登录实现错误地将秒当成了分钟处理。

---

### 1.6 Header 认证

✅ **代码事实**：`util/validate-user.ts:44-68` + `app-account.js:79-95`

Header 认证是为反向代理场景设计的：
- 信任的代理在 `x-actual-password` Header 中传递密码
- `validateAuthHeader()` 验证请求来源 IP 是否在 `trustedAuthProxies` 列表中
- 认证通过后调用 `loginWithPassword()`，复用密码登录的 session 机制

✅ **代码事实**：`validateAuthHeader()` 检查的是 `req.socket.remoteAddress`（直接连接的对端 IP），不是 `X-Forwarded-For`。

⚠️ **风险推断**：如果代理层没有正确配置，可能导致信任链断裂。

---

## 第二层：会话凭证层

### 2.1 会话验证中间件

✅ **代码事实**：`util/middlewares.ts:33-45`

```typescript
const validateSessionMiddleware = async (req, res, next) => {
  const session = await validateSession(req, res);
  if (!session) return;  // 验证失败时已在 validateSession 中发送 401
  res.locals = session;   // 会话信息注入响应上下文
  next();
};
```

**应用范围**：
- ✅ `/sync/*` 所有接口：`app-sync.ts:45` 强制应用
- ⚠️ `/admin/*` 部分接口：选择性应用（如 `POST /admin/access` 直接调用 `validateSession`，不使用中间件）

### 2.2 Token 验证逻辑

✅ **代码事实**：`util/validate-user.ts:10-42`

```typescript
export function validateSession(req: Request, res: Response) {
  // Token 提取优先级：body.token > header['x-actual-token']
  let token = req.body?.token || req.headers['x-actual-token'];
  
  // 1. 检查 Token 存在性
  const session = getSession(token);
  if (!session) {
    res.status(401).send({ 
      status: 'error', reason: 'unauthorized', details: 'token-not-found' 
    });
    return null;
  }

  // 2. 检查 Token 过期时间
  // expires_at 单位是秒，-1 表示永不过期
  if (session.expires_at !== TOKEN_EXPIRATION_NEVER && 
      session.expires_at * MS_PER_SECOND <= Date.now()) {
    res.status(401).send({ status: 'error', reason: 'token-expired' });
    return null;
  }

  return session;
}
```

### 2.3 会话数据结构

✅ **代码事实**：`migrations/1719409568000-multiuser.js:19-37`

`sessions` 表字段：

| 字段 | 类型 | 说明 |
|-----|------|------|
| `token` | TEXT (PK) | UUID v4 格式的会话令牌 |
| `expires_at` | INTEGER | Unix 时间戳（秒），-1 表示永不过期 |
| `user_id` | TEXT | 关联用户 ID |
| `auth_method` | TEXT | 认证方式：`password`/`openid` |

### 2.4 会话清理

✅ **代码事实**：`account-db.js:259-268`

```javascript
export function clearExpiredSessions() {
  const clearThreshold = Math.floor(Date.now() / 1000) - 3600; // 1小时前
  const deletedSessions = getAccountDb().mutate(
    'DELETE FROM sessions WHERE expires_at <> -1 and expires_at < ?',
    [clearThreshold],
  ).changes;
}
```

- ✅ **代码事实**：每次登录时自动调用
- ✅ **代码事实**：清理过期时间超过 1 小时的会话（给用户留了宽限期）

---

## 第三层：预算文件访问授权层

### 3.1 两个权限检查函数的关键区别

✅ **代码事实**：系统中存在两个不同的权限检查函数，应用场景完全不同！

| 函数 | 代码位置 | 权限判定逻辑 | 应用场景 |
|-----|---------|------------|---------|
| `requireFileAccess()` | `app-sync.ts:119-129` | owner **OR** admin **OR** user_access | 同步接口（/sync/*） |
| `checkFilePermission()` | `user-service.ts:184-193` | **仅** owner | 管理接口（/admin/access） |

### 3.2 同步接口权限检查 (`requireFileAccess`)

✅ **代码事实**：`app-sync.ts:119-129`

```typescript
function requireFileAccess(file: File, userId: string) {
  const isOwner = file.owner === userId;           // 检查文件所有者
  const isServerAdmin = isAdmin(userId);           // 检查服务器管理员
  if (isOwner || isServerAdmin) {
    return null;  // 有权限
  }
  // 检查 user_access 共享表
  if (UserService.countUserAccess(file.id, userId) > 0) {
    return null;  // 被共享的用户
  }
  return 'file-access-not-allowed';  // 无权限
}
```

**权限判定优先级**：
1. ✅ 文件所有者：`file.owner === userId`
2. ✅ 服务器管理员：`isAdmin(userId)`（role = 'ADMIN'）
3. ✅ 共享用户：`user_access` 表中有记录
4. ❌ 无权限

### 3.3 管理接口权限检查 (`checkFilePermission`)

✅ **代码事实**：`user-service.ts:184-193`

```typescript
export function checkFilePermission(fileId, userId) {
  // ⚠️ 只检查是否是文件所有者，不包含 user_access！
  return (
    getAccountDb().first(
      `SELECT 1 as granted
       FROM files
       WHERE files.id = ? and (files.owner = ?)`,
      [fileId, userId],
    ) || { granted: 0 }
  );
}
```

**关键影响**：
- ✅ **代码事实**：被共享的用户（通过 user_access）**不能**再将文件共享给其他人
- ✅ **代码事实**：被共享的用户**不能**转移文件所有权
- ✅ **代码事实**：只有文件所有者和管理员可以管理共享权限

### 3.4 `countUserAccess` 的完整逻辑

✅ **代码事实**：`user-service.ts:169-182`

```typescript
export function countUserAccess(fileId, userId) {
  const { accessCount } =
    getAccountDb().first(
      `SELECT COUNT(*) as accessCount
       FROM files
       WHERE files.id = ? AND (
         files.owner = ? OR EXISTS (
           SELECT 1 FROM user_access
           WHERE user_access.user_id = ? AND user_access.file_id = ?
         )
       )`,
      [fileId, userId, userId, fileId],
    ) || {};
  return accessCount || 0;
}
```

✅ **代码事实**：这个函数在 `requireFileAccess` 中被调用时已经检查过 owner 和 admin，所以实际上只用于检查 user_access 表。

### 3.5 文件列表权限过滤

✅ **代码事实**：`app-sync/services/files-service.ts:167-189`

`FilesService.find()` 方法在 SQL 层面进行权限过滤：

```typescript
find({ userId, limit = 1000 }) {
  const canSeeAll = isAdmin(userId);
  
  return canSeeAll
    ? // 管理员：查看所有未删除文件
      this.accountDb.all('SELECT * FROM files WHERE deleted = 0 LIMIT ?', [limit])
    : // 普通用户：自己的文件 + 被共享的文件（UNION 去重）
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

---

## 第四层：API 接口鉴权层

### 4.1 同步接口统一防护

✅ **代码事实**：`app-sync.ts:44-47`

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
| `POST /sync/sync` | ✅ | ✅ `requireFileAccess` | ✅ |
| `POST /sync/user-get-key` | ✅ | ✅ `requireFileAccess` | - |
| `POST /sync/user-create-key` | ✅ | ✅ `requireFileAccess` | - |
| `POST /sync/reset-user-file` | ✅ | ✅ `requireFileAccess` | - |
| `POST /sync/upload-user-file` | ⚠️ 新建时跳过 | ⚠️ 新建时跳过 | ✅ |
| `GET /sync/download-user-file` | ✅ | ✅ `requireFileAccess` | ✅ (路径遍历防护) |
| `POST /sync/update-user-filename` | ✅ | ✅ `requireFileAccess` | - |
| `GET /sync/list-user-files` | - (查询时过滤) | - (查询时过滤) | - |
| `GET /sync/get-user-file-info` | ✅ | ✅ `requireFileAccess` | - |
| `POST /sync/delete-user-file` | ✅ | ✅ `requireFileAccess` | - |

### 4.4 特殊安全防护

#### 路径遍历防护
✅ **代码事实**：`app-sync.ts:440-444`

```typescript
const path = getPathForUserFile(fileId);
if (!path.startsWith(resolve(config.get('userFiles')))) {
  res.status(403).send('Access denied');
  return;
}
```

防止通过构造恶意 `fileId` 访问服务器其他目录。

#### 文件 ID 格式验证
确保 fileId 和 groupId 符合 UUID v4 格式，防止 SQL 注入和路径遍历。

---

## /admin/access 对授权的影响

### 5.1 授予访问权限 (`POST /admin/access`)

✅ **代码事实**：`app-admin.js:218-271`

```javascript
app.post('/access', (req, res) => {
  const session = validateSession(req, res);
  if (!session) return;

  // ⚠️ 使用 checkFilePermission：仅 owner 或 admin 可以授予权限
  // 被共享的用户不能再共享给其他人！
  const { granted } = UserService.checkFilePermission(
    userAccess.fileId, session.user_id,
  ) || { granted: 0 };

  if (granted === 0 && !isAdmin(session.user_id)) {
    res.status(400).send({ status: 'error', reason: 'file-denied' });
    return;
  }

  // 检查目标用户是否已有权限（通过 owner 或 user_access）
  if (UserService.countUserAccess(userAccess.fileId, userAccess.userId) > 0) {
    res.status(400).send({ status: 'error', reason: 'user-already-have-access' });
    return;
  }

  // 插入 user_access 记录
  UserService.addUserAccess(userAccess.userId, userAccess.fileId);
  res.status(200).send({ status: 'ok' });
});
```

**授权变化**：
- 目标用户获得该文件的访问权限
- 目标用户可以：同步、下载、查看文件信息
- 目标用户不能：共享给他人、转移所有权、删除文件

### 5.2 撤销访问权限 (`DELETE /admin/access`)

✅ **代码事实**：`app-admin.js:273-318`

- 同样需要 owner 或 admin 权限
- 从 `user_access` 表删除记录
- 被撤销的用户立即失去访问权限（下次请求时被 `requireFileAccess` 阻断）

### 5.3 转移所有权 (`POST /admin/access/transfer-ownership`)

✅ **代码事实**：`app-admin.js:353-408`

```javascript
app.post('/access/transfer-ownership/', validateSessionMiddleware, (req, res) => {
  // ⚠️ 同样使用 checkFilePermission：仅 owner 或 admin 可以转移
  const { granted } = UserService.checkFilePermission(
    newUserOwner.fileId, res.locals.user_id,
  ) || { granted: 0 };

  if (granted === 0 && !isAdmin(res.locals.user_id)) {
    res.status(400).send({ status: 'error', reason: 'file-denied' });
    return;
  }

  // 更新 files.owner 字段
  UserService.updateFileOwner(newUserOwner.newUserId, newUserOwner.fileId);
  res.status(200).send({ status: 'ok' });
});
```

**所有权转移后的权限变化**：

| 用户 | 转移前权限 | 转移后权限 |
|-----|-----------|-----------|
| 旧所有者 | Owner（完整权限） | 失去 Owner 权限<br>⚠️ 如果旧所有者在 user_access 表中仍有记录，则保留访问权限<br>否则完全失去访问 |
| 新所有者 | 可能无权限 / 可能有 user_access | 获得 Owner 权限<br>可以共享、转移、删除文件 |
| 其他共享用户 | user_access 权限 | 不变 |

⚠️ **风险推断**：转移所有权后，系统不会自动清理 user_access 表中旧所有者的记录。如果旧所有者之前在 user_access 表中（通常不会，但理论可能），则仍然保留访问权限。

---

## 越权攻击阻断路径分析

### 场景 1：尝试访问他人预算文件

**攻击路径**：
1. 攻击者登录获取自己的 Token
2. 调用 `GET /sync/download-user-file`，Header 中传入他人的 `x-actual-file-id`

**阻断路径**：
```
validateSessionMiddleware (验证 Token)
        ↓
verifyFileExists (文件存在检查) → 不存在返回 400
        ↓
requireFileAccess (权限检查)
        ├─→ 检查 file.owner === userId → ❌ 不相等
        ├─→ 检查 isAdmin(userId) → ❌ 不是管理员
        └─→ 检查 user_access 表 → ❌ 无记录
        ↓
返回 403: file-access-not-allowed
```

### 场景 2：尝试列出所有用户文件

**攻击路径**：
1. 攻击者登录获取 Token
2. 调用 `GET /sync/list-user-files`

**阻断路径**：
```
validateSessionMiddleware (验证 Token)
        ↓
FilesService.find() (SQL 层面过滤)
        ├─→ isAdmin(userId) → ❌ false
        └─→ UNION 查询：owner = ? OR user_access.user_id = ?
        ↓
只返回攻击者自己的文件和被共享的文件
```

### 场景 3：使用过期 Token

**攻击路径**：
1. 攻击者获取一个过期的 Token
2. 使用该 Token 调用任意同步接口

**阻断路径**：
```
validateSessionMiddleware
        ↓
validateSession
        ├─→ getSession(token) → ✅ 找到记录
        └─→ 检查 expires_at * 1000 <= Date.now() → ✅ 已过期
        ↓
返回 401: token-expired
```

### 场景 4：使用伪造 Token

**攻击路径**：
1. 攻击者构造一个随机 UUID 作为 Token
2. 调用同步接口

**阻断路径**：
```
validateSessionMiddleware
        ↓
validateSession
        └─→ getSession(token) → ❌ 无记录
        ↓
返回 401: token-not-found
```

### 场景 5：被共享用户尝试再共享

**攻击路径**：
1. 用户 A 共享文件给用户 B
2. 用户 B 尝试调用 `POST /admin/access` 将文件共享给用户 C

**阻断路径**：
```
validateSession (验证 B 的 Token)
        ↓
checkFilePermission(fileId, B's userId)
        └─→ 只检查 files.owner = B's userId → ❌ false
        ↓
检查 isAdmin(B's userId) → ❌ false
        ↓
返回 400: file-denied
```

### 场景 6：通过 Header 认证绕过

**攻击路径**：
1. 攻击者直接访问服务，在 Header 中设置 `x-actual-password`

**阻断路径**：
```
/login 接口
        ↓
getLoginMethod → 返回 'header'
        ↓
validateAuthHeader(req)
        └─→ 检查 remoteAddress 是否在 trustedAuthProxies → ❌ 不在
        ↓
返回 proxy-not-trusted
```

### 场景 7：路径遍历攻击

**攻击路径**：
1. 攻击者构造恶意 fileId，如 `../../../etc/passwd`
2. 调用 `GET /sync/download-user-file`

**阻断路径**：
```
validateSessionMiddleware
        ↓
isValidFileId(fileId) → ❌ 不符合 UUID v4 格式
        ↓
返回 400: invalid fileId

（如果 fileId 格式验证被绕过）
        ↓
getPathForUserFile(fileId) → 构造路径
        ↓
检查 path.startsWith(userFilesDir) → ❌ 不匹配
        ↓
返回 403: Access denied
```

### 场景 8：管理员创建的用户尝试密码登录

**攻击路径**：
1. 管理员创建了一个 user_name = 'alice' 的用户
2. Alice 尝试使用密码登录

**阻断路径**：
```
/login 接口
        ↓
loginWithPassword(password)
        ↓
查询 user_name = '' 的用户 → ❌ Alice 的 user_name 不是空字符串
        ↓
返回 user-not-found
```

---

## 数据库表结构关系

```
┌─────────────┐       ┌──────────────┐       ┌─────────────┐
│   users     │       │   sessions   │       │    files    │
├─────────────┤       ├──────────────┤       ├─────────────┤
│ id (PK)     │◄──────│ user_id      │       │ id (PK)     │
│ user_name   │       │ token (PK)   │       │ group_id    │
│ display_name│       │ expires_at   │       │ name        │
│ enabled     │       │ auth_method  │       │ encrypt_*   │
│ owner       │       └──────────────┘       │ deleted     │
│ role        │                              │ owner       │◄──┐
└─────────────┘                              └──────┬──────┘   │
         ▲                                          │          │
         │                                          │          │
         │                                          ▼          │
         │                                ┌──────────────┐     │
         │                                │ user_access  │     │
         │                                ├──────────────┤     │
         └────────────────────────────────│ user_id (FK) │     │
                                          │ file_id (FK) │─────┘
                                          └──────────────┘
```

**表结构说明**：
- ✅ `users.role`：'ADMIN' 或 'BASIC'
- ✅ `users.owner`：1 表示初始所有者（OpenID 模式下首个用户）
- ✅ `users.user_name`：密码登录模式下有一个特殊用户 `user_name = ''`
- ✅ `files.deleted`：软删除标记
- ✅ `user_access`：联合主键 (user_id, file_id)

---

## 不准确结论修正汇总

| 之前的结论 | 修正后的准确结论 | 证据 |
|----------|----------------|------|
| Token 过期时间单位统一 | ❌ 密码登录是 **秒 × 60**，OpenID 是 **秒**，存在不一致<br>文档规定单位是秒，密码登录实现错误 | `password.js:93` vs `openid.ts:323`<br>`docs/config/oauth-auth.md:180` |
| 每次密码登录创建新 session | ❌ 所有密码用户**共享同一个 session** | `password.js:56-61` |
| 多用户场景下 session 会被接管 | ❌ 密码登录设计为单用户模式，只有 `user_name=''` 的用户可以登录<br>管理员创建的其他用户无法通过密码登录，不存在接管场景 | `password.js:74-77`<br>`app-admin.js:52` |
| `/admin/access` 权限检查与同步接口一致 | ❌ 同步接口用 `requireFileAccess` (3 条件)<br>管理接口用 `checkFilePermission` (仅 owner) | `app-sync.ts:119` vs `user-service.ts:184` |
| 被共享用户可以再共享 | ❌ 被共享用户不能再共享或转移所有权 | `app-admin.js:224-238` |
| 转移所有权后旧所有者完全失去访问 | ❌ 如果旧所有者在 user_access 表中，仍保留访问权限 | `requireFileAccess` 逻辑 |
| Header 认证是独立的登录方式 | ❌ Header 认证只是密码登录的信任代理，复用密码 session | `app-account.js:89` |

---

## 关键安全模块索引

| 功能 | 文件位置 | 关键函数/类 |
|-----|---------|------------|
| 登录方式决策 | `account-db.js:56-79` | `getLoginMethod` |
| 会话验证 | `util/validate-user.ts` | `validateSession`, `validateAuthHeader` |
| 会话中间件 | `util/middlewares.ts` | `validateSessionMiddleware` |
| 密码登录 | `accounts/password.js` | `loginWithPassword`, `bootstrapPassword` |
| OIDC 登录 | `accounts/openid.ts` | `loginWithOpenIdSetup`, `loginWithOpenIdFinalize` |
| 同步接口权限 | `app-sync.ts:119-129` | `requireFileAccess` |
| 管理接口权限 | `user-service.ts:184-193` | `checkFilePermission` |
| 共享权限计数 | `user-service.ts:169-182` | `countUserAccess` |
| 文件服务 | `app-sync/services/files-service.ts` | `FilesService`, `File`, `FileUpdate` |
| 用户权限服务 | `services/user-service.ts` | `addUserAccess`, `updateFileOwner`, `getUserByUsername` |
| 账户数据库操作 | `account-db.js` | `getSession`, `isAdmin`, `hasPermission` |

---

## 总结

Actual Budget 的授权机制采用**纵深防御**策略，以下是经过代码验证的关键结论：

### ✅ 代码事实总结

1. **登录方式决策链**：5 级优先级，Header 认证可强制绕过其他配置
2. **Session 复用**：密码登录所有用户共享同一个 session，但密码登录设计为单用户模式（只有 `user_name=''` 的用户可以登录）
3. **Token 过期 Bug**：文档规定单位是秒，但密码登录实现中错误地乘以 60
4. **两套权限检查**：同步接口用 `requireFileAccess`（owner/admin/user_access），管理接口用 `checkFilePermission`（仅 owner）
5. **共享权限不可传递**：被共享用户不能再共享或转移所有权
6. **Header 认证本质**：只是密码登录的信任代理，auth_method 仍为 'password'
7. **管理员创建的用户无法密码登录**：因为密码登录只认 `user_name=''` 的用户

### ⚠️ 风险推断总结

1. **Token 过期不一致**：密码登录用户的 Token 实际过期时间比配置值长 60 倍
2. **所有权转移不清理**：转移所有权后，旧所有者可能因 user_access 表残留记录而保留访问权限
3. **Header 认证信任链**：如果代理层配置不当，可能导致信任链断裂
4. **单用户密码模式限制**：密码登录模式下无法真正支持多用户登录

这种多层级的设计确保了即使某一层被绕过，下一层仍然能够提供防护，有效阻断越权访问。
