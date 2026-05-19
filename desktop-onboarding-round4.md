# 桌面客户端首次启动引导流程深度分析（第四轮）

> 基于实际代码逐行核对后的修正版本

---

## 1. get-remote-files 在鉴权失败（401）时的返回路径

### 1.1 完整调用链与错误处理

**函数**: `cloud-storage.ts:374-403`

```typescript
export async function listRemoteFiles(): Promise<RemoteFile[]> {
  const userToken = await asyncStorage.getItem('user-token');
  if (!userToken) {
    return null;  // 路径A: 无token直接返回
  }

  let res;
  try {
    res = await fetchJSON(getServer().SYNC_SERVER + '/list-user-files', {
      headers: {
        'X-ACTUAL-TOKEN': userToken,
      },
    });
  } catch (e) {
    logger.log('Unexpected error fetching file list from server', e);
    return null;  // 路径B: 任何异常都返回null
  }

  if (res.status === 'error') {
    logger.log('Error fetching file list from server', res);
    return null;  // 路径C: 业务错误返回null
  }

  return res.data.map(...);  // 路径D: 正常返回
}
```

### 1.2 fetchJSON 的 HTTP 错误处理

**函数**: `cloud-storage.ts:65-69`

```typescript
async function fetchJSON(...args: Parameters<typeof fetch>) {
  let res = await fetch(...args);
  res = await checkHTTPStatus(res);  // ⭐ 这里会抛出非200状态码的错误
  return res.json();
}
```

**checkHTTPStatus 逻辑**: `cloud-storage.ts:43-63`

```typescript
async function checkHTTPStatus(res) {
  if (res.status !== 200) {
    if (res.status === 403) {
      // 特殊处理403 token过期
      try {
        const text = await res.text();
        const data = JSON.parse(text)?.data;
        if (data?.reason === 'token-expired') {
          await asyncStorage.removeItem('user-token');
          throw new HTTPError(403, 'token-expired');
        }
      } catch (e) {
        if (e instanceof HTTPError) throw e;
      }
    }
    // ⭐ 所有其他非200状态码（包括401）都会走到这里
    return res.text().then(str => {
      throw new HTTPError(res.status, str);
    });
  } else {
    return res;
  }
}
```

### 1.3 401 鉴权失败的具体路径

```
有 token 场景
  │
  ▼
调用 fetchJSON()
  │
  ▼
fetch() 返回 Response { status: 401 }
  │
  ▼
checkHTTPStatus(res)
  ├─> res.status !== 200 ✓
  ├─> res.status === 403? ✗
  │
  └─> 走到通用错误分支
       │
       ▼
throw new HTTPError(401, responseBody)
  │
  ▼
被 listRemoteFiles 的 catch(e) 捕获
  │
  ▼
logger.log(...)
  │
  ▼
return null
```

### 1.4 特殊场景：403 token-expired

```
有 token 场景
  │
  ▼
服务器返回 403，body: { data: { reason: 'token-expired' } }
  │
  ▼
checkHTTPStatus 识别到 token-expired
  │
  ▼
asyncStorage.removeItem('user-token')  ⭐ token被清除!
  │
  ▼
throw new HTTPError(403, 'token-expired')
  │
  ▼
catch 捕获，return null
```

**关键副作用**: 403 token-expired 会主动清除本地存储的 user-token，下次调用会直接走"无token"路径。

---

## 2. 有 token 鉴权失败 vs 无 token 场景对比

### 2.1 并列对比表

