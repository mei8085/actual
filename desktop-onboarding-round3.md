# 桌面客户端首次启动引导流程深度分析（第三轮）

> 基于实际代码核对后的修正版本

---

## 1. 本地模式下 disabled 与 offline 的设置顺序、语义差异及影响

### 1.1 关键代码位置核对

**设置时序 (按执行顺序)**：

| 步骤 | 位置 | 代码 | 效果 |
|------|------|------|------|
| 1 | `budgetfiles/app.ts:622-626` | `_loadBudget` 中 | `setSyncingMode('disabled')` |
| 2 | `budgetfiles/app.ts:637` | 触发事件 | `app.events.emit('load-budget', { id })` |
| 3 | `main.ts:314-316` | 事件监听器 | `setSyncingMode('offline')` |

**监听器注册时机** (`main.ts:309-317`):
```typescript
} else {
  // This turns off all server URLs. In this mode we don't want any
  // access to the server, we are doing things locally
  setServer(null);

  app.events.on('load-budget', () => {
    setSyncingMode('offline');  // ⭐ 覆盖之前的 disabled
  });
}
```

> ⚠️ **重要修正**: 之前认为 `disabled` 是最终状态，实际最终状态是 `offline`！

### 1.2 四种同步模式的语义详解

**位置**: `sync/index.ts:41-78`

| 模式 | 语义 | `checkSyncingMode('enabled')` | `checkSyncingMode('disabled')` | 对同步行为的影响 |
|------|------|-------------------------------|--------------------------------|-----------------|
| `enabled` | 正常同步 | ✅ true | ❌ false | 消息会记录并发送到服务器，定期执行 `fullSync()` |
| `offline` | 离线模式 | ✅ true (重要!) | ❌ false | 消息会记录到 `messages_crdt` 表，但 `scheduleFullSync()` 被跳过 (L555)，不会主动发送 |
| `disabled` | 完全禁用 | ❌ false | ✅ true | 消息不记录到 `messages_crdt` 表，不生成同步消息 |
| `import` | 导入模式 | ❌ false | ✅ true | 快速导入，跳过同步逻辑 |

**关键判断条件**:
```typescript
// L65-77: checkSyncingMode 的实现
export function checkSyncingMode(mode: SyncingMode): boolean {
  switch (mode) {
    case 'enabled':
      return SYNCING_MODE === 'enabled' || SYNCING_MODE === 'offline';  // ⭐ offline 也算 enabled!
    case 'disabled':
      return SYNCING_MODE === 'disabled' || SYNCING_MODE === 'import';
    // ...
  }
}
```

**scheduleFullSync 的跳过逻辑** (`sync/index.ts:555`):
```typescript
if (checkSyncingMode('enabled') && !checkSyncingMode('offline')) {
  // 只有真正的 enabled 模式才会调度同步
  // offline 模式会跳过这里
}
```

### 1.3 本地模式下的实际行为

**状态路径**:
```
加载预算
  │
  ▼
_loadBudget: setSyncingMode('disabled')   // 临时状态
  │
  ▼
触发 'load-budget' 事件
  │
  ▼
事件监听器: setSyncingMode('offline')     // 最终稳定状态
  │
  ▼
用户操作数据
  ├─> 生成同步消息 ✓ (因为 checkSyncingMode('enabled') === true)
  ├─> 写入 messages_crdt 表 ✓
  └─> 调度 fullSync() ✗ (因为 checkSyncingMode('offline') === true)
```

**实际效果**:
- 本地模式下，CRDT 消息**仍然被记录**（为未来可能的同步做准备）
- 但**不会主动发送**到服务器
- 如果用户后续配置了服务器，可以执行全量同步，所有历史消息都会被上传

---

## 2. get-remote-files 返回 null 的真实条件

### 2.1 调用链

```
UI -> get-remote-files
        │
        ▼
getRemoteFiles() (budgetfiles/app.ts:142-144)
        │
        ▼
cloudStorage.listRemoteFiles() (cloud-storage.ts:374-403)
```

### 2.2 返回 null 的完整条件分析

**位置**: `cloud-storage.ts:374-395`

```typescript
export async function listRemoteFiles(): Promise<RemoteFile[]> {
  const userToken = await asyncStorage.getItem('user-token');
  if (!userToken) {
    return null;  // 条件1: 没有用户token
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
    return null;  // 条件2: 请求失败（网络错误、CORS、getServer()为null等）
  }

  if (res.status === 'error') {
    logger.log('Error fetching file list from server', res);
    return null;  // 条件3: 服务器返回错误
  }

  return res.data.map(...);  // 正常返回数组
}
```

### 2.3 条件2的详细拆解

`fetchJSON(getServer().SYNC_SERVER + ...)` 可能失败的场景：

