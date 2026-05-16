# Subscription Login & OIDC 流程分析文档（已校正）

## 概述

Actual Budget 的登录系统采用模块化设计，支持多种认证方式：
- **密码认证** (Password)
- **OpenID Connect (OIDC)** 认证
- **Header** 认证 (反向代理场景)

整个系统围绕 `bootstrap` (服务器初始化) 和 `login` (用户登录) 两个核心状态流转，配合服务端交互建立登录态。

---

## 1. Bootstrap 流程（服务器初始化）

### 1.1 核心逻辑位置
- **前端 Bootstrap 页面**: `packages/desktop-client/src/components/manager/subscribe/Bootstrap.tsx`
- **后端 bootstrap 函数**: `packages/sync-server/src/account-db.js:81-137`
- **后端 /bootstrap 端点**: `packages/sync-server/src/app-account.js:59-67`

### 1.2 Bootstrap 触发时机
`useBootstrapped()` Hook (`common.tsx:25-102`) 是状态流转的核心，它在 `/login` 和 `/bootstrap` 页面自动执行：

```
用户访问页面
    ↓
useBootstrapped() 自动执行
    ↓
调用 subscribe-needs-bootstrap 查询服务器状态
    ↓
调用 /needs-bootstrap 端点
    ↓
auth 表为空?
    ├─ 是 → 重定向到 /bootstrap (首次初始化)
    └─ 否 → 重定向到 /login
```

### 1.3 Bootstrap 初始化流程的关键校正

> **重要发现**: Bootstrap 页面**只支持密码初始化**，不支持 OpenID 配置！
> OpenID 配置入口在 **Login 页面**，且有特定触发条件。

#### Bootstrap 页面（仅密码初始化）
```
用户访问 /bootstrap
    ↓
显示 ConfirmPasswordForm（仅密码输入框）
    ↓
用户输入密码并提交
    ↓
前端调用 subscribe-bootstrap { password }
    ↓
服务端 /bootstrap 端点（带 authRateLimiter）
    ↓
bootstrap() 函数执行：
    ├─ 检查：needsBootstrap() → auth 表是否为空
    ├─ 检查：不能同时启用 password + openId
    ├─ bootstrapPassword() 写入 auth 表
    │   (method='password', extra_data=bcryptHash)
    └─ 立即调用 loginWithPassword() 生成 session token
    ↓
返回 { status: 'ok', data: { token: 'xxx' } }
    ↓
前端刷新登录方法列表
    ↓
重定向到 /login
```

#### bootstrap() 函数边界条件（account-db.js:81-137）
| 条件 | 返回错误 |
|------|---------|
| loginSettings 为空 | `invalid-login-settings` |
| 既无 password 也无 openId | `no-auth-method-selected` |
| 同时有 password + openId（非 forced） | `max-one-method-allowed` |
| 已初始化（非 forced） | `already-bootstrapped` |

---

## 2. OpenID 配置入口校正

> **关键校正**: OpenID 配置**不在 Bootstrap 页面**，而是在**Login 页面**内！

### 2.1 OIDC 配置的两个入口

#### 入口 A: 首次登录配置（Login 页面）

**触发条件**:
1. `warnMasterCreation = true` → 尚无 owner 用户（`owner-created` 返回 false）
2. `askForPassword = true` → 可用登录方法包含 password

**流程**:
```
Login 页面 (OpenIdLogin 组件)
    ↓
显示 "Review OpenID configuration" 按钮
    ↓
用户输入 server password 后点击按钮
    ↓
前端调用 get-openid-config { password }
    ↓
服务端 /openid/config 端点（带 openIdConfigRateLimiter）
    ├─ 验证 ownerCount === 0（不能已有用户）
    ├─ 验证 password 正确
    ├─ 读取 auth 表中 openid 配置
    └─ 返回 { status: 'ok', data: { openId: config } }
    ↓
前端显示 OpenIdForm 表单（可编辑 issuer/client_id/client_secret）
    ↓
用户提交配置
    ↓
前端调用 subscribe-bootstrap { openId: config }
    ↓
服务端 bootstrap() 函数 (forced=false!)
    ↓
bootstrapOpenId() 执行:
    ├─ 验证 issuer/discoveryURL 存在
    ├─ 验证 client_id/client_secret 存在
    ├─ 验证 server_hostname 存在
    ├─ Issuer.discover() 尝试验证 OIDC 提供商
    └─ 写入 auth 表 (method='openid', active=1)
    ↓
成功 → 前端 navigate('/')
```