| 对比项 | 无 token 场景 | 有 token 但 401 | 有 token 但 403 token-expired |
|--------|-------------|----------------|-----------------------------|
| **user-token 状态** | `null` / 空字符串 | 有效字符串 | 有效字符串 |
| **是否发送网络请求** | ❌ 否 | ✅ 是 | ✅ 是 |
| **HTTP 状态码** | 无 | 401 | 403 |
| **是否清除 token** | ❌ 否 | ❌ 否 | ✅ 是 |
| **返回值** | `null` | `null` | `null` |
| **日志输出** | 无 | `Unexpected error fetching...` | `Unexpected error fetching...` |
| **下次调用行为** | 直接返回 null | 继续发送请求（可能循环401） | 下次走无token路径 |
| **对 UI 的影响** | 不显示远程文件列表 | 不显示远程文件列表 | 不显示远程文件列表 |
| **对登录状态的影响** | 无 | 无（token仍在） | 可能触发登出（token被清） |

### 2.2 路径差异可视化

```
┌─────────────────────────────────────────────────────────────┐
│                     listRemoteFiles()                       │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
              ┌──────────────────────────┐
              │   userToken 存在吗？     │
              └──────────────────────────┘
                   │          │
                  否          是
                   │          │
                   ▼          ▼
              return null  发送请求
                              │
                        ┌─────┴─────┐
                        │  响应状态  │
                        └───────────┘
                         │    │    │
                       200   401  403
                         │    │    │
                         ▼    ▼    ▼
                    返回数据  null  null
                                      │
                                      ▼
                                清除 user-token
```

### 2.3 401 场景的潜在问题

**问题**: 401 鉴权失败但 token 仍保留在本地存储中，可能导致：
1. 每次调用 `listRemoteFiles` 都会发送一次失败的网络请求
2. 用户看到的现象是"远程文件列表不显示"，但不知道是鉴权失败
3. 没有自动登出机制，用户需要手动重新登录

**对比 403 token-expired**: 主动清除 token，至少避免了重复的失败请求。

---

## 3. 预算目录 ID 生成规则（逐字符详细版）

### 3.1 源码再确认

**位置**: `budget-name.ts:41-61`

```typescript
export async function idFromBudgetName(name: string): Promise<string> {
  // 核心公式: 规范化(名称) + "-" + UUID前7位
  let id = name.replace(/( |[^A-Za-z0-9])/g, '-') + '-' + uuidv4().slice(0, 7);
  
  // 目录存在性检查（极小概率）
  let index = 0;
  let budgetDir = fs.getBudgetDir(id);
  while (await fs.exists(budgetDir)) {
    index++;
    budgetDir = fs.getBudgetDir(id + index.toString());
  }
  if (index > 0) {
    id = id + index.toString();
  }
  
  return id;
}
```

### 3.2 正则规则详解

正则: `/( |[^A-Za-z0-9])/g`

**逻辑**: 全局匹配以下任意一种情况，替换为 `-`
1. 空格字符 ` `（第一个分支）
2. 任何不是 A-Z、a-z、0-9 的字符（第二个分支 `[^A-Za-z0-9]`）

**字符分类表**:

| 字符类型 | 示例 | 是否匹配 | 替换为 |
|---------|------|----------|--------|
| 大写字母 | A-Z | ❌ 否 | 保留 |
| 小写字母 | a-z | ❌ 否 | 保留 |
| 数字 | 0-9 | ❌ 否 | 保留 |
| 空格 | ` ` | ✅ 是 | `-` |
| 中文 | 家预算 | ✅ 是 | `-` |
| 日文 | 予算 | ✅ 是 | `-` |
| 韩文 | 예산 | ✅ 是 | `-` |
| 常用标点 | `!@#$%^&*()_+` | ✅ 是 | `-` |
| 下划线 | `_` | ✅ 是 | `-` |
| 连字符 | `-` | ❌ 否 | 保留 |
|  emoji | 💰📊 | ✅ 是 | `-` |
| 控制字符 | `\t\n\r` | ✅ 是 | `-` |

### 3.3 逐字符替换示例

#### 示例1: `家庭预算 2024`

