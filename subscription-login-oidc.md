# Subscription Login & OIDC 流程分析文档

## 概述

Actual Budget 的登录系统采用模块化设计，支持多种认证方式：
- **密码认证** (Password)
- **OpenID Connect (OIDC)** 认证
- **Header** 认证 (反向代理场景)

整个系统围绕 `bootstrap` (初始化) 和 `login` (登录) 两个核心状态流转，配合服务端交互建立登录态。

---

## 1. Bootstrap 流程 (服务器初始化)

### 1.1 核心逻辑位置
- **前端**: `packages/desktop-client/src/components/manager/subscribe/Bootstrap.tsx`
- **服务端**: `packages/sync-server/src/app-account.js` → `/bootstrap` 端点

### 1.2 Bootstrap 触发时机
`useBootstrapped()` Hook (`common.tsx:25-102`) 是状态流转的核心，它在 `/login` 和 `/bootstrap` 页面自动执行：

```
用户访问页面
    ↓
useBootstrapped() 自动执行
    ↓
调用 subscribe-needs-bootstrap 查询服务器状态
    ↓
┌─────────────────────────────────────────┐
│  服务器未初始化?                         │
│  → 是: 重定向到 /bootstrap              │
│  → 否: 重定向到 /login                  │
└─────────────────────────────────────────┘
```

### 1.3 Bootstrap 初始化流程

**支持两种初始化方式：**

#### 方式 A: 密码初始化
```
Bootstrap 页面
    ↓
用户输入密码 (ConfirmPasswordForm)
    ↓
前端发送 subscribe-bootstrap { password }
    ↓
服务端 /bootstrap 端点
    ├─ 验证密码非空
    ├─ 写入 auth 表 (method='password')
    └─ 返回成功
    ↓
刷新登录方法列表
    ↓
重定向到 /login
```

#### 方式 B: OpenID 初始化
```
OpenIdForm 配置 OIDC 提供商
    ↓
用户输入: issuer URL, client_id, client_secret
    ↓
前端发送 subscribe-bootstrap { openId: config }
    ↓
服务端 bootstrapOpenId() 验证:
    ├─ issuer/discoveryURL 存在
    ├─ client_id 存在
    ├─ client_secret 存在
    ├─ server_hostname 存在
    └─ 尝试发现 OIDC 提供商 (Issuer.discover)
    ↓
写入 auth 表 (method='openid', extra_data=JSON配置)
    ↓
初始化成功 → 重定向到首页
```

### 1.4 Bootstrap 边界条件
| 条件 | 行为 |
|------|------|
| 服务器已初始化 | 拒绝再次 bootstrap，返回 `already-bootstraped` |
| 配置 OIDC 时发现失败 | 返回 `configuration-error` |
| 数据库写入失败 | 返回 `database-error` |
| 网络不可达 | 显示错误页面 `/error` |

---

## 2. 登录流程 (Login Flow)

### 2.1 核心组件位置
- **前端**: `packages/desktop-client/src/components/manager/subscribe/Login.tsx`
- **服务端**: `packages/sync-server/src/app-account.js` → `/login` 端点

### 2.2 登录方法选择逻辑
```
Login 页面加载
    ↓
useBootstrapped() 检查服务器状态
    ↓
获取可用登录方法 (availableLoginMethods)
    ↓
┌─────────────────────────────────────────┐
│  可用登录方法数量?                        │
│  → 1个: 直接使用该方法                    │
│  → 多个: 显示下拉菜单供用户选择           │
└─────────────────────────────────────────┘
```

### 2.3 三种登录方式详解

#### 方式 1: 密码登录 (Password)
```
用户输入密码
    ↓
发送 subscribe-sign-in { password, loginMethod: 'password' }
    ↓
服务端 /login 端点
    ↓
loginWithPassword() 验证
    ├─ 查询 auth 表获取密码哈希
    ├─ bcrypt 比对
    └─ 验证通过 → 生成 session token
    ↓
前端存储 token 到 asyncStorage
    ↓
dispatch(loggedIn()) → 完成登录
```

#### 方式 2: Header 登录 (反向代理场景)
```
页面自动检测 loginMethod === 'header'
    ↓
自动发送 subscribe-sign-in { password: '', loginMethod: 'header' }
    ↓
服务端验证:
    ├─ 读取 x-actual-password header
    ├─ validateAuthHeader() 验证代理信任
    └─ loginWithPassword() 验证密码
    ↓
成功 → dispatch(loggedIn())
失败 → 显示错误: invalid-header / proxy-not-trusted
```