| 场景 | 原因 | 是否返回 null |
|------|------|--------------|
| `serverURL === null` | `getServer()` 返回 `null`，调用 `null.SYNC_SERVER` 抛 TypeError | ✅ 是 |
| `serverURL` 无效 | `new URL()` 失败，`getServer()` 返回 `null` | ✅ 是 |
| 网络不可达 | 嵌入式服务器未启动、远程服务器离线 | ✅ 是 |
| 服务器返回 5xx | 服务端错误 | ✅ 是 |
| 服务器返回 401 | token 无效（但会被条件1提前挡住） | ✅ 是 |

### 2.4 与 user-token、server-url 的真实关系

| server-url | user-token | 返回值 | 说明 |
|------------|------------|--------|------|
| `null` | 任意 | `null` | `getServer()` 为 null，抛异常被 catch |
| 有效 URL | `null` / 空 | `null` | 条件1直接返回 |
| 有效 URL | 有效 token | `RemoteFile[]` / `null` | 取决于网络和服务器状态 |
| 无效 URL | 有效 token | `null` | `getServer()` 返回 null，抛异常 |

> ⚠️ **重要修正**: 之前认为 `server-url === null` 时直接返回 null，实际是通过异常路径间接返回 null。代码没有显式检查 `getServer()` 是否为 null！

---

## 3. 预算目录 ID 生成规则（按实际代码重写）

### 3.1 实际代码

**位置**: `budget-name.ts:41-61`

```typescript
export async function idFromBudgetName(name: string): Promise<string> {
  // 步骤1: 正则替换 + UUID后缀
  let id = name.replace(/( |[^A-Za-z0-9])/g, '-') + '-' + uuidv4().slice(0, 7);

  // 步骤2: 确保目录唯一（极小概率冲突）
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

含义：匹配 **空格** 或 **任何非字母数字字符**，全部替换为 `-`

**允许的字符**（不被替换）:
- 大写字母 A-Z
- 小写字母 a-z
- 数字 0-9

**会被替换的字符**（全部变 `-`）:
- 空格 ` ` → `-`
- 中文字符 → `-`
- 标点符号 `!@#$%^&*()_+` 等 → `-`
- 其他 Unicode 字符 → `-`

### 3.3 正确示例

| 输入名称 | 生成过程 | 最终 ID |
|---------|----------|---------|
| `My Finances` | `My-Finances` + `-a1b2c3d` | `My-Finances-a1b2c3d` |
| `家庭预算 2024` | `------2024` + `-x9y8z7w` | `-------2024-x9y8z7w` |
| `Bob's Budget` | `Bob-s-Budget` + `-k3m5n7p` | `Bob-s-Budget-k3m5n7p` |
| `预算！@#$` | `--------` + `-q1w2e3r` | `--------q1w2e3r` |
| `Budget123` | `Budget123` + `-t5y6u7i` | `Budget123-t5y6u7i` |

> ⚠️ **重要修正**: 之前的示例"家庭预算 2024 → 2024-x9y8z7w"是错误的！中文字符每个都会变成 `-`，所以"家庭预算"变成 `------`（6个汉字 → 6个连字符）。

### 3.4 ID 的安全性校验

**位置**: `platform/server/fs/shared.ts:26-30`

```typescript
if (id.match(/[^A-Za-z0-9\-_]/)) {
  throw new Error(
    `Invalid budget id "${id}". Check the id of your budget...`,
  );
}
```

校验规则：只允许字母、数字、`-`、`_`

> 💡 由于生成规则保证了 ID 只包含这些字符，正常流程下不会触发此错误。此校验主要防止用户手动创建恶意目录名。

---

## 4. 预算目录重命名与移出扫描根目录的边界差异

### 4.1 扫描机制

**位置**: `budgetfiles/app.ts:105-140`

```typescript
async function getBudgets() {
  const paths = await fs.listDir(fs.getDocumentDir());  // 扫描整个文档目录
  const budgets = await Promise.all(
    paths.map(async name => {
      const prefsPath = fs.join(fs.getDocumentDir(), name, 'metadata.json');
      if (await fs.exists(prefsPath)) {
        // 有 metadata.json 才认为是有效预算目录
        return {
          id: name,  // ⭐ 目录名即 ID！
          name: prefs.budgetName,
          // ... 其他字段
        };
      }
      return null;
    }),
  );
  return budgets.filter(Boolean) as Budget[];
}
```

**扫描根目录**: `fs.getDocumentDir()`
- 桌面端: `<用户文档>/Actual/`
- 由 `ACTUAL_DOCUMENT_DIR` 环境变量控制

### 4.2 预算目录重命名（在扫描根目录内）

**场景**: 用户在文件管理器中把 `My-Finances-a1b2c3d` 重命名为 `My-Old-Budget`