```
输入: "家庭预算 2024"
字符: 家  庭  预  算    2  0  2  4
ASCII: 非  非  非  非  空 数 数 数 数
匹配:  ✅  ✅  ✅  ✅  ✅ ❌ ❌ ❌ ❌
替换:  -   -   -   -   -  2  0  2  4
└──────────────────────────────┘
规范化结果: "-------2024"
                  │
                  ▼
加上UUID后缀: "-------2024-a1b2c3d"
                  │
                  ▼
最终ID: "-------2024-a1b2c3d"
```

> **说明**: 5个非ASCII字符（家、庭、预、算）+ 1个空格 = 6个 `-`，然后是数字 2024，总共 `-------2024`（7个连字符）

#### 示例2: `Bob's Budget_2024!`

```
输入: "Bob's Budget_2024!"
字符: B  o  b  '  s    B  u  d  g  e  t  _  2  0  2  4  !
匹配: ❌ ❌ ❌ ✅ ❌ ✅ ❌ ❌ ❌ ❌ ❌ ✅ ❌ ❌ ❌ ❌ ✅
替换: B  o  b  -  s  -  B  u  d  g  e  t  -  2  0  2  4  -
└──────────────────────────────────────────────────────────┘
规范化结果: "Bob-s-Budget-2024-"
                  │
                  ▼
加上UUID后缀: "Bob-s-Budget-2024--x9y8z7w"
                  │
                  ▼
最终ID: "Bob-s-Budget-2024--x9y8z7w"
```

#### 示例3: `💰 My Finances 📊`

```
输入: "💰 My Finances 📊"
字符: 💰    M  y    F  i  n  a  n  c  e  s    📊
匹配: ✅  ✅ ❌ ❌ ✅ ❌ ❌ ❌ ❌ ❌ ❌ ❌ ✅ ✅
替换:  -  -  M  y  -  F  i  n  a  n  c  e  s  -  -
└─────────────────────────────────────────────────┘
规范化结果: "--My-Finances--"
                  │
                  ▼
加上UUID后缀: "--My-Finances---k3m5n7p"
                  │
                  ▼
最终ID: "--My-Finances---k3m5n7p"
```

#### 示例4: `预算！@#$Test123`

```
输入: "预算！@#$Test123"
字符: 预  算  ！  @  #  $   T  e  s  t  1  2  3
匹配: ✅  ✅  ✅ ✅ ✅ ✅  ❌ ❌ ❌ ❌ ❌ ❌ ❌
替换:  -   -   -  -  -  -   T  e  s  t  1  2  3
└─────────────────────────────────────────────────┘
规范化结果: "------Test123"
                  │
                  ▼
加上UUID后缀: "------Test123-q1w2e3r"
                  │
                  ▼
最终ID: "------Test123-q1w2e3r"
```

### 3.4 ID 安全性校验

**位置**: `platform/server/fs/shared.ts:26-30`

```typescript
if (id.match(/[^A-Za-z0-9\-_]/)) {
  throw new Error(
    `Invalid budget id "${id}". Check the id of your budget...`,
  );
}
```

**允许的字符**: `A-Z`, `a-z`, `0-9`, `-`, `_`

> 💡 由于生成规则只产生 `A-Za-z0-9-`，正常生成的 ID 永远不会触发此错误。此校验主要防止用户手动创建目录或通过 API 传入恶意 ID。

---

## 4. 目录重命名 vs 移出根目录：可发现性与可加载性差异

### 4.1 核心概念定义

| 概念 | 定义 | 涉及代码 |
|------|------|---------|
| **可发现性** | `getBudgets()` 能否扫描到并返回该预算 | `budgetfiles/app.ts:105-140` |
| **可加载性** | `loadBudget(id)` 能否成功打开该预算 | `budgetfiles/app.ts:508-640` |

### 4.2 可发现性判定逻辑

**函数**: `getBudgets()` (`budgetfiles/app.ts:105-140`)