#### 入口 B: 已登录后启用 OIDC（管理员）

**入口路径**: 管理员设置页面（非 bootstrap/login 流程）

```
管理员已登录 (password 方式)
    ↓
进入设置 → 启用 OpenID
    ↓
前端调用 enable-openid { openId: config }
    ↓
服务端 /openid/enable 端点 (validateSessionMiddleware)
    ├─ 验证 isAdmin(res.locals.user_id)
    ├─ bootstrapOpenId() 配置 OIDC
    └─ DELETE FROM sessions（强制所有人重新登录）
```

### 2.2 OIDC 配置禁用流程

**端点**: `POST /openid/disable`

```
管理员输入当前密码
    ↓
disableOpenID() 执行:
    ├─ bcrypt.compareSync() 验证密码
    ├─ bootstrapPassword() 重新激活密码方式
    ├─ DELETE FROM sessions（清除所有会话）
    ├─ DELETE FROM users（清除 OIDC 用户，除空用户）
    └─ DELETE FROM auth WHERE method='openid'
    ↓
成功 → 退回密码登录方式
```

---

## 3. 登录流程（Login Flow）

### 3.1 登录方法选择逻辑

**后端 getLoginMethod() 优先级**（account-db.js:56-79）:
1. **前端指定** → `req.body.loginMethod`（需在 allowedLoginMethods 中）
2. **Header 强制** → 配置 loginMethod='header'（优先级 bypass 其他）
3. **当前激活** → `getActiveLoginMethod()` (auth 表 active=1)
4. **默认配置** → `config.get('loginMethod')`

### 3.2 /login 端点真实返回结构

**文件**: `packages/sync-server/src/app-account.js:74-129`

```
POST /login (authRateLimiter: 15分钟最多5次失败)
    ↓
getLoginMethod() 确定登录方式
    ↓
├─ 方式 A: header → 验证 x-actual-password header + validateAuthHeader()
├─ 方式 B: openid → loginWithOpenIdSetup() + 返回 redirectUrl
└─ 方式 C: password → loginWithPassword() + 返回 token
```

#### OpenID 登录分支返回结构：
```javascript
// 成功:
{ status: 'ok', data: { returnUrl: 'https://oidc-provider.com/auth?...' } }

// 失败 (400):
{ status: 'error', reason: 'Invalid redirect URL' | 'invalid-password' | ... }
```

#### 密码/Header 登录分支返回结构：
```javascript
// 成功:
{ status: 'ok', data: { token: 'uuid-session-token' } }

// 失败 (400):
{ status: 'error', reason: 'invalid-header' | 'proxy-not-trusted' | 'invalid-password' }
```

---

## 4. OIDC 完整流程三阶段

### 4.1 阶段一: 发起 OIDC 认证请求

**文件**: `packages/sync-server/src/accounts/openid.ts:101-176`

```
用户点击 "Sign in with OpenID" / "Start using OpenID"
    ↓
前端 subscribe-sign-in {
    returnUrl: window.location.origin (或 Electron OAuth server)
    loginMethod: 'openid'
    password?: '首次登录需验证server密码'
}
    ↓
服务端 loginWithOpenIdSetup(returnUrl, password)
    ├─ returnUrl 验证：isValidRedirectUrl() (hostname 匹配或 localhost)
    ├─ 首次登录特殊：countUsersWithUserName===0 且有password方法 → 验证server密码
    ├─ 从 auth 表读取 OIDC 配置并初始化 client
    ├─ 生成 state + code_verifier + code_challenge (PKCE)
    ├─ 写入 pending_openid_requests 表 (5分钟过期 expiry_time)
    └─ client.authorizationUrl() 生成认证URL
    ↓
返回 { redirectUrl: 'https://oidc-provider.com/auth?...' }
    ↓
前端跳转:
    ├─ 浏览器: window.location.href = redirectUrl
    └─ Electron: shell.openExternal(redirectUrl)
```

