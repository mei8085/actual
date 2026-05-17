# 管理员侧用户目录与访问授权机制报告

## 1. 目录的读取方式

### 1.1 数据库表结构

用户目录数据存储在 `account.sqlite` 数据库的 `users` 表中：

```sql
CREATE TABLE users (
    id TEXT PRIMARY KEY,
    user_name TEXT,
    display_name TEXT,
    role TEXT,
    enabled INTEGER NOT NULL DEFAULT 1,
    owner INTEGER NOT NULL DEFAULT 0
);
```

### 1.2 服务端读取实现

**位置**: `packages/sync-server/src/services/user-service.ts:62-68`

```typescript
export function getAllUsers() {
  return getAccountDb().all(
    `SELECT users.id, user_name as userName, display_name as displayName, 
            enabled, ifnull(owner,0) as owner, role
     FROM users
     WHERE users.user_name <> ''`,
  );
}
```

### 1.3 API 端点

**位置**: `packages/sync-server/src/app-admin.js:29-38`

```javascript
app.get('/users/', validateSessionMiddleware, (req, res) => {
  const users = UserService.getAllUsers();
  res.json(
    users.map(u => ({
      ...u,
      owner: u.owner === 1,
      enabled: u.enabled === 1,
    })),
  );
});
```

### 1.4 客户端调用流程

**位置**: `packages/loot-core/src/server/admin/app.ts:37-58`

```typescript
async function getUsers() {
  const userToken = await asyncStorage.getItem('user-token');
  if (userToken) {
    const res = await get(getServer().BASE_SERVER + '/admin/users/', {
      headers: { 'X-ACTUAL-TOKEN': userToken },
    });
    if (res) {
      return JSON.parse(res) as UserEntity[];
    }
  }
  return null;
}
```

## 2. 角色与权限关系

### 2.1 角色定义

**位置**: `packages/loot-core/src/shared/user.ts:1-4`

```typescript
export const PossibleRoles = {
  ADMIN: 'Admin',
  BASIC: 'Basic',
};
```

### 2.2 权限检查机制

**位置**: `packages/sync-server/src/account-db.js:139-145`

```javascript
export function isAdmin(userId) {
  return hasPermission(userId, 'ADMIN');
}

export function hasPermission(userId, permission) {
  return getUserPermission(userId) === permission;
}

export function getUserPermission(userId) {
  const accountDb = getAccountDb();
  const { role } = accountDb.first(
    `SELECT role FROM users WHERE users.id = ?`,
    [userId],
  ) || { role: '' };
  return role;
}
```

### 2.3 角色权限对比

| 权限/能力 | BASIC 角色 | ADMIN 角色 | Owner |
|-----------|-----------|-----------|-------|
| 创建预算 | ✅ | ✅ | ✅ |
| 管理自己的预算 | ✅ | ✅ | ✅ |
| 访问所有用户预算 | ❌ | ✅ | ✅ |
| 添加新用户 | ❌ | ✅ | ✅ |
| 删除用户 | ❌ | ✅ | ✅ |
| 转移预算所有权 | ❌ | ✅ | ✅ |
| 更改服务器密码 | ❌ | ✅ | ✅ |
| 管理服务器配置 | ❌ | ✅ | ✅ |

### 2.4 管理员权限验证

**位置**: `packages/sync-server/src/app-admin.js:40-48`

```javascript
app.post('/users', validateSessionMiddleware, async (req, res) => {
  if (!isAdmin(res.locals.user_id)) {
    res.status(403).send({
      status: 'error',
      reason: 'forbidden',
      details: 'permission-not-found',
    });
    return;
  }
  // ... 后续操作
});
```

## 3. 对家庭与协作预算的影响

### 3.1 用户访问授权表

```sql
CREATE TABLE user_access (
  user_id TEXT,
  file_id TEXT,
  PRIMARY KEY (user_id, file_id),
  FOREIGN KEY (user_id) REFERENCES users(id),
  FOREIGN KEY (file_id) REFERENCES files(id)
);
```

### 3.2 文件权限检查

**位置**: `packages/sync-server/src/app-sync.ts:119-129`

