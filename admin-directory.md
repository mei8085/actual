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

Actual Budget 采用 **本地优先（Local-First）** 架构，这是理解数据可见性边界的核心：

```
┌─────────────────────────────────────────────────────────────┐
│                     客户端（浏览器/桌面端）                    │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ 本地 SQLite 数据库                                    │  │
│  │ ┌─────────────────┐  ┌────────────────────────────┐ │  │
│  │ │ 预算数据（所有）│  │ 加密密钥（本地存储）         │ │  │
│  │ │ 交易、分类、预算│  │ asyncStorage: encrypt-keys  │ │  │
│  │ └─────────────────┘  └────────────────────────────┘ │  │
│  └───────────────────────────────────────────────────────┘  │
│                            │                                │
│                            ▼ 同步（CRDT）                    │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                       同步服务端                              │
│  ┌─────────────────┐  ┌────────────┐  ┌─────────────────┐  │
│  │ files 表        │  │ user_access│  │ userFiles/ 目录  │  │
│  │ 元数据、所有者  │  │ 授权关系    │  │ 加密的预算文件   │  │
│  └─────────────────┘  └────────────┘  └─────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

**核心特征**：
1. **客户端**：SQLite 数据库存储完整预算数据（端到端加密）
2. **同步服务**：仅存储加密的文件内容和访问控制元数据
3. **同步协议**：基于 CRDT 的增量同步，服务端无法解密数据内容

### 6.2 场景一：角色变更的可见性边界

#### 6.2.1 BASIC → ADMIN（权限提升）

| 维度 | 服务端可见性 | 客户端本地残留 | 同步能力 |
|-----|------------|---------------|---------|
| **文件列表** | ✅ 立即可见所有用户文件（`list-user-files` 返回全部） | 仅原有缓存文件可见，新授权文件需主动下载 | ✅ 可同步所有文件 |
| **预算内容** | ✅ 可下载加密文件，但需要解密密钥 | 已缓存的预算数据不受影响 | ✅ 可同步新变更 |
| **管理操作** | ✅ 可转移所有权、删除文件 | 无 | 无 |

**服务端生效点**：
- `files-service.ts:167-189` - `find()` 方法中 `isAdmin(userId)` 返回 true，绕过 UNION 查询直接返回所有文件

**客户端感知延迟**：
- 直到调用 `loadAllFiles()` 才会在文件列表中看到新文件
- 新可见的文件需要主动点击下载才会同步到本地

#### 6.2.2 ADMIN → BASIC（权限降级）

| 维度 | 服务端可见性 | 客户端本地残留 | 同步能力 |
|-----|------------|---------------|---------|
| **文件列表** | ❌ 立即不可见其他用户私有文件（`list-user-files` 仅返回授权文件） | ⚠️ 已缓存的文件列表仍在 Redux Store 中，直到下一次 `loadAllFiles()` | ❌ 未授权文件 |
| **预算内容** | ❌ 服务端拒绝同步/下载请求 | ⚠️ **本地已同步的完整预算数据仍然可读**（在本地 SQLite 中） | ❌ 无法拉取新变更 |
| **管理操作** | ❌ 无法访问管理员 API | 无 | 无 |

**关键发现 - 本地数据残留风险**：

当管理员被降级为普通用户时，**客户端本地已经同步的预算数据不会被清除**：

1. 本地 SQLite 数据库中仍存储着其他用户预算的完整历史数据
2. 用户可以继续离线浏览这些数据
3. 仅阻止了新变更的同步
4. 如果用户知道如何直接访问 SQLite 文件，可以导出所有历史数据

**代码证据**：权限检查仅在同步时进行，不影响本地数据访问

**位置**: `packages/sync-server/src/app-sync.ts:170-175`
```typescript
const fileAccessError = requireFileAccess(currentFile, res.locals.user_id);
if (fileAccessError) {
  res.status(403);
  res.send(fileAccessError);
  return;
}
```

### 6.3 场景二：撤销授权的可见性边界

#### 6.3.1 撤销单个预算的访问权限

| 维度 | 服务端可见性 | 客户端本地残留 | 同步能力 |
|-----|------------|---------------|---------|
| **文件列表** | ❌ `list-user-files` 不再返回该文件 | ⚠️ 仍在 Redux Store 中，直到刷新 | ❌ |
| **同步操作** | ❌ `POST /sync/sync` 返回 403 | ⚠️ 本地完整数据仍可访问 | ❌ |
| **下载操作** | ❌ `GET /sync/download-user-file` 返回 403 | 无 | ❌ |
| **加密密钥** | 服务端不存储密钥 | ✅ 本地仍保留解密密钥 | 无 |

**最关键的安全边界**：

**服务端无法删除客户端本地已缓存的数据**。这是 Local-First 架构的固有特性：

1. 预算文件一旦下载到客户端，就完整存储在浏览器的 IndexedDB 或桌面端的文件系统中
2. 撤销授权仅阻止后续同步，不影响已存在的本地副本
3. 客户端的加密密钥也不会被撤销（密钥由用户管理，不在服务端存储）

### 6.4 场景三：禁用/删除用户的可见性边界

#### 6.4.1 禁用用户（enabled = 0）

| 维度 | 服务端可见性 | 客户端本地残留 | 同步能力 |
|-----|------------|---------------|---------|
| **新登录** | ❌ OpenID 登录时检查 `enabled = 1` | 无 | ❌ |
| **现有会话** | ⚠️ `validateSession` 不检查 `enabled`，现有 Token 继续有效 | ✅ 正常使用 | ✅ 直到 Token 过期 |
| **本地数据** | 无 | ✅ 完整可读 | ✅ 直到 Token 过期 |

**设计缺陷**：禁用用户不会立即使其会话失效

**位置**: `packages/sync-server/src/util/validate-user.ts:10-42`
```typescript
export function validateSession(req: Request, res: Response) {
  // 仅检查：
  // 1. Token 是否存在于 sessions 表
  // 2. Token 是否已过期
  // ❌ 不检查 users.enabled 状态
  // ❌ 不检查用户是否已被删除
}
```

这意味着：
- 被禁用的用户如果已有有效 Token（密码模式默认为永不过期），可以继续访问
- 需要手动删除该用户的会话记录才能立即生效

#### 6.4.2 删除用户

**删除用户时的数据库操作序列**：

**位置**: `packages/sync-server/src/app-admin.js:156-165`
```javascript
ids.forEach(item => {
  UserService.deleteUserAccess(item);           // 1. DELETE FROM user_access WHERE user_id = ?
  UserService.transferAllFilesFromUser(ownerId, item);  // 2. UPDATE files SET owner = ? WHERE owner = ?
  UserService.deleteUser(item);                // 3. DELETE FROM users WHERE id = ?
  // ❌ 不执行: DELETE FROM sessions WHERE user_id = ?
});
```

**按请求链路的实际行为分析**：

| 操作 | 验证阶段 | 实际结果 | HTTP 状态 |
|-----|---------|---------|----------|
| **`validateSession` 中间件** | 检查 sessions 表 | ✅ Token 仍存在且未过期 → 通过 | - |
| **`GET /account/validate`** | 1. validateSession<br>2. getUserInfo(user_id) | ❌ 用户不存在 → 返回错误 | 400 |
| **`GET /sync/list-user-files`** | 1. validateSession<br>2. isAdmin(userId) → false<br>3. UNION 查询（所有权 + user_access） | ✅ API 成功，但返回空列表 | 200 |
| **`POST /sync/sync`** | 1. validateSession<br>2. requireFileAccess() | ❌ 无所有权、非管理员、无授权 | 403 |
| **`GET /sync/download-user-file`** | 1. validateSession<br>2. requireFileAccess() | ❌ 同上 | 403 |
| **`POST /admin/*`** | 1. validateSession<br>2. isAdmin(userId) | ❌ 非管理员 | 403 |
| **`POST /sync/upload-user-file`（新文件）** | 1. validateSession<br>2. currentFile 不存在 → 跳过权限检查<br>3. 写入文件并创建记录 | ✅ 可创建新文件！owner = 已删除用户ID | 200 |
| **`POST /sync/upload-user-file`（覆盖现有）** | 1. validateSession<br>2. requireFileAccess() | ❌ 无所有权、非管理员、无授权 | 403 |

**各维度详细分析**：

| 维度 | 服务端行为 | 客户端表现 | 备注 |
|-----|----------|-----------|------|
| **会话有效性** | ✅ sessions 表记录保留 | ✅ 会话中间件通过 | Token 未过期则一直有效 |
| **身份验证** | ❌ `/validate` 返回 "User not found" | ⚠️ 客户端登出/显示错误 | 客户端检测到用户不存在 |
| **文件列表** | ✅ 返回空列表 | ⚠️ 看不到任何文件 | 所有权已转移 + 授权已清除 |
| **同步操作** | ❌ 全部 403 拒绝 | ⚠️ 同步失败提示 | 无法拉取新变更 |
| **下载操作** | ❌ 全部 403 拒绝 | ⚠️ 下载失败提示 | 无法下载新文件 |
| **本地数据** | 无控制 | ✅ 所有已同步预算完整可读 | 本地 SQLite 不受影响 |

### 6.5 删除用户后的请求链路时序

```
已删除用户的客户端（持有有效 Token）
    │
    ├─→ 调用 GET /account/validate
    │     ├─ validateSession() → ✅ 通过（Token 在 sessions 表中）
    │     ├─ getUserInfo(user_id) → ❌ 返回 null（用户已删除）
    │     └─→ 返回 400 "User not found"
    │           │
    │           └─→ 客户端：触发登出逻辑，清除本地用户状态
    │
    ├─→ 调用 GET /sync/list-user-files（如果未登出）
    │     ├─ validateSession() → ✅ 通过
    │     ├─ isAdmin(user_id) → getUserPermission() → role = '' → false
    │     ├─ UNION 查询：
    │     │   ├─ files.owner = user_id → 已转移 → 无结果
    │     │   └─ user_access.user_id = user_id → 已删除 → 无结果
    │     └─→ 返回 200 + 空数组
    │           │
    │           └─→ 客户端：文件列表为空
    │
    ├─→ 调用 POST /sync/sync（同步现有预算）
    │     ├─ validateSession() → ✅ 通过
    │     ├─ requireFileAccess()：
    │     │   ├─ isOwner → 所有权已转移 → false
    │     │   ├─ isAdmin → false
    │     │   └─ countUserAccess → 已删除 → 0
    │     └─→ 返回 403 "file-access-not-allowed"
    │           │
    │           └─→ 客户端：同步失败
    │
    └─→ 本地离线浏览
          └─→ ✅ 所有已同步的预算数据完整可读（本地 SQLite）
```

### 6.6 三类场景可见性边界对比矩阵

| 场景 | 操作 | `/validate` | 文件列表 | 同步 | 下载 | 本地数据可读 | 数据泄露风险 |
|-----|-----|------------|---------|-----|-----|------------|------------|
| **角色变更** | ADMIN→BASIC | ✅ 成功，返回 BASIC | ⚠️ 仅授权文件 | ❌ 403（未授权） | ❌ 403 | ✅ 完整历史 | **高** |
| **撤销授权** | 移除 user_access | ✅ 成功 | ❌ 不显示该文件 | ❌ 403 | ❌ 403 | ✅ 该预算完整历史 | **高** |
| **删除用户** | DELETE users | ❌ 400 User not found | ✅ 空列表 | ❌ 403 | ❌ 403 | ✅ 完整历史 | **中**（客户端会登出） |
| **禁用用户** | enabled=0 | ⚠️ 成功（不检查 enabled） | ✅ 正常 | ✅ 正常 | ✅ 正常 | ✅ 完整 | **高**（会话继续有效） |

### 6.7 之前报告结论的统一与澄清

**关于"删除用户后会话继续有效"的精确描述**：

| 之前的结论 | 澄清后的准确描述 | 证据 |
|----------|----------------|------|
| "现有会话继续有效直到过期" | ✅ 正确，但需补充：`/validate` 会返回 400，客户端通常会登出 | `app-account.js:188-195` |
| "用户可继续访问" | ⚠️ 技术上 API 层通过中间件，但业务端点均失败 | 同步/下载均返回 403 |
| "同步能力直到 Token 过期" | ⚠️ 技术上中间件通过，但 `requireFileAccess` 拒绝所有文件 | `app-sync.ts:170-175` |

**关键修正**：删除用户后，虽然 `validateSession` 中间件仍然通过（因为 sessions 表未清理），但：
1. `/validate` 端点会检测到用户不存在并返回错误 → 触发客户端登出
2. 所有文件相关操作都会被 `requireFileAccess` 拒绝（所有权转移 + 授权清除）
3. 实际有效攻击窗口很小，仅限于用户在客户端登出前的短暂时间

### 6.8 设计取舍分析

#### 6.8.1 核心架构取舍

**取舍 1：Local-First 架构 vs 集中式权限控制**

| 设计选择 | 优点 | 代价 |
|---------|------|------|
| **Local-First**（Actual 的选择） | 离线可用、性能好、用户数据主权 | 服务端无法控制客户端已缓存数据 |
| **集中式权限** | 管理方便、可即时撤销 | 依赖网络、单点故障 |

**对协作预算的影响**：
- ✅ 协作者可以离线编辑预算，同步时自动合并
- ❌ 撤销协作权限后，协作者仍保有本地完整历史数据
- ❌ 服务端无法强制擦除客户端数据

**取舍 2：无状态权限检查 vs 会话缓存**

| 设计选择 | 优点 | 代价 |
|---------|------|------|
| **每次请求查数据库**（Actual 的选择） | 权限变更立即生效（服务端侧）、实现简单 | 数据库查询开销 |
| **会话缓存权限** | 性能好 | 权限变更有延迟、需要失效机制 |

**对协作预算的影响**：
- ✅ 预算所有者撤销权限后，协作者的下一次同步请求立即被拒绝
- ✅ 无需复杂的会话失效机制
- ❌ 每次同步增加 2-3 次 SQLite 查询（性能影响可忽略）

**取舍 3：Token 永不过期 vs 定期重新认证**

| 设计选择 | 优点 | 代价 |
|---------|------|------|
| **永不过期（默认）**（Actual 的选择） | 用户体验好，无需频繁登录 | 禁用/删除用户无法立即生效 |
| **Token 过期** | 安全性好 | 用户体验差，需要重新登录 |

**对协作预算的影响**：
- ✅ 协作者无需频繁重新登录，协作体验流畅
- ❌ 被禁用的协作者如果保持浏览器打开，可以继续访问（直到 `/validate` 被调用）
- ❌ 多用户部署中存在安全窗口

#### 6.8.2 会话管理的设计权衡

**设计决策：不级联删除会话**

删除用户时不清理 sessions 表的设计选择：

| 优点 | 代价 |
|------|------|
| 实现简单，无需级联删除逻辑 | 删除用户后会话记录成为"孤儿"数据 |
| 避免意外删除其他用户的会话 | 存在理论上的攻击窗口（直到客户端调用 `/validate`） |
| | sessions 表会逐渐积累无效记录 |

**实际风险评估**：
- 攻击窗口很小：客户端通常会定期调用 `/validate`（应用启动、从离线恢复、刷新页面）
- `/validate` 检测到用户不存在后，客户端会自动登出
- 即使攻击者截获了 Token，也只能访问空的文件列表和被拒绝的同步请求

#### 6.8.3 权限检查层级的设计

Actual 采用了**多层防御**的权限检查设计：

```
请求入口
    ↓
validateSession（检查 Token 有效性）
    ↓
业务层权限检查
    ├─ 管理员操作 → isAdmin(userId)
    ├─ 文件列表 → find(userId) 基于角色过滤
    ├─ 同步操作 → requireFileAccess(file, userId)
    └─ 下载操作 → requireFileAccess(file, userId)
```

**每层检查的作用**：
1. **会话层**：确保请求来自已认证用户
2. **角色层**：控制管理功能访问
3. **资源层**：控制具体文件的访问权限

**设计优点**：
- 每层检查独立，不依赖上层结果
- 即使某一层出现漏洞，其他层仍提供保护
- `requireFileAccess` 是最核心的防线，覆盖所有文件操作

### 6.9 风险评估与改进建议

#### 6.9.1 已识别风险的精确评估

| 风险 | 严重性 | 可利用性 | 实际影响 | 建议优先级 |
|-----|--------|---------|---------|----------|
| **权限降级后历史数据泄露** | 高 | 高（用户主动操作） | 已同步的所有预算历史数据可被导出 | 🔴 高 |
| **撤销协作后历史数据残留** | 高 | 高（协作者主动操作） | 该预算的完整历史数据可被导出 | 🔴 高 |
| **禁用用户会话不失效** | 中 | 中（需保持会话不刷新） | 可继续访问直到 Token 过期或页面刷新 | 🟡 中 |
| **删除用户会话不清理** | 低 | 低（`/validate` 会触发登出） | 理论窗口极小，实际几乎无法利用 | 🟢 低 |

#### 6.9.2 对协作预算安全性的实际影响

**场景 1：企业内部协作（员工离职）**

| 操作 | 安全保障 | 残留风险 |
|-----|---------|---------|
| 撤销该员工对所有预算的访问权限 | ✅ 立即阻止同步新变更 | ⚠️ 本地已同步数据仍可查看和导出 |
| 将员工角色从 ADMIN 改为 BASIC | ✅ 立即阻止管理操作 | ⚠️ 之前作为管理员时同步的所有预算数据仍在本地 |
| 禁用该员工账号 | ⚠️ 现有会话继续有效 | ⚠️ 如果员工不刷新页面，可以继续访问 |

**实际建议**：
1. 高安全要求的企业应配置 Token 过期时间（如 24 小时）
2. 员工离职时应轮换预算加密密钥
3. 告知员工其本地数据的留存情况，要求其自行删除

**场景 2：家庭预算协作（伴侣分手）**

| 操作 | 安全保障 | 残留风险 |
|-----|---------|---------|
| 撤销前伴侣的预算访问权限 | ✅ 立即阻止同步新变更 | ⚠️ 历史交易记录、预算规划等敏感信息仍在其设备上 |
| 删除前伴侣的用户账号 | ⚠️ 触发客户端登出 | ⚠️ 本地数据仍然存在 |

**实际建议**：
1. 协作前应了解本地数据留存的特性
2. 敏感预算可考虑使用独立的预算文件
3. 关系结束时考虑迁移到新的预算文件（使用新的加密密钥）

**场景 3：临时协作（顾问、会计师）**

| 操作 | 安全保障 | 残留风险 |
|-----|---------|---------|
| 仅授予特定预算的访问权限 | ✅ 只能访问授权预算 | ⚠️ 授权期间同步的该预算历史数据留存 |
| 协作结束后撤销访问权限 | ✅ 立即阻止新同步 | ⚠️ 历史数据仍在顾问设备上 |

**实际建议**：
1. 仅授予必要的预算访问权限（最小权限原则）
2. 避免在协作预算中包含过于敏感的历史数据
3. 考虑使用数据导出功能而非直接共享预算

#### 6.9.3 建议的改进措施

**改进 1：增强会话验证（高优先级）**

在 `validateSession` 中增加用户状态检查：
```typescript
// packages/sync-server/src/util/validate-user.ts
export function validateSession(req: Request, res: Response) {
  // ... 现有 Token 检查 ...
  
  // 新增：检查用户是否存在且已启用
  const user = getUserInfo(session.user_id);
  if (!user) {
    res.status(401).send({ status: 'error', reason: 'user-deleted' });
    return null;
  }
  if (user.enabled !== 1) {
    res.status(401).send({ status: 'error', reason: 'user-disabled' });
    return null;
  }
  
  return session;
}
```

**改进 2：删除用户时清理会话（中优先级）**

在 `DELETE /admin/users` 中增加会话清理：
```javascript
// packages/sync-server/src/app-admin.js
ids.forEach(item => {
  UserService.deleteUserAccess(item);
  UserService.transferAllFilesFromUser(ownerId, item);
  UserService.deleteUser(item);
  // 新增：清理该用户的会话
  getAccountDb().mutate('DELETE FROM sessions WHERE user_id = ?', [item]);
});
```

**改进 3：文档化安全边界（高优先级）**

在管理员文档中明确说明：
- 权限撤销仅阻止新数据同步，不影响已缓存的本地数据
- 多用户部署建议配置合理的 Token 过期时间
- 高敏感场景应配合加密密钥轮换

**改进 4：客户端安全提示（低优先级）**

当权限被撤销时，客户端可以：
- 显示明确的权限变更提示
- 提供清理本地数据的选项（用户自愿选择）
- 解释本地数据留存的原因（Local-First 架构特性）

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