### 4.2 阶段二: OIDC 回调处理

**文件**: `packages/sync-server/src/app-openid.ts:99-113`
**回调端点**: `GET /openid/callback`

```
OIDC Provider 重定向回 Actual
    ↓
携带 query 参数: code, state, iss?
    ↓
/loginWithOpenIdFinalize(req.query) 执行:
    ├─ 验证 code 参数存在
    ├─ 验证 state 参数存在
    ├─ 验证 auth 表 openid 方法 active=1
    ├─ 查询 pending_openid_requests:
    │   └─ state 匹配 AND expiry_time > Date.now()
    ├─ client.callback() / client.grant() 交换 code → tokenSet
    ├─ client.userinfo(access_token) 获取用户信息
    ├─ 提取 identity (优先级: preferred_username > login > email > id > sub)
    ↓
用户创建/验证:
    ├─ 用户不存在 + (无现有用户 OR userCreationMode='login')
    │   └─ 创建新用户: 首个用户 owner=1/ADMIN, 后续 owner=0/BASIC
    └─ 用户已存在 → 验证 enabled=1
    ↓
生成 Session:
    ├─ token = uuidv4()
    ├─ expiration 策略: provider/never/自定义 (默认10分钟)
    └─ 写入 sessions 表 (token, expires_at, user_id, auth_method='openid')
    ↓
成功 → 302 重定向: return_url + `/openid-cb?token=${token}`
失败 → 400 返回 JSON: { status: 'error', reason: 'xxx' }
```

#### /openid/callback 真实错误码：
| 触发条件 | 返回 reason |
|---------|------------|
| code 参数缺失 | `missing-authorization-code` |
| state 参数缺失 | `missing-state` |
| OIDC 未配置/非激活 | `openid-not-configured` |
| state 过期或不匹配 | `invalid-or-expired-state` |
| Token 交换失败 | `openid-grant-failed` |
| 用户不存在或禁用 | `openid-grant-failed` |
| 无法提取用户身份标识 | `openid-grant-failed: no identification was found` |
| returnUrl 不在白名单 | `Invalid redirect URL` |

### 4.3 阶段三: 前端完成登录（OpenIdCallback 组件）

**文件**: `packages/desktop-client/src/components/manager/subscribe/OpenIdCallback.ts`

```
浏览器加载 /openid-cb 路由
    ↓
从 URL search 参数提取 token
    ↓
前端调用 subscribe-set-token { token }
    ↓
loot-core 服务层: asyncStorage.setItem('user-token', token)
    ↓
dispatch(loggedIn()) → 触发用户状态刷新
    ↓
用户成功进入应用
```

---

## 5. Token 写入到服务端会话验证的完整闭环

### 5.1 闭环流程图

```
【前端】                              【服务端】
  |                                     |
  |-- 1. OIDC callback 返回 token ---->|
  |                                     |
  |-- 2. subscribe-set-token --------->| (仅本地存储，无服务端请求)
  |     (asyncStorage.setItem)          |
  |                                     |
  |-- 3. dispatch(loggedIn()) --------->|
  |        ↓                            |
  |     getUser() 被调用                 |
  |        ↓                            |
  |-- 4. GET /validate ---------------->|
  |        Header: X-ACTUAL-TOKEN       |
  |                                     |
  |        validateSession()            |
  |        ├─ 从 header 提取 token      |
  |        ├─ getSession(token)         |
  |        │   查询 sessions 表         |
  |        ├─ 验证 token 未过期         |
  |        │   expires_at * 1000 > now  |
  |        └─ 返回 session              |
  |                                     |
  |        getUserInfo(user_id)         |
  |        └─ 查询 users 表             |
  |                                     |
  |<-- 返回用户信息 --------------------|
  |     userName, permission, etc.      |
  |                                     |
  |-- 5. 后续所有受保护请求 ------------>|
  |     均携带 X-ACTUAL-TOKEN header    |
  |     通过 validateSessionMiddleware  |
```

### 5.2 validateSession 验证细节

**文件**: `packages/sync-server/src/util/validate-user.ts:10-42`