```typescript
function requireFileAccess(file: File, userId: string) {
  const isOwner = file.owner === userId;
  const isServerAdmin = isAdmin(userId);
  if (isOwner || isServerAdmin) {
    return null;
  }
  if (UserService.countUserAccess(file.id, userId) > 0) {
    return null;
  }
  return 'file-access-not-allowed';
}
```

### 3.3 文件列表可见性

**位置**: `packages/sync-server/src/app-sync/services/files-service.ts:167-189`

```typescript
find({ userId, limit = 1000 }: { userId: string; limit?: number }) {
  const canSeeAll = isAdmin(userId);
  return (
    canSeeAll
      ? this.accountDb.all('SELECT * FROM files WHERE deleted = 0 LIMIT ?', [limit])
      : this.accountDb.all(
          `SELECT files.* FROM files WHERE files.owner = ? and deleted = 0
           UNION
           SELECT files.* FROM files
           JOIN user_access ON user_access.file_id = files.id
             AND user_access.user_id = ?
           WHERE files.deleted = 0 LIMIT ?`,
          [userId, userId, limit],
        )
  ).map((item: RawFile) => this.validate(item));
}
```

### 3.4 协作预算的权限流程

1. **预算创建者**：自动成为文件所有者（`files.owner` 字段）
2. **添加协作者**：通过 `/admin/access` 端点添加记录到 `user_access` 表
3. **协作者访问**：通过 `user_access` 表关联获得访问权限
4. **管理员访问**：绕过权限检查，可访问所有预算文件

## 4. 与同步服务的鉴权链路

### 4.1 登录认证流程

**位置**: `packages/sync-server/src/accounts/password.js:35-111`

```javascript
export function loginWithPassword(password) {
  // 1. 验证密码
  const confirmed = bcrypt.compareSync(password, passwordHash);
  
  // 2. 获取或创建会话 token
  const token = sessionRow ? sessionRow.token : uuidv4();
  
  // 3. 关联用户 ID
  let userId = getOrCreateUserId();
  
  // 4. 创建/更新会话
  accountDb.mutate(
    'INSERT INTO sessions (token, expires_at, user_id, auth_method) VALUES (?, ?, ?, ?)',
    [token, expiration, userId, 'password'],
  );
  
  return { token };
}
```

### 4.2 会话验证中间件

**位置**: `packages/sync-server/src/util/middlewares.ts:33-45`

```typescript
const validateSessionMiddleware = async (
  req: Request,
  res: Response,
  next: NextFunction,
) => {
  const session = await validateSession(req, res);
  if (!session) {
    return;
  }
  res.locals = session;
  next();
};
```

### 4.3 Token 验证实现

**位置**: `packages/sync-server/src/util/validate-user.ts:10-42`

```typescript
export function validateSession(req: Request, res: Response) {
  let { token } = req.body || {};
  if (!token) {
    token = req.headers['x-actual-token'];
  }

  const session = getSession(token);

  if (!session) {
    res.status(401);
    res.send({ status: 'error', reason: 'unauthorized', details: 'token-not-found' });
    return null;
  }

  if (session.expires_at !== TOKEN_EXPIRATION_NEVER &&
      session.expires_at * MS_PER_SECOND <= Date.now()) {
    res.status(401);
    res.send({ status: 'error', reason: 'token-expired' });
    return null;
  }

  return session;
}
```

### 4.4 完整鉴权链路

```
客户端请求
    ↓
[HTTP Header: X-ACTUAL-TOKEN]
    ↓
validateSessionMiddleware
    ↓
validateSession() → 查询 sessions 表
    ↓
会话信息注入 res.locals (user_id, auth_method)
    ↓
业务逻辑 → isAdmin(res.locals.user_id) 或 requireFileAccess()
    ↓
返回响应
```

### 4.5 同步端点的鉴权

**位置**: `packages/sync-server/src/app-sync.ts:44-47`

```typescript
const app = express();
app.use(validateSessionMiddleware);  // 所有 /sync/* 端点都需要鉴权
```