#### 方式 3: OpenID Connect (OIDC) 登录
这是最复杂的流程，分为 **发起阶段** 和 **回调阶段**。

---

## 3. OIDC 完整流程详解

### 3.1 阶段一: 发起 OIDC 认证请求

```
用户点击 "Sign in with OpenID"
    ↓
Login.tsx: onSubmitOpenId()
    ↓
发送 subscribe-sign-in { returnUrl, loginMethod: 'openid', password? }
    ↓
服务端 /login 端点 (openid 分支)
    ↓
loginWithOpenIdSetup() 执行:
    ├─ 验证 returnUrl 合法性 (白名单: 同hostname或localhost)
    ├─ 首次登录特殊处理:
    │   └─ 若无用户但有密码方法 → 验证server密码
    ├─ 从数据库读取 OIDC 配置
    ├─ setupOpenIdClient() 初始化 OIDC 客户端
    ├─ 生成 state, code_verifier, code_challenge (PKCE)
    ├─ 存储 pending 请求到数据库 (5分钟过期)
    └─ 生成 authorizationUrl
    ↓
返回 redirectUrl 给前端
    ↓
前端跳转:
    ├─ Electron: 调用 shell.openExternal
    └─ 浏览器: window.location.href = redirectUrl
```

### 3.2 OIDC 配置存储结构
```sql
-- auth 表存储
INSERT INTO auth (method, display_name, extra_data, active)
VALUES ('openid', 'OpenID', JSON_STRINGIFY(config), 1)

-- config 包含:
{
  selectedProvider: "google" | "microsoft" | "github" | "other",
  issuer: "https://accounts.google.com",
  client_id: "xxx",
  client_secret: "xxx",
  server_hostname: "https://actual.example.com",
  authMethod?: "openid"  // 可选
}
```

### 3.3 阶段二: OIDC 回调处理

**回调端点**: `GET /openid/callback`

```
OIDC 提供商重定向回 Actual
    ↓
携带参数: code, state, iss?
    ↓
app-openid.ts: /callback 端点
    ↓
loginWithOpenIdFinalize() 执行:
    ├─ 验证 code 和 state 参数存在
    ├─ 从 pending_openid_requests 表查询:
    │   ├─ state 匹配
    │   └─ 未过期 (expiry_time > now)
    ├─ 读取 OIDC 配置并初始化客户端
    ├─ client.callback() 交换 code → tokenSet
    ├─ client.userinfo() 获取用户信息
    ├─ 提取 identity (优先级: preferred_username > login > email > id > sub)
    ↓
用户存在性检查:
    ├─ 数据库中无用户 AND (无现有用户 OR userCreationMode='login')
    │   └─ 创建新用户:
    │       ├─ 第一个用户 → ADMIN 角色 + owner=1
    │       └─ 后续用户 → BASIC 角色 + owner=0
    └─ 用户已存在 → 验证 enabled=1
    ↓
创建 session:
    ├─ 生成 UUID token
    ├─ 过期策略: provider/never/自定义 (默认10分钟)
    └─ 写入 sessions 表
    ↓
重定向到: return_url/openid-cb?token=xxx
```

### 3.4 阶段三: 前端完成登录

前端路由 `/openid-cb` 处理最终登录:

```
前端加载 /openid-cb 路由
    ↓
从 URL query 提取 token 参数
    ↓
发送 subscribe-set-token { token }
    ↓
存储 token 到 asyncStorage
    ↓
dispatch(loggedIn())
    ↓
登录完成 → 进入应用
```

---

## 4. 登录态建立与维持

### 4.1 Session Token 机制

**Token 生成**:
- 格式: UUID v4
- 存储: `sessions` 表
- 关联: user_id, auth_method, expires_at

**Token 验证流程**:
```
请求受保护接口
    ↓
validateSessionMiddleware() / validateSession()
    ├─ 读取 X-ACTUAL-TOKEN header
    ├─ 查询 sessions 表
    ├─ 验证 token 存在且未过期
    ├─ res.locals.user_id = session.user_id
    └─ res.locals.auth_method = session.auth_method
    ↓
通过 → 执行业务逻辑
失败 → 返回 401 unauthorized
```

### 4.2 前端登录状态管理