```typescript
function validateSession(req, res) {
  // Token 提取优先级: body > header
  token = req.body.token || req.headers['x-actual-token'];

  // DB 查询 sessions 表
  session = getSession(token);

  if (!session) {
    // 返回 401: { status: 'error', reason: 'unauthorized', details: 'token-not-found' }
    return null;
  }

  // 过期验证: TOKEN_EXPIRATION_NEVER = -1 (永不过期)
  if (session.expires_at !== -1 &&
      session.expires_at * 1000 <= Date.now()) {
    // 返回 401: { status: 'error', reason: 'token-expired' }
    return null;
  }

  return session; // { token, user_id, auth_method, expires_at }
}
```

### 5.3 /validate 端点返回结构

**成功**:
```json
{
  "status": "ok",
  "data": {
    "validated": true,
    "userName": "user@example.com",
    "permission": "ADMIN",
    "userId": "uuid-user-id",
    "displayName": "User Name",
    "loginMethod": "openid",
    "prefs": {...}
  }
}
```

**失败 (401)**:
```json
{ "status": "error", "reason": "unauthorized", "details": "token-not-found" }
// 或
{ "status": "error", "reason": "token-expired" }
```

---

## 6. 失败分支对照表（按触发条件 → reason → 前端表现）

### 6.1 Bootstrap 相关

| 触发条件 | 返回 reason | 前端表现 |
|---------|------------|---------|
| password 为空 | `invalid-password` | Bootstrap 页面红色文字提示 |
| 两次密码不匹配 | `password-match` | Bootstrap 页面红色文字提示 |
| 服务器已初始化 | `already-bootstrapped` | 后端返回，前端重定向 |
| 同时启用两种方式 | `max-one-method-allowed` | 后端返回 |
| 无任何认证方式 | `no-auth-method-selected` | 后端返回 |

### 6.2 /login 端点（所有方式）

| 触发条件 | 返回 reason | 前端表现 |
|---------|------------|---------|
| header 值为空 | `invalid-header` | Login 页面提示 |
| 反向代理IP不在信任列表 | `proxy-not-trusted` | Login 页面提示 |
| 密码错误 | `invalid-password` | Login 页面红色文字提示 |
| 15分钟 >5次失败 | `too-many-requests` | rate-limiter 拦截 |
| redirectUrl 不在白名单 | `Invalid redirect URL` | 前端提示 |
| OIDC 未配置 | `openid-not-configured` | 前端提示 |
| returnUrl 为空 | `return-url-missing` | 前端提示 |
| returnUrl 无效 | `invalid-return-url` | 前端提示 |
| OIDC setup 失败 | `openid-setup-failed` | 前端提示 |

### 6.3 /openid/callback 端点

| 触发条件 | 返回 reason | 前端表现 |
|---------|------------|---------|
| code 参数缺失 | `missing-authorization-code` | 显示 400 JSON 错误页 |
| state 参数缺失 | `missing-state` | 显示 400 JSON 错误页 |
| state 过期或不匹配 | `invalid-or-expired-state` | 显示 400 JSON 错误页 |
| OIDC token 交换失败 | `openid-grant-failed` | 显示 400 JSON 错误页 |
| 用户不存在或已禁用 | `openid-grant-failed` | 显示 400 JSON 错误页 |
| 无用户身份标识 | `openid-grant-failed: no identification was found` | 显示 400 JSON 错误页 |
| 回调 URL 不在白名单 | `Invalid redirect URL` | 显示 400 JSON 错误页 |

### 6.4 /openid/config 端点（Review配置时）

| 触发条件 | 返回 reason |
|---------|------------|
| 已有 owner 用户 | `already-bootstraped` |
| 密码错误 | `invalid-password` |
| 15分钟 >5次失败 | `too-many-requests` |
| OIDC 配置不存在 | `OpenID configuration not found` |
| OIDC 配置 JSON 解析失败 | `Invalid OpenID configuration` |

### 6.5 Session 验证失败

| 触发条件 | 返回 reason | 前端表现 |
|---------|------------|---------|
| Token 在 DB 不存在 | `unauthorized` (details: token-not-found) | 前端 getUser() 返回 null → 视为未登录 |
| Token 已过期 | `token-expired` | 前端标记 tokenExpired=true → 提示重新登录 |
| 网络不可达 | - | 前端标记 offline=true |