同步操作中的权限检查：
- `POST /sync/sync` - 检查文件访问权限
- `POST /sync/upload-user-file` - 检查文件访问权限（新文件自动归属当前用户）
- `GET /sync/download-user-file` - 检查文件访问权限
- `GET /sync/list-user-files` - 根据角色过滤可见文件

## 5. 权限变更下发到客户端会话的机制

### 5.1 权限变更的触发点

#### 5.1.1 用户角色更新

**位置**: `packages/sync-server/src/app-admin.js:92-142`

```javascript
app.patch('/users', validateSessionMiddleware, async (req, res) => {
  if (!isAdmin(res.locals.user_id)) { /* 403 */ }
  
  // 更新用户角色
  UserService.updateUserWithRole(
    userIdInDb, userName, displayName, enabled ? 1 : 0, role,
  );
  
  res.status(200).send({ status: 'ok', data: { id: userIdInDb } });
});
```

#### 5.1.2 用户访问授权变更

**位置**: `packages/sync-server/src/app-admin.js:218-271`

```javascript
app.post('/access', (req, res) => {
  const session = validateSession(req, res);
  // 检查当前用户是否有权限授权
  const { granted } = UserService.checkFilePermission(
    userAccess.fileId, session.user_id,
  ) || { granted: 0 };
  
  if (granted === 0 && !isAdmin(session.user_id)) { /* 400 */ }
  
  UserService.addUserAccess(userAccess.userId, userAccess.fileId);
  res.status(200).send({ status: 'ok', data: {} });
});
```

### 5.2 权限检查的核心机制

#### 5.2.1 服务端无状态权限检查

权限检查**不依赖会话缓存**，每次请求都会实时查询数据库：

**位置**: `packages/sync-server/src/account-db.js:143-145`

```javascript
export function getUserPermission(userId) {
  const accountDb = getAccountDb();
  const { role } = accountDb.first(
    `SELECT role FROM users WHERE users.id = ?`,
    [userId],
  ) || { role: '' };
  return role;
}
```

**文件权限检查**（每次同步/下载操作都会执行）：

**位置**: `packages/sync-server/src/app-sync.ts:119-129`

```typescript
function requireFileAccess(file: File, userId: string) {
  const isOwner = file.owner === userId;
  const isServerAdmin = isAdmin(userId);  // 每次调用都会查询数据库
  if (isOwner || isServerAdmin) {
    return null;
  }
  if (UserService.countUserAccess(file.id, userId) > 0) {  // 查询 user_access 表
    return null;
  }
  return 'file-access-not-allowed';
}
```

#### 5.2.2 客户端权限检查

客户端基于 Redux Store 中缓存的 `user.data.permission` 进行 UI 层面的权限控制：

**位置**: `packages/desktop-client/src/auth/AuthProvider.tsx:23-32`

```typescript
const hasPermission = (permission?: Permissions) => {
  if (!permission) {
    return true;
  }
  return (
    !serverUrl ||
    userData?.permission?.toUpperCase() === permission?.toUpperCase()
  );
};
```

### 5.3 权限变更生效时序详解

#### 5.3.1 立即生效场景（服务端侧）

以下变更在数据库写入完成后，**下一次 API 请求立即生效**：

| 操作 | 生效时机 | 检查点 |
|-----|---------|--------|
| 用户角色 BASIC → ADMIN | 数据库事务提交后 | `isAdmin(userId)` 每次调用查询最新 role |
| 用户角色 ADMIN → BASIC | 数据库事务提交后 | `isAdmin(userId)` 返回 false |
| 授予预算访问权限 | `user_access` 记录插入后 | `countUserAccess(fileId, userId)` 返回 > 0 |
| 撤销预算访问权限 | `user_access` 记录删除后 | `countUserAccess(fileId, userId)` 返回 0 |
| 启用/禁用用户 | `users.enabled` 字段更新后 | 登录时检查（OpenID 模式） |

**关键代码**（文件列表查询时的权限过滤）：

**位置**: `packages/sync-server/src/app-sync/services/files-service.ts:167-189`