**核心上下文**: `ServerContext.tsx`
- `useAvailableLoginMethods()` - 获取可用登录方式
- `useLoginMethod()` - 获取当前默认登录方式
- `useRefreshLoginMethods()` - 刷新登录方法列表

**Redux 状态**: `usersSlice.loggedIn()`
- 触发登录完成后的状态更新
- 清除登录相关的临时状态

---

## 5. 错误页面与失败分支边界

### 5.1 错误页面 (Error Component)

**位置**: `packages/desktop-client/src/components/manager/subscribe/Error.tsx`

**触发场景**:
```
subscribe-needs-bootstrap 失败
    ↓
navigate('/error', { state: { error: reason } })
    ↓
Error 组件显示:
    ├─ network-failure → "无法访问服务器..."
    └─ 其他 → "服务器状态检查失败"
    ↓
提供 "重试" 按钮 → 返回首页
```

### 5.2 Bootstrap 错误边界

| 错误类型 | 错误码 | 触发场景 | 处理方式 |
|---------|--------|---------|---------|
| 密码空 | `invalid-password` | password='' | 前端表单验证+后端验证 |
| OIDC 配置缺失 | `missing-issuer` / `missing-client-id` / `missing-client-secret` | 配置不完整 | 前端+后端双重验证 |
| 网络失败 | `network-failure` | 服务端不可达 | 跳转到 /error 页面 |
| 已初始化 | `already-bootstraped` | 重复调用 bootstrap | 后端拒绝 |

### 5.3 Login 错误边界

| 错误类型 | 错误码 | 触发场景 | 处理方式 |
|---------|--------|---------|---------|
| 无效密码 | `invalid-password` | 密码错误 | 登录页显示红色提示 |
| Header 缺失 | `invalid-header` | 反向代理未传 header | 登录页显示 |
| 代理不信任 | `proxy-not-trusted` | IP不在信任列表 | 登录页显示 |
| 请求过多 | `too-many-requests` | 15分钟>5次失败 | rate limiter 拦截 |
| 内部错误 | `internal-error` | 服务端异常 | 通用错误提示 |

### 5.4 OIDC 特有错误边界

| 错误类型 | 错误码 | 触发场景 |
|---------|--------|---------|
| 配置错误 | `configuration-error` | OIDC provider 发现失败 |
| 未配置 | `openid-not-configured` | 未启用 OIDC |
| 无效重定向 | `invalid-return-url` | returnUrl 不在白名单 |
| State 过期 | `invalid-or-expired-state` | pending 请求过期或不匹配 |
| 授权失败 | `openid-grant-failed` | token 交换或 userinfo 失败 |
| 用户禁用 | `openid-grant-failed` | 用户 exists 但 enabled=0 |
| 无身份标识 | `openid-grant-failed: no identification was found` | OIDC 返回无可用用户ID |

### 5.5 关键安全边界

#### 1. Redirect URL 白名单验证
```typescript
// isValidRedirectUrl() - openid.ts:359-381
function isValidRedirectUrl(url) {
  const serverHostname = getServerHostname();
  const redirectUrl = new URL(url);
  const serverUrl = new URL(serverHostname);
  
  // 只允许:
  // 1. 与 server_hostname 同 hostname
  // 2. localhost (开发环境)
  return redirectUrl.hostname === serverUrl.hostname 
      || redirectUrl.hostname === 'localhost';
}
```

#### 2. Rate Limiting (速率限制)
- **位置**: `app-account.js:26-33`
- **策略**: 15分钟窗口内最多 5 次失败尝试
- **影响**: 登录、bootstrap 端点
- **跳过**: 成功的请求不计入计数

#### 3. PKCE (Proof Key for Code Exchange)
- OIDC 流程强制使用 PKCE
- 生成: `code_verifier` → `code_challenge` (S256)
- 存储: state + code_verifier 关联 pending 请求
- 防止: 授权码拦截攻击

---

## 6. 首次登录 OIDC 的特殊处理

### 6.1 Owner 用户创建逻辑
```
OIDC 回调时检查用户数量
    ↓
countUsersWithUserName === 0
    ↓
┌─────────────────────────────────────────┐
│  第一个登录的 OIDC 用户成为:             │
│  → owner = 1 (服务器所有者)              │
│  → role = 'ADMIN'                        │
│                                          │
│  同时:                                   │
│  → 将密码方法的空用户的所有文件转移给TA    │
│  → 保留原文件访问权限                     │
└─────────────────────────────────────────┘
```