---

## 7. 关键安全机制

### 7.1 Rate Limiting（速率限制）

```
authRateLimiter (app-account.js):
  - 作用端点: /login, /bootstrap
  - 窗口: 15分钟
  - 最大失败次数: 5
  - 成功请求不计入

openIdConfigRateLimiter (app-openid.ts):
  - 作用端点: /openid/config
  - 窗口: 15分钟
  - 最大请求次数: 5
```

### 7.2 PKCE（Proof Key for Code Exchange）
- OIDC 流程强制使用 PKCE，防止授权码拦截
- `generators.state()` + `generators.codeVerifier()` + `generators.codeChallenge()`
- state + code_verifier 关联存储，5分钟过期

### 7.3 Redirect URL 白名单验证
**文件**: `openid.ts:359-381`

```javascript
function isValidRedirectUrl(url) {
  // 仅允许:
  // 1. 与 OIDC 配置中 server_hostname 相同 hostname
  // 2. localhost（开发环境）
  const redirectHostname = new URL(url).hostname;
  const serverHostname = new URL(getServerHostname()).hostname;
  return redirectHostname === serverHostname || redirectHostname === 'localhost';
}
```

### 7.4 Header Auth 代理信任验证
**文件**: `validate-user.ts:44-67`
- `trustedAuthProxies` / `trustedProxies` 配置信任的 IP CIDR
- `ipaddr.subnetMatch()` 验证请求来源 IP
- 未验证 → 拒绝 Header 登录

---

## 8. 完整时序图（校正版）

```
前端 (React)                          服务端 (Express)                     OIDC Provider
    |                                     |                                      |
    |-- 1. GET /needs-bootstrap --------->|                                      |
    |                                     |-- 查询 auth 表                     |
    |<-- bootstrapped=false --------------|                                      |
    |                                     |                                      |
    |-- 用户访问 /bootstrap               |                                      |
    |-- 输入密码并提交                    |                                      |
    |-- 2. POST /bootstrap {password} --->|                                      |
    |                                     |-- bootstrapPassword()               |
    |                                     |-- 写入 auth 表                     |
    |                                     |-- 生成 session token               |
    |<-- { token } -----------------------|                                      |
    |-- 重定向到 /login                   |                                      |
    |                                     |                                      |
    |-- (首次 OIDC 配置场景)              |                                      |
    |-- 3. POST /openid/config {password} |                                      |
    |<-- 返回 OIDC 配置 ------------------|                                      |
    |-- 显示 OpenIdForm 编辑              |                                      |
    |-- 4. POST /bootstrap {openId} ----->|                                      |
    |                                     |-- bootstrapOpenId()                 |
    |                                     |-- 写入 auth 表 (openid, active=1)  |
    |<-- 成功 ----------------------------|                                      |
    |-- navigate('/')                     |                                      |
    |                                     |                                      |
    |-- 5. POST /login {                  |                                      |
    |       loginMethod: 'openid',        |                                      |
    |       returnUrl, password?          |                                      |
    |    } ------------------------------>|                                      |
    |                                     |                                      |
    |                                     |-- loginWithOpenIdSetup()            |
    |                                     |-- 生成 state/code_verifier          |
    |                                     |-- 写入 pending_openid_requests      |
    |<-- { returnUrl: oidcAuthUrl } ------|                                      |
    |                                     |                                      |
    |--------------------------------------------------> 6. 浏览器跳转 OIDC 登录 |
    |                                     |                                      |<-- 用户同意授权
    |                                     |                                      |-- 生成 code
    |<-- 7. 302 重定向 /openid/callback?code=xxx&state=xxx --------------------|
    |                                     |                                      |
    |                                     |-- 8. loginWithOpenIdFinalize()      |
    |                                     |-- 验证 state 有效                    |
    |                                     |-- 交换 code → tokenSet               |
    |                                     |<-- POST /token --------------------->|
    |                                     |-- 获取 userInfo                     |
    |                                     |<-- GET /userinfo ------------------->|
    |                                     |-- 创建/更新用户                     |
    |                                     |-- 写入 sessions 表                  |
    |                                     |                                      |
    |<-- 9. 302 重定向 /openid-cb?token=sessionToken ---------------------------|
    |                                     |                                      |
    |-- 10. OpenIdCallback 组件执行       |                                      |
    |-- subscribe-set-token {token}       |                                      |
    |    (写入 asyncStorage, 无服务端请求) |                                      |
    |-- dispatch(loggedIn())              |                                      |
    |    ↓                                |                                      |
    |-- getUser() 调用 /validate         |                                      |
    |-- 携带 X-ACTUAL-TOKEN header       |                                      |
    |-- 11. GET /validate --------------->|                                      |
    |                                     |-- validateSession()                  |
    |                                     |   查询 sessions 表                  |
    |                                     |   验证 token 有效                    |
    |                                     |-- getUserInfo()                      |
    |<-- 返回用户信息 ---------------------|                                      |
    |                                     |                                      |
    |-- 登录完成，进入应用                  |                                      |
```