```typescript
find({ userId, limit = 1000 }: { userId: string; limit?: number }) {
  const canSeeAll = isAdmin(userId);  // 每次查询都会检查最新角色
  return (
    canSeeAll
      ? this.accountDb.all('SELECT * FROM files WHERE deleted = 0 LIMIT ?', [limit])
      : this.accountDb.all(
          `SELECT files.* FROM files WHERE files.owner = ? and deleted = 0
           UNION
           SELECT files.* FROM files
           JOIN user_access ON user_access.file_id = files.id
             AND user_access.user_id = ?
           WHERE files.deleted = 0 LIMIT ?`,
          [userId, userId, limit],
        )
  ).map((item: RawFile) => this.validate(item));
}
```

#### 5.3.2 依赖下一次校验场景（客户端侧）

客户端 UI 层面的权限展示依赖 `getUserData()` 轮询获取最新信息：

**客户端触发时机**：

1. **应用初始化**：`LoggedInUser` 组件挂载时调用 `initializeUserData()`
   **位置**: `packages/desktop-client/src/components/LoggedInUser.tsx:59-71`
   ```typescript
   const initializeUserData = useCallback(async () => {
     try {
       await dispatch(getUserData());
     } catch (error) {
       console.error('Failed to initialize user data:', error);
     } finally {
       setLoading(false);
     }
   }, [dispatch]);

   useEffect(() => {
     void initializeUserData();
   }, [initializeUserData]);
   ```

2. **同步状态变化**：从离线恢复到在线时重新拉取用户信息
   **位置**: `packages/desktop-client/src/components/LoggedInUser.tsx:73-92`
   ```typescript
   useEffect(() => {
     return listen('sync-event', ({ type }) => {
       const shouldReinitialize =
         userData &&
         ((type === 'success' && userData.offline) ||
           (type === 'error' && !userData.offline));

       if (shouldReinitialize) {
         void initializeUserData();
       }
     });
   }, [initializeUserData, userData]);
   ```

3. **用户手动刷新**：预算文件列表页面的刷新按钮
   **位置**: `packages/desktop-client/src/components/manager/BudgetFileSelection.tsx:588-591`
   ```typescript
   const refresh = () => {
     void dispatch(getUserData());
     void dispatch(loadAllFiles());
   };
   ```

4. **打开预算列表**：进入文件管理页面时
   **位置**: `packages/desktop-client/src/components/manager/BudgetFileSelection.tsx:593-596`
   ```typescript
   const initialMount = useInitialMount();
   if (initialMount && quickSwitchMode) {
     refresh();
   }
   ```

#### 5.3.3 需要等待会话过期场景

以下操作**不会立即失效现有会话**，需要等待 token 过期：

| 操作 | 现有会话行为 | 失效时机 |
|-----|-------------|---------|
| 禁用用户（enabled = 0） | 现有 Token 仍然有效，可继续访问 | Token 自然过期 / 用户重新登录时检查 |
| 删除用户 | 现有 Token 仍然有效 | Token 自然过期 / 下次请求时用户不存在 |
| 修改密码（密码模式） | 不清空现有会话（单用户模式） | 无（单用户永不过期） |

**关键发现**：在 OpenID 登录流程中，会检查 `enabled` 字段，但**会话验证中间件不检查**：

**位置**: `packages/sync-server/src/accounts/openid.ts:285-289`
```javascript
const { id: userIdFromDb, display_name: displayName } =
  accountDb.first(
    'SELECT id, display_name FROM users WHERE user_name = ? and enabled = 1',
    [identity],
  ) || {};
```

但在 `validateSession` 中**不检查 `enabled` 状态**：

**位置**: `packages/sync-server/src/util/validate-user.ts:10-42`
```typescript
export function validateSession(req: Request, res: Response) {
  // 仅检查 token 是否存在和是否过期
  // 不检查 users.enabled 字段
}
```

### 5.4 主动失效机制

#### 5.4.1 更改登录方式时的会话清理

**位置**: `packages/sync-server/src/account-db.js:147-158`

```javascript
export async function enableOpenID(loginSettings) {
  // ... 配置 OpenID
  getAccountDb().mutate('DELETE FROM sessions');  // 清除所有会话，强制所有用户重新登录
}
```

#### 5.4.2 Token 过期机制