```typescript
async function getBudgets() {
  const paths = await fs.listDir(fs.getDocumentDir());  // 步骤1: 列出扫描根目录下所有条目
  const budgets = await Promise.all(
    paths.map(async name => {
      const prefsPath = fs.join(fs.getDocumentDir(), name, 'metadata.json');
      if (await fs.exists(prefsPath)) {  // 步骤2: 检查是否有 metadata.json
        return {
          id: name,  // 步骤3: 目录名即ID
          name: prefs.budgetName,
          // ...
        };
      }
      return null;
    }),
  );
  return budgets.filter(Boolean) as Budget[];
}
```

**可发现性的两个必要条件**:
1. 目录必须在 `getDocumentDir()` 的直接子级（不能嵌套）
2. 目录内必须存在 `metadata.json` 文件

### 4.3 可加载性判定逻辑

**函数**: `_loadBudget()` (`budgetfiles/app.ts:508-640`)

```typescript
async function _loadBudget(id: Budget['id']) {
  let dir: string;
  try {
    dir = fs.getBudgetDir(id);  // 检查1: ID格式校验
  } catch (e) {
    return { error: 'budget-not-found' };
  }

  if (!(await fs.exists(dir))) {  // 检查2: 目录存在性
    return { error: 'budget-not-found' };
  }

  try {
    await prefs.loadPrefs(id);     // 检查3: metadata.json 可读取
    await db.openDatabase(id);     // 检查4: db.sqlite 可打开
  } catch (e) {
    return { error: 'opening-budget' };
  }
  // ... 后续加载步骤
}
```

**可加载性的必要条件**:
1. ID 通过格式校验（只含 `A-Za-z0-9-_`）
2. 目录 `getBudgetDir(id)` 存在
3. `metadata.json` 存在且可解析
4. `db.sqlite` 存在且可打开

### 4.4 操作对比分析

#### 场景1: 在扫描根目录内重命名

**操作**: `Actual/My-Finances-a1b2c3d` → `Actual/My-Renamed-Budget`

| 方面 | 状态 | 说明 |
|------|------|------|
| **可发现性** | ✅ 保留 | 仍在 DocumentDir 下，且有 metadata.json |
| **可加载性** | ✅ 保留 | 新目录名作为 ID，通过格式校验，目录存在 |
| **ID 变化** | ✅ 变化 | 新 ID = `My-Renamed-Budget` |
| **metadata.id** | 加载时被覆盖 | `loadPrefs(id)` 强制设置 `prefs.id = id` |
| **lastBudget 记录** | ⚠️ 失效 | 存储的是旧 ID，下次打开会报 `budget-not-found` |
| **数据完整性** | ✅ 完整 | 所有文件都在，只是目录名变了 |
| **同步关联** | ✅ 保留 | cloudFileId/groupId 仍在 metadata.json 中 |

**代码验证** (`prefs.ts:41-44`):
```typescript
export async function loadPrefs(id?: string): Promise<MetadataPrefs> {
  prefs = JSON.parse(await fs.readFile(fullpath));
  prefs.id = id;  // ⭐ 强制覆盖，不管文件里存的是什么
  return prefs;
}
```

#### 场景2: 移出扫描根目录

**操作**: `Actual/My-Finances-a1b2c3d` → `Desktop/My-Finances-a1b2c3d`

| 方面 | 状态 | 说明 |
|------|------|------|
| **可发现性** | ❌ 丢失 | 不在 DocumentDir 下，`listDir` 扫描不到 |
| **可加载性** | ❌ 丢失 | `fs.exists(dir)` 返回 false |
| **ID 变化** | - | 无关（因为找不到） |
| **数据完整性** | ✅ 完整 | 文件只是被移动，没有损坏 |
| **同步关联** | ✅ 保留 | metadata.json 仍在目录内 |
| **移回后的可发现性** | ✅ 恢复 | 移回 DocumentDir 后重新出现 |
| **移回后的可加载性** | ✅ 恢复 | 如果目录名没变，ID 不变 |

#### 场景3: 移出后重命名再移回

**操作**: `Actual/My-Finances-a1b2c3d` → 移出 → 重命名为 `OldBudget` → 移回 `Actual/OldBudget`