| 方面 | 行为 | 说明 |
|------|------|------|
| **ID 变化** | 变为新目录名 | `getBudgets` 返回的 `id` 是新目录名 |
| **metadata.json 中的 id** | 加载时被覆盖 | `loadPrefs(id)` 会强制设置 `prefs.id = id` |
| **UI 显示** | 显示 `budgetName` 字段 | 不受目录名影响，显示用户友好名称 |
| **数据完整性** | 完整保留 | SQLite 数据库和所有文件都在 |
| **同步状态** | 保留 cloudFileId/groupId | metadata.json 中的同步字段不受影响 |
| **lastBudget 记录** | 失效 | 需要用户重新打开一次 |

**代码验证** (`prefs.ts:41-44`):
```typescript
export async function loadPrefs(id?: string): Promise<MetadataPrefs> {
  prefs = JSON.parse(await fs.readFile(fullpath));
  prefs.id = id;  // ⭐ 强制覆盖，不管文件里存的是什么
  return prefs;
}
```

### 4.3 预算目录移出扫描根目录

**场景**: 用户把 `My-Finances-a1b2c3d` 文件夹移动到桌面或其他位置

| 方面 | 行为 | 说明 |
|------|------|------|
| **UI 可见性** | 消失 | `getBudgets` 扫描不到 |
| **本地数据** | 完整保留 | 文件只是被移动，没有删除 |
| **云端数据** | 不受影响 | 已同步的数据仍在服务器上 |
| **移回后** | 重新出现 | 移回 `Actual/` 目录后会被重新扫描到 |
| **移回时 ID** | 取决于目录名 | 如果目录名没变，ID 不变 |
| **lastBudget 记录** | 失效 | 加载时会报 `budget-not-found` |

### 4.4 两种操作的边界对比

| 对比项 | 重命名（根目录内） | 移出根目录 |
|--------|-------------------|-----------|
| 是否在扫描列表中 | ✅ 是 | ❌ 否 |
| ID 是否变化 | ✅ 变化（新目录名） | ❌ 无（移出前不变） |
| 数据完整性 | ✅ 完整 | ✅ 完整（只是移动） |
| 能否正常打开 | ✅ 能 | ❌ 不能（路径不存在） |
| 同步关联 | ✅ 保留 | ✅ 保留（metadata.json 仍在） |
| 恢复难度 | 无需恢复 | 移回目录即可恢复 |
| 风险 | 低 | 中（用户可能忘记移回） |

### 4.5 特殊边界情况

**情况1: 移出后重命名再移回**
```
My-Finances-a1b2c3d → 移出 → 重命名为 Old-Budget → 移回
  │
  ▼
getBudgets() 返回 id = 'Old-Budget'
```
- ID 变了，但数据完整
- 如果之前已同步，cloudFileId/groupId 仍然有效
- 可以正常上传/同步

**情况2: 两个目录包含相同的 cloudFileId**
- 用户复制预算目录产生两个文件夹
- 两个 metadata.json 有相同的 cloudFileId
- 同步时会产生冲突（后上传的会覆盖先上传的）
- 但 `duplicateBudget` 函数会主动清除这些字段（L339-347）

**duplicateBudget 中的安全处理**:
```typescript
[
  'cloudFileId',
  'groupId',
  'lastUploaded',
  'encryptKeyId',
  'lastSyncedTimestamp',
].forEach(item => {
  if (metadata[item]) delete metadata[item];  // ⭐ 复制时清除同步关联
});
```

---

## 5. 修正总结

| 之前理解 | 实际代码行为 | 修正影响 |
|---------|-------------|---------|
| 本地模式下同步模式为 `disabled` | 先 `disabled` 后 `offline`，最终是 `offline` | CRDT 消息仍被记录，可后续同步 |
| `get-remote-files` 显式检查 `serverURL` | 通过 `getServer().SYNC_SERVER` 抛异常间接返回 `null` | 没有显式防御，依赖异常处理 |
| 中文字符在 ID 中保留 | 所有非 ASCII 字符都变成 `-` | 纯中文名称会变成全连字符 ID |
| 重命名目录会破坏功能 | 系统自动适配，目录名即 ID | 用户可以自由重命名/移动预算目录 |

---

## 6. 关键代码位置速查

| 功能 | 文件 | 行号 |
|------|------|------|
| 同步模式设置时序 | `budgetfiles/app.ts` | L618-637 |
| offline 模式事件监听器 | `main.ts` | L309-317 |
| 四种同步模式语义 | `sync/index.ts` | L41-78 |
| scheduleFullSync 跳过逻辑 | `sync/index.ts` | L549-567 |
| listRemoteFiles 返回 null 条件 | `cloud-storage.ts` | L374-395 |
| 预算 ID 生成规则 | `budget-name.ts` | L41-61 |
| ID 安全性校验 | `platform/server/fs/shared.ts` | L26-30 |
| 预算目录扫描逻辑 | `budgetfiles/app.ts` | L105-140 |
| loadPrefs 强制覆盖 ID | `prefs.ts` | L41-44 |
| duplicateBudget 清除同步字段 | `budgetfiles/app.ts` | L339-347 |