### 6.2 Server Password 验证场景
```
尚无用户 + 有密码登录方法
    ↓
用户登录 OpenID 前需输入 server password
    ↓
前端密码输入框 + 按钮: "Start using OpenID"
    ↓
携带 password 参数发送到后端
    ↓
checkPassword() 验证通过 → 继续 OIDC 流程
```

---

## 7. 会话管理与登出

### 7.1 登出流程 (`signOut()`)
```
用户点击登出
    ↓
encryption.unloadAllKeys() 清除加密密钥
    ↓
asyncStorage 批量删除:
    ├─ user-token (会话token)
    ├─ encrypt-keys (加密密钥)
    ├─ lastBudget (最近预算)
    └─ readOnly (只读模式)
    ↓
Redux 状态重置 → 回到登录页
```

### 7.2 会话清理
- **自动清理**: 每次新会话创建时调用 `clearExpiredSessions()`
- **过期策略**:
  - 默认: 10 分钟
  - `openid-provider`: 使用 OIDC provider 的 expires_at
  - `never`: 永不过期
  - 自定义: 配置秒数

---

## 8. 完整流程时序图

```
前端 (React)                          服务端 (Express)                     OIDC Provider
    |                                     |                                      |
    |--- 1. subscribe-needs-bootstrap -->|                                      |
    |                                     |--- 查询 auth 表                     |
    |<-- 返回 bootstrapped=false ---------|                                      |
    |                                     |                                      |
    |--- 2. subscribe-bootstrap {password/openId} -->|                        |
    |                                     |--- 写入 auth 表                     |
    |<-- 成功 ----------------------------|                                      |
    |                                     |                                      |
    |--- 3. subscribe-sign-in {returnUrl, openid} -->|                        |
    |                                     |                                      |
    |                                     |--- 生成 state/code_verifier         |
    |                                     |--- 存储 pending 请求                 |
    |<-- 返回 redirectUrl ----------------|                                      |
    |                                     |                                      |
    |---------------------------------------------------> 4. 授权请求 (浏览器) |
    |                                     |                                      |<-- 用户登录/同意
    |                                     |                                      |--- 生成 code
    |<-- 5. 重定向 /openid/callback?code ------------------------------------------------|
    |                                     |                                      |
    |                                     |--- 6. 验证 state + 交换 token       |
    |                                     |<-- /token 端点 --------------------->|
    |                                     |<-- /userinfo 端点 ------------------>|
    |                                     |                                      |
    |                                     |--- 创建/更新用户                     |
    |                                     |--- 创建 session                     |
    |<-- 7. 重定向 /openid-cb?token ------|                                      |
    |                                     |                                      |
    |--- 8. subscribe-set-token {token} ->|                                      |
    |--- 9. dispatch(loggedIn())          |                                      |
    |--- 登录完成进入应用                  |                                      |
```

---

## 9. 关键文件索引

| 功能模块 | 文件路径 |
|---------|---------|
| Bootstrap 页面 | `packages/desktop-client/src/components/manager/subscribe/Bootstrap.tsx` |
| Login 页面 | `packages/desktop-client/src/components/manager/subscribe/Login.tsx` |
| OIDC 表单 | `packages/desktop-client/src/components/manager/subscribe/OpenIdForm.tsx` |
| 通用 Hook | `packages/desktop-client/src/components/manager/subscribe/common.tsx` |
| 错误页面 | `packages/desktop-client/src/components/manager/subscribe/Error.tsx` |
| 账户主路由 | `packages/sync-server/src/app-account.js` |
| OIDC 路由 | `packages/sync-server/src/app-openid.ts` |
| OIDC 核心逻辑 | `packages/sync-server/src/accounts/openid.ts` |
| 密码认证 | `packages/sync-server/src/accounts/password.ts` |
| 服务端认证应用 | `packages/loot-core/src/server/auth/app.ts` |

---

## 10. 总结: 设计特点

1. **状态驱动**: `useBootstrapped()` 自动管理页面流转，避免手动判断
2. **多层验证**: 前端表单验证 + 后端业务验证 + 安全边界验证
3. **安全优先**: PKCE、Rate Limit、Redirect 白名单、bcrypt 哈希
4. **多方式支持**: 密码、OIDC、Header 三种认证方式统一接口
5. **容错处理**: 网络失败、配置错误、过期状态均有明确错误分支
6. **首次用户**: OIDC 首个用户自动成为管理员，无缝迁移文件权限