- **密码模式默认**：`TOKEN_EXPIRATION_NEVER = -1`（永不过期）
- **可配置过期**：`config.get('token_expiration')` 分钟
- **OpenID 模式**：可配置为跟随 OpenID Provider 的过期时间
- **定期清理**：`clearExpiredSessions()` 在每次登录时清理过期 1 小时以上的会话

**位置**: `packages/sync-server/src/account-db.js:259-268`
```javascript
export function clearExpiredSessions() {
  const clearThreshold = Math.floor(Date.now() / 1000) - 3600;
  const deletedSessions = getAccountDb().mutate(
    'DELETE FROM sessions WHERE expires_at <> -1 and expires_at < ?',
    [clearThreshold],
  ).changes;
  console.log(`Deleted ${deletedSessions} old sessions`);
}
```

### 5.5 完整权限变更下发时序图

```
管理员在 UI 执行操作
    │
    ├─→ 发送 API 请求（PATCH /admin/users）
    │     │
    │     └─→ 服务端更新 users 表（事务提交）
    │           │
    │           ├─→ [立即生效] 后续所有 API 请求使用新权限
    │           │     │
    │           │     ├─→ 同步请求：requireFileAccess() 检查最新权限
    │           │     ├─→ 文件列表：find() 基于最新角色过滤
    │           │     └─→ 下载请求：检查文件访问权限
    │           │
    │           └─→ [客户端感知] 依赖触发时机
    │                 │
    │                 ├─→ 应用初始化：LoggedInUser 调用 getUserData()
    │                 ├─→ 离线转在线：sync-event 触发重新拉取
    │                 ├─→ 手动刷新：用户点击刷新按钮
    │                 └─→ 进入文件列表：BudgetFileSelection 刷新
    │                       │
    │                       ├─→ GET /account/validate
    │                       ├─→ 返回最新 permission 字段
    │                       ├─→ 更新 Redux state.user.data
    │                       └─→ UI 重新渲染（隐藏/显示管理员功能）
    │
    └─→ [无主动推送] 服务端不主动通知客户端权限变更
```

### 5.6 权限变更矩阵

| 变更类型 | 服务端生效 | 客户端 UI 生效 | 现有会话 | 备注 |
|---------|-----------|--------------|---------|------|
| 角色 BASIC→ADMIN | 立即（下一次请求） | 下一次 getUserData() | 继续有效 | 用户立即拥有所有文件访问权 |
| 角色 ADMIN→BASIC | 立即（下一次请求） | 下一次 getUserData() | 继续有效 | 用户失去其他用户文件访问权 |
| 授予文件访问 | 立即（下一次请求） | 下一次 loadAllFiles() | 继续有效 | 用户可以看到并同步该文件 |
| 撤销文件访问 | 立即（下一次请求） | 下一次 loadAllFiles() | 继续有效 | 用户无法同步但仍可看本地数据 |
| 禁用用户 | 登录时检查 | 下一次 getUserData() | 继续有效直到过期 | 现有会话不受影响 |
| 删除用户 | 立即（用户不存在） | 下一次 getUserData() | 继续有效直到过期 | 文件所有权自动转移 |
| 修改服务器密码 | 不影响 | 不影响 | 继续有效 | 单用户模式不清空会话 |
| 切换登录方式 | 立即失效 | 立即失效 | 全部删除 | DELETE FROM sessions |

## 6. 对历史数据可见性的影响

### 6.1 数据存储模型

Actual Budget 采用 **本地优先（Local-First）** 架构：

1. **客户端**：SQLite 数据库存储完整预算数据（加密）
2. **同步服务**：
   - `files` 表：文件元数据（所有者、加密信息等）
   - `user_access` 表：用户-文件访问关系
   - 同步数据：存储在文件系统（`userFiles/` 目录）

### 6.2 权限变更对历史数据的影响

#### 6.2.1 用户角色从 BASIC 提升为 ADMIN

**影响范围**：
- ✅ 立即可以列出所有用户的预算文件（通过 `list-user-files`）
- ✅ 可以下载并同步任何预算文件
- ✅ 可以转移预算所有权
- ❌ 客户端本地已缓存的数据不受影响