---

## 9. 关键文件索引

| 功能模块 | 文件路径 |
|---------|---------|
| Bootstrap 页面 | `packages/desktop-client/src/components/manager/subscribe/Bootstrap.tsx` |
| Login 页面（含 OIDC 入口） | `packages/desktop-client/src/components/manager/subscribe/Login.tsx` |
| OIDC 回调完成登录 | `packages/desktop-client/src/components/manager/subscribe/OpenIdCallback.ts` |
| OIDC 配置表单 | `packages/desktop-client/src/components/manager/subscribe/OpenIdForm.tsx` |
| 通用 Hook (useBootstrapped) | `packages/desktop-client/src/components/manager/subscribe/common.tsx` |
| 错误页面 | `packages/desktop-client/src/components/manager/subscribe/Error.tsx` |
| 账户 DB 操作（bootstrap/getSession） | `packages/sync-server/src/account-db.js` |
| 账户 API 路由（/login,/bootstrap） | `packages/sync-server/src/app-account.js` |
| OIDC API 路由（/callback,/enable） | `packages/sync-server/src/app-openid.ts` |
| OIDC 核心逻辑（setup/finalize） | `packages/sync-server/src/accounts/openid.ts` |
| Session 验证函数 | `packages/sync-server/src/util/validate-user.ts` |
| loot-core 认证服务层 | `packages/loot-core/src/server/auth/app.ts` |

---

## 10. 校正总结

### 之前分析的主要错误 vs 真实代码

| 之前的错误描述 | 真实代码逻辑 |
|--------------|------------|
| Bootstrap 页面支持 OpenID 配置 | ❌ Bootstrap 页面只支持密码；OpenID 配置在 Login 页面内通过 Review 按钮触发 |
| OpenID bootstrap 直接写入 | ❌ 通过 get-openid-config 获取现有配置 → OpenIdForm 编辑 → subscribe-bootstrap 提交 |
| /openid/callback 失败跳转到前端错误页 | ❌ callback 失败直接返回 400 JSON，浏览器显示原始错误 |
| OIDC 有单独的 token 验证接口 | ❌ OIDC token 写入后，通过通用 /validate 端点验证会话 |
| bootstrap 只允许一种方式 | ✅ 正确，但 openId 需要 forced=true 标志 |
| 前端错误统一处理 | ❌ callback 失败无统一错误页，直接显示 JSON |

### 设计特点总结

1. **状态驱动**: `useBootstrapped()` 自动管理 `/bootstrap` ↔ `/login` 页面流转
2. **多层验证**: 前端表单验证 + 后端业务验证 + 安全边界验证
3. **OIDC 入口隐蔽**: 配置入口不在 bootstrap，在 login 页面且有触发条件
4. **会话闭环简洁**: OIDC callback 返回 token → 本地存储 → 通用 validate 端点验证
5. **错误分层**: API 失败统一返回 reason，前端根据 reason 本地化显示
6. **首次用户特殊**: OIDC 首个用户自动成为 owner，继承空用户文件权限
7. **安全优先**: PKCE、Rate Limit、Redirect 白名单、bcrypt 哈希全链路保护