| 方面 | 状态 | 说明 |
|------|------|------|
| **可发现性** | ✅ 恢复 | 回到 DocumentDir 且有 metadata.json |
| **可加载性** | ✅ 恢复 | 新目录名 `OldBudget` 通过格式校验 |
| **ID** | ✅ 变化 | 新 ID = `OldBudget` |
| **同步关联** | ✅ 保留 | cloudFileId/groupId 仍在 metadata.json 中 |
| **lastBudget** | ❌ 失效 | 需要用户手动打开一次 |

#### 场景4: 根目录内嵌套（如 `Actual/Subdir/MyBudget`）

| 方面 | 状态 | 说明 |
|------|------|------|
| **可发现性** | ❌ 丢失 | `listDir` 只扫描直接子级 |
| **可加载性** | ❌ 丢失 | `getBudgetDir('Subdir/MyBudget')` 会触发路径遍历校验失败 |

> 💡 这是故意设计的安全机制，防止路径遍历攻击。预算必须是 DocumentDir 的直接子目录。

### 4.5 边界差异总结表

| 操作 | 可发现性 | 可加载性 | ID变化 | 数据完整 | 同步关联 | 可恢复 |
|------|---------|---------|--------|---------|---------|--------|
| 根目录内重命名 | ✅ | ✅ | ✅ 是 | ✅ | ✅ | - |
| 移出根目录 | ❌ | ❌ | - | ✅ | ✅ | ✅ 移回即可 |
| 移出后重命名再移回 | ✅ | ✅ | ✅ 是 | ✅ | ✅ | - |
| 根目录内嵌套 | ❌ | ❌ | - | ✅ | ✅ | ✅ 移到直接子级 |
| 删除 metadata.json | ❌ | ❌ | - | ⚠️ db还在 | ✅ | ✅ 恢复文件即可 |
| 删除 db.sqlite | ✅ | ❌ | - | ❌ | ✅ | ❌ |

### 4.6 关键设计洞察

1. **目录即身份**: ID 不存储在数据库或元数据中，而是由目录名动态确定
2. **松散耦合**: metadata.json 中的 id 字段只是缓存，加载时会被覆盖
3. **容错设计**: 用户可以自由重命名/移动目录，系统会自动适配
4. **安全优先**: 严格的路径校验防止目录遍历攻击
5. **无状态扫描**: getBudgets 不依赖任何索引或缓存，每次都是实时扫描文件系统

---

## 5. 本论修正总结

| 之前理解 | 实际代码行为 | 修正影响 |
|---------|-------------|---------|
| 401 鉴权失败会清除 token | 只有 403 token-expired 才清除 token，401 不清除 | 401 场景会重复发送失败请求 |
| 移出根目录数据会丢失 | 只是不可见，数据完整保留在原位置 | 用户可以通过移回恢复 |
| ID 生成正则保留下划线 | 下划线会被替换为 `-` | 生成的 ID 只包含字母、数字、`-` |
| getBudgets 扫描嵌套目录 | 只扫描直接子级，不递归 | 嵌套目录不可见 |

---

## 6. 关键代码位置速查

| 功能 | 文件 | 行号 |
|------|------|------|
| listRemoteFiles 错误处理 | `cloud-storage.ts` | L374-403 |
| checkHTTPStatus 状态码处理 | `cloud-storage.ts` | L43-63 |
| fetchJSON 封装 | `cloud-storage.ts` | L65-69 |
| HTTPError 类定义 | `errors.ts` | L30-39 |
| 预算 ID 生成 | `budget-name.ts` | L41-61 |
| ID 安全性校验 | `platform/server/fs/shared.ts` | L26-30 |
| getBudgets 扫描逻辑 | `budgetfiles/app.ts` | L105-140 |
| _loadBudget 加载检查 | `budgetfiles/app.ts` | L508-541 |
| loadPrefs 强制覆盖 ID | `prefs.ts` | L41-44 |