**关键代码**: `files-service.ts:167-189` - `find()` 方法根据 `isAdmin(userId)` 决定返回全部文件还是仅授权文件

#### 6.2.2 用户角色从 ADMIN 降级为 BASIC

**影响范围**：
- ✅ 立即无法列出其他用户的私有预算
- ✅ 无法访问未被显式授权的预算
- ⚠️ **重要**：客户端本地已同步的数据仍然存在于本地数据库中
- ⚠️ 但无法继续同步这些预算文件的新变更

#### 6.2.3 用户被禁用（enabled = 0）

**影响范围**：
- ✅ 无法登录（登录时会检查用户状态）
- ✅ 现有会话 Token 仍然有效直到过期
- ❌ 禁用不会自动删除现有会话

#### 6.2.4 移除用户的预算访问权限

**影响范围**：
- ✅ 立即无法同步该预算的新变更
- ✅ 无法下载该预算文件
- ⚠️ 客户端本地已有的数据副本仍然存在（SQLite 数据库文件）
- ⚠️ 用户可以继续查看本地数据，但无法同步

#### 6.2.5 删除用户

**位置**: `packages/sync-server/src/app-admin.js:144-178`

```javascript
app.delete('/users', validateSessionMiddleware, async (req, res) => {
  // ...
  ids.forEach(item => {
    UserService.deleteUserAccess(item);  // 删除所有访问授权
    UserService.transferAllFilesFromUser(ownerId, item);  // 转移文件所有权
    UserService.deleteUser(item);  // 删除用户
  });
});
```

**影响范围**：
- ✅ 用户被删除，无法登录
- ✅ 用户的所有预算文件所有权转移给服务器 Owner
- ✅ 用户的所有访问授权被清除
- ⚠️ 客户端本地数据仍然存在（但无法同步）

### 6.3 历史数据可见性总结

| 权限变更操作 | 服务端访问控制 | 客户端已缓存数据 | 同步能力 |
|------------|--------------|----------------|---------|
| BASIC → ADMIN | 立即允许访问所有文件 | 需要重新同步获取 | ✅ 恢复 |
| ADMIN → BASIC | 立即限制可见文件 | 已缓存数据仍可见 | ❌ 未授权文件 |
| 禁用用户 | 阻止新登录 | 本地数据仍可见 | ❌ Token 过期后 |
| 移除预算访问 | 阻止同步/下载 | 本地数据仍可见 | ❌ 无法同步 |
| 删除用户 | 阻止登录+转移所有权 | 本地数据仍可见 | ❌ 无法同步 |

### 6.4 数据安全考虑

1. **加密保护**：预算文件使用端到端加密，即使获得文件访问权限也需要加密密钥
2. **本地数据残留**：权限撤销不会删除客户端本地已缓存的数据
3. **所有权转移**：删除用户时自动转移文件所有权，避免数据丢失
4. **会话清理**：重大安全变更（如改密码、切换登录方式）会清除所有会话

## 7. 关键代码位置索引

| 功能模块 | 文件路径 |
|---------|---------|
| 用户类型定义 | `packages/loot-core/src/types/models/user.ts` |
| 用户访问类型 | `packages/loot-core/src/types/models/user-access.ts` |
| 角色定义 | `packages/loot-core/src/shared/user.ts` |
| 管理员 API（客户端） | `packages/loot-core/src/server/admin/app.ts` |
| 管理员 API（服务端） | `packages/sync-server/src/app-admin.js` |
| 用户服务 | `packages/sync-server/src/services/user-service.ts` |
| 认证 API | `packages/loot-core/src/server/auth/app.ts` |
| 账户数据库操作 | `packages/sync-server/src/account-db.js` |
| 会话验证 | `packages/sync-server/src/util/validate-user.ts` |
| 同步服务权限检查 | `packages/sync-server/src/app-sync.ts` |
| 文件服务 | `packages/sync-server/src/app-sync/services/files-service.ts` |
| 密码登录 | `packages/sync-server/src/accounts/password.js` |
| 数据库迁移（多用户） | `packages/sync-server/migrations/1719409568000-multiuser.js` |
