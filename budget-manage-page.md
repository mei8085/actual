# 预算管理页面状态机分析（代码证实版）

## 重要说明

本文档所有结论均基于代码实证。每条结论前标注：
- ✅ **代码证实**：有明确代码为依据
- 🤔 **合理推断**：代码未直接说明，基于上下文的逻辑推断
- ❌ **之前错误**：指出之前版本分析中的错误

---

## 一、loadAllFiles 获取顺序（✅ 代码证实）

### ✅ 代码证实：串行执行，不是并行

`budgetfilesSlice.ts:37-47`：

```typescript
export const loadAllFiles = createAppAsyncThunk(
  `${sliceName}/loadAllFiles`,
  async (_, { dispatch, getState }) => {
    // 注意：这里是两个 await 串行执行，不是 Promise.all
    const budgets = await send('get-budgets');      // 先执行
    const files = await send('get-remote-files');   // 前一个完成后才执行

    dispatch(setAllFiles({ budgets, remoteFiles: files }));

    return getState().budgetfiles.allFiles;
  }
);
```

---

## 二、closeBudget 的异常处理边界（✅ 代码证实）

### ✅ 代码证实：前端 closeBudget 有前置条件检查

`budgetfilesSlice.ts:98-113`：

```typescript
export const closeBudget = createAppAsyncThunk(
  `${sliceName}/closeBudget`,
  async (_, { dispatch, getState, extra: { queryClient } }) => {
    const prefs = getState().prefs.local;
    // ⚠️ 只有 prefs.id 存在时才执行关闭逻辑
    if (prefs && prefs.id) {
      dispatch(resetApp());
      queryClient.clear();
      dispatch(setAppState({ loadingText: t('Closing...') }));
      await send('close-budget');
      dispatch(setAppState({ loadingText: null }));
      if (localStorage.getItem('SharedArrayBufferOverride')) {
        window.location.reload();
      }
    }
  }
);
```

### ✅ 代码证实：后端 closeBudget 只有一处 try-catch

`budgetfiles/app.ts:257-280`：

```typescript
async function closeBudget() {
  captureBreadcrumb({ message: 'Closing budget' });

  // 以下步骤全部没有 try-catch，任何一步失败都会导致整个函数抛出异常
  await sheet.waitOnSpreadsheet();     // ❌ 无保护
  sheet.unloadSpreadsheet();           // ❌ 无保护
  clearFullSyncTimeout();              // ❌ 无保护
  await mainApp.stopServices();        // ❌ 无保护
  db.closeDatabase();                  // ❌ 无保护

  // ✅ 只有这一处有 try-catch
  try {
    await asyncStorage.setItem('lastBudget', '');
  } catch {
    // 注释说明：加载预算失败后关闭时可能失败，要保持弹性
  }

  prefs.unloadPrefs();                 // ❌ 无保护
  stopBackupService();                 // ❌ 无保护
  return 'ok';
}
```

**prefs.unloadPrefs 实现**（`prefs.ts:81-83` 代码证实）：
```typescript
export function unloadPrefs(): void {
  prefs = null;  // 直接将内存变量设为 null
}
```

---

## 三、detached / unknown / broken 的真实触发条件与界面呈现

### 3.1 unknown 状态（✅ 代码证实）

**触发条件**：`remoteFiles == null`

`budgetfilesSlice.ts:524-538`：
```typescript
if (cloudFileId && groupId) {
  // 注释明确说明：获取服务器文件失败时，不吓用户，显示 unknown
  if (remoteFiles == null) {
    return {
      ...localFile,
      cloudFileId,
      groupId,
      deleted: false,
      state: 'unknown',
      hasKey: true,
      owner: '',
    };
  }
  // ...
}
```

**remoteFiles 为 null 的三种情况**（`cloud-storage.ts:374-403` 代码证实）：
1. 没有 user token（用户未登录）
2. 网络请求失败（fetch 抛出异常）
3. 服务器返回 `res.status === 'error'`

**界面呈现**（`BudgetFileSelection.tsx:188-193` 代码证实）：
- 图标：`SvgCloudUnknown`
- 文字："Network unavailable"
- 颜色：`theme.buttonNormalDisabledText`

### 3.2 detached 状态（✅ 代码证实）

**触发条件**：云端文件存在但 `remote.groupId !== localFile.groupId`

`budgetfilesSlice.ts:546-571`：
```typescript
const remote = remoteFiles.find(f => localFile.cloudFileId === f.fileId);
if (remote) {
  reconciled.add(remote.fileId);
  if (remote.groupId === localFile.groupId) {
    state: 'synced'
  } else {
    // groupId 不匹配 → detached
    state: 'detached'
  }
}
```

**界面呈现**（`BudgetFileSelection.tsx:187-214` 代码证实）：
```typescript
switch (file.state) {
  case 'unknown': /* ... */
  case 'remote': /* ... */
  case 'local': /* ... */
  case 'broken': /* ... */
  // ⚠️ 注意：没有 case 'detached'！
  default:
    // detached 走到这里
    Icon = SvgCloudCheck;       // ✅ 代码证实：显示同步成功图标
    status = t('Syncing');      // ✅ 代码证实：显示"同步中"
    ownerName = getOwnerDisplayName();
    break;
}
```

**⚠️ 代码证实的 UI 缺陷**：detached 状态被错误地显示为 "Syncing" + 云勾选图标，用户完全无法感知同步已断裂。

### 3.3 broken 状态（✅ 代码证实）

**触发条件**：本地有 `cloudFileId` 但在 `remoteFiles` 中找不到匹配项

`budgetfilesSlice.ts:573-582`：
```typescript
} else {
  // 本地有 cloudFileId，但 remoteFiles 中找不到
  return {
    ...localFile,
    cloudFileId,
    groupId,
    deleted: false,
    state: 'broken',
    hasKey: true,
    owner: '',
  };
}
```

**界面呈现**：
1. 图标和文字（`BudgetFileSelection.tsx:204-208` 代码证实）：
   - 图标：`SvgFileDouble`（本地文件图标）
   - 文字："Local"
   - Owner："Unknown"

2. 排序位置（`budgetfilesSlice.ts:613-615` 代码证实）：
   ```typescript
   // 注释：把 broken（未授权）文件列在底部
   return sorted
     .filter(f => f.state !== 'broken')
     .concat(sorted.filter(f => f.state === 'broken'));
   ```

✅ **代码证实**：broken 文件不是被隐藏，而是被移到列表末尾。

---

## 四、切换活跃预算的完整时序（✅ 代码证实，逐行追踪）

### 4.1 关键基础概念（✅ 代码证实）

在分析时序前，先明确几个核心操作的真实行为：

| 操作 | 真实行为 | 代码位置 |
|------|---------|----------|
| `prefs.loadPrefs(id)` | 从文件系统读取 metadata.json → 写入内存变量 `prefs` | `prefs.ts:24-45` |
| `prefs.unloadPrefs()` | 直接将内存变量 `prefs` 设为 `null` | `prefs.ts:81-83` |
| `prefs.getPrefs()` | 返回内存变量 `prefs` 的当前值 | `prefs.ts:85-87` |
| `db.openDatabase(id)` | 打开指定预算的 SQLite 数据库连接 | `db/index.ts:70-80` |
| `db.closeDatabase()` | 关闭当前 SQLite 数据库连接 | `db/index.ts:81-83` |
| `importBuffer()` | 解压 zip → 写入 db.sqlite 和 metadata.json 到**文件系统**，**不加载到内存** | `cloud-storage.ts:193-249` |

---

### 4.2 分支3时序追踪：当前有预算 → 下载并切换到云端预算

这是最复杂的场景，让我们逐行追踪 prefs 内存状态和数据库状态的变化：

```
初始状态：
  ✅ prefs 内存 = 旧预算的配置
  ✅ 数据库连接 = 旧预算已打开

┌─────────────────────────────────────────────────────────────────┐
│  前端 closeAndDownloadBudget thunk                              │
│  (budgetfilesSlice.ts:289-295)                                   │
└─────────────────────────────┬───────────────────────────────────┘
                              │
┌─────────────────────────────▼───────────────────────────────────┐
│  第1次调用 closeBudget()                                         │
└─────────────────────────────┬───────────────────────────────────┘
                              │
┌─────────────────────────────▼───────────────────────────────────┐
│  前端 closeBudget thunk (budgetfilesSlice.ts:98-113)            │
│  ├─ ✅ prefs.id 存在，执行关闭                                   │
│  ├─ dispatch(resetApp()) → 重置 Redux 状态                       │
│  ├─ queryClient.clear() → 清除 React Query 缓存                  │
│  └─ send('close-budget') → 发送到后端                            │
└─────────────────────────────┬───────────────────────────────────┘
                              │
┌─────────────────────────────▼───────────────────────────────────┐
│  后端 closeBudget handler (budgetfiles/app.ts:257-280)          │
│  ├─ await sheet.waitOnSpreadsheet()                              │
│  ├─ sheet.unloadSpreadsheet()                                    │
│  ├─ clearFullSyncTimeout()                                       │
│  ├─ await mainApp.stopServices()                                 │
│  ├─ db.closeDatabase() → ✅ 关闭旧数据库                          │
│  ├─ try { asyncStorage.setItem('lastBudget', '') }               │
│  ├─ prefs.unloadPrefs() → ✅ prefs 内存 = null                    │
│  └─ stopBackupService()                                          │
└─────────────────────────────┬───────────────────────────────────┘
                              │
                              ├─ 此时状态：
                              │   ✅ prefs 内存 = null
                              │   ✅ 数据库连接 = 已关闭
                              │
┌─────────────────────────────▼───────────────────────────────────┐
│  调用 downloadBudget({ cloudFileId, replace: true })            │
└─────────────────────────────┬───────────────────────────────────┘
                              │
┌─────────────────────────────▼───────────────────────────────────┐
│  前端 downloadBudget thunk (budgetfilesSlice.ts:302-379)        │
│  ├─ setAppState({ loadingText: 'Downloading...' })              │
│  └─ send('download-budget', { cloudFileId })                    │
└─────────────────────────────┬───────────────────────────────────┘
                              │
┌─────────────────────────────▼───────────────────────────────────┐
│  后端 downloadBudget handler (budgetfiles/app.ts:182-216)        │
│  ├─ try {                                                        │
│  │    result = await cloudStorage.download(cloudFileId)          │
│  │    ├─ 并行获取 fileInfo 和 fileBuffer                         │
│  │    ├─ 解密（如有 encryptMeta）                                │
│  │    └─ importBuffer(fileData, buffer)                          │
│  │       ├─ 解压 zip                                             │
│  │       ├─ 更新 metadata（设置 cloudFileId, groupId 等）        │
│  │       ├─ ✅ 写入 db.sqlite 和 metadata.json 到文件系统         │
│  │       └─ ✅ return { id: meta.id } → ❌ 只是写文件！不加载内存 │
│  └─ } catch (e) { 错误处理 }                                     │
└─────────────────────────────┬───────────────────────────────────┘
                              │
                              ├─ 此时状态：
                              │   ✅ prefs 内存 = null（importBuffer 没改内存）
                              │   ✅ 数据库连接 = 已关闭
                              │   ✅ 文件系统 = 新预算文件已写入
                              │
┌─────────────────────────────▼───────────────────────────────────┐
│  const id = result.id                                            │
└─────────────────────────────┬───────────────────────────────────┘
                              │
┌─────────────────────────────▼───────────────────────────────────┐
│  第2次调用 closeBudget()                                        │
│  (budgetfiles/app.ts:208)                                        │
└─────────────────────────────┬───────────────────────────────────┘
                              │
┌─────────────────────────────▼───────────────────────────────────┐
│  后端 closeBudget handler                                       │
│  ├─ await sheet.waitOnSpreadsheet() → 已经关了                   │
│  ├─ sheet.unloadSpreadsheet() → 已经关了                        │
│  ├─ clearFullSyncTimeout() → 已经清了                           │
│  ├─ await mainApp.stopServices() → 已经停了                      │
│  ├─ db.closeDatabase() → ✅ 再次关闭（已关则无操作）              │
│  ├─ try { asyncStorage.setItem('lastBudget', '') }               │
│  ├─ prefs.unloadPrefs() → ✅ prefs 内存 = null（已经是 null）    │
│  └─ stopBackupService() → 已经停了                               │
└─────────────────────────────┬───────────────────────────────────┘
                              │
                              ├─ 此时状态：
                              │   ✅ prefs 内存 = null
                              │   ✅ 数据库连接 = 已关闭
                              │
┌─────────────────────────────▼───────────────────────────────────┐
│  第1次调用 loadBudget({ id })                                    │
│  (budgetfiles/app.ts:209)                                        │
└─────────────────────────────┬───────────────────────────────────┘
                              │
┌─────────────────────────────▼───────────────────────────────────┐
│  后端 loadBudget handler (budgetfiles/app.ts:226-242)            │
│  ├─ currentPrefs = prefs.getPrefs() → ✅ 返回 null                │
│  ├─ currentPrefs 为假，跳过 id 检查和 closeBudget                │
│  └─ await _loadBudget(id)                                        │
│     ├─ await prefs.loadPrefs(id) → ✅ 从文件读取，prefs 内存 = 新预算配置 │
│     ├─ await db.openDatabase(id) → ✅ 打开新预算数据库            │
│     ├─ ... 版本迁移、加载电子表格、启动服务等 ...                 │
│     └─ setSyncingMode('enabled') → 启用同步                       │
└─────────────────────────────┬───────────────────────────────────┘
                              │
                              ├─ 此时状态：
                              │   ✅ prefs 内存 = 新预算配置
                              │   ✅ 数据库连接 = 新预算已打开
                              │
┌─────────────────────────────▼───────────────────────────────────┐
│  调用 syncBudget()                                               │
│  (budgetfiles/app.ts:210)                                        │
│  └─ initialFullSync() → ✅ 执行初始全量同步                        │
└─────────────────────────────┬───────────────────────────────────┘
                              │
┌─────────────────────────────▼───────────────────────────────────┐
│  后端 downloadBudget handler 返回 { id } → 成功                   │
└─────────────────────────────┬───────────────────────────────────┘
                              │
┌─────────────────────────────▼───────────────────────────────────┐
│  前端 downloadBudget thunk 成功后，Promise.all 并行执行：        │
│  ├─ loadGlobalPrefs()                                           │
│  ├─ loadAllFiles() → ✅ 重新获取并协调文件列表                    │
│  └─ 第2次调用 loadBudget({ id })                                 │
│     └─ 前端 loadBudget thunk                                    │
│        └─ send('load-budget', { id })                            │
│           └─ 后端 loadBudget handler                             │
│              ├─ currentPrefs = prefs.getPrefs() → ✅ 返回新预算配置 │
│              ├─ currentPrefs.id === id → ✅ true                 │
│              └─ return {} → ⚠️ 幂等性检查生效，直接空转返回！      │
└─────────────────────────────────────────────────────────────────┘
```

### 4.3 时序关键结论（✅ 代码证实）

| 调用次数 | 函数 | prefs 内存状态（调用前） | 数据库状态（调用前） | 实际行为 |
|---------|------|-------------------------|---------------------|----------|
| 第1次 | `closeBudget()` | 旧预算配置 | 旧预算已打开 | 正常执行所有关闭步骤 |
| 第2次 | `closeBudget()` | null | 已关闭 | 重复执行，大部分是无操作 |
| 第1次 | `loadBudget({ id })` | null | 已关闭 | 真正执行加载，prefs 和数据库都打开 |
| 第2次 | `loadBudget({ id })` | 新预算配置 | 新预算已打开 | 幂等性检查生效，直接空转返回 |

### 4.4 ❌ 之前分析的错误修正

1. **错误**：认为 `importBuffer` 会加载 prefs 到内存
   - **修正**：`importBuffer` 只写入文件系统，不修改内存状态。prefs 内存是 `null`，直到 `_loadBudget` 调用 `prefs.loadPrefs(id)`。

2. **错误**：认为第2次 `closeBudget` 时 prefs 已有值
   - **修正**：第2次 `closeBudget` 时 prefs 仍然是 `null`，因为 `importBuffer` 没有加载内存。

3. **错误**：没有明确区分「文件系统写入」和「内存加载」是两个独立操作
   - **修正**：`importBuffer` 写文件，`prefs.loadPrefs()` 读文件到内存，这是两个独立步骤。

---

## 五、其他分支时序（✅ 代码证实）

### 5.1 分支1：无打开预算 → 加载本地预算

```
初始状态：prefs 内存 = null，数据库 = 已关闭

前端 loadBudget thunk
    ↓
send('load-budget', { id })
    ↓
后端 loadBudget handler
    ├─ currentPrefs = prefs.getPrefs() → null
    └─ _loadBudget(id)
        ├─ prefs.loadPrefs(id) → prefs 内存 = 预算配置
        ├─ db.openDatabase(id) → 数据库打开
        └─ ... 后续加载 ...
    ↓
前端 loadPrefs() → 刷新前端偏好设置
```

### 5.2 分支2：有打开预算 → 切换到另一个本地预算

```
初始状态：prefs 内存 = 旧预算配置，数据库 = 旧预算已打开

前端 closeAndLoadBudget thunk
    ↓
closeBudget()
    ├─ resetApp() + queryClient.clear()
    └─ send('close-budget')
        └─ 后端 closeBudget handler
            ├─ db.closeDatabase() → 关闭旧数据库
            └─ prefs.unloadPrefs() → prefs 内存 = null
    ↓
此时状态：prefs 内存 = null，数据库 = 已关闭
    ↓
loadBudget({ id: fileId })
    └─ send('load-budget', { id })
        └─ 后端 loadBudget handler
            ├─ currentPrefs = null
            └─ _loadBudget(id)
                ├─ prefs.loadPrefs(id) → prefs 内存 = 新预算配置
                ├─ db.openDatabase(id) → 新数据库打开
                └─ ... 后续加载 ...
    ↓
前端 loadPrefs()
```

---

## 六、状态机图示（基于代码证实）

```
                    ┌─────────────────┐
                    │  loadAllFiles   │
                    │  (串行执行)     │
                    └─────────────────┘
                             │
                    ┌────────▼────────┐
                    │ send('get-      │
                    │ budgets')       │
                    └────────┬────────┘
                             │ 等待返回
                    ┌────────▼────────┐
                    │ send('get-      │
                    │ remote-files')  │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │ reconcileFiles  │
                    └────────┬────────┘
         ┌───────────┬───────┴───────┬───────────┐
         ▼           ▼               ▼           ▼
    ┌─────────┐ ┌────────┐      ┌──────────┐ ┌────────┐
    │ local   │ │ synced │      │ detached │ │ remote │
    └─────────┘ └────────┘      └──────────┘ └────────┘
         │          │                 │           │
         │          │                 │ ⚠️ UI 显示 │
         │          │                 │ Syncing    │
         └──────────┼─────────────────┴───────────┘
                    ▼
              用户点击切换
                    │
    ┌───────────────┼───────────────┐
    ▼               ▼               ▼
无打开预算     有预算→本地     有预算→云端
    │               │               │
    ▼               ▼               ▼
loadBudget   closeAndLoad     closeAndDownload
    │               │               │
    │               ├─ closeBudget  ├─ closeBudget (第1次)
    │               │               │  prefs: 旧→null
    │               │               │  DB: 已打开→已关闭
    │               └─ loadBudget   ├─ downloadBudget
    │                               │  ├─ cloudStorage.download
    │                               │  │  └─ importBuffer → 写文件
    │                               │  │     ⚠️ 不加载内存
    │                               │  ├─ closeBudget (第2次)
    │                               │  │  prefs: null→null
    │                               │  │  DB: 已关闭→已关闭
    │                               │  ├─ loadBudget (第1次)
    │                               │  │  prefs: null→新配置
    │                               │  │  DB: 已关闭→已打开
    │                               │  └─ syncBudget
    │                               └─ 前端并行:
    │                                  loadGlobalPrefs
    │                                  loadAllFiles
    │                                  loadBudget (第2次)
    │                                    → 幂等检查，空转返回 ✅
    │
    └───────────────┬───────────────┘
                    ▼
            预算加载完成
```

---

## 七、设计要点汇总

### ✅ 代码证实的优秀设计

| 设计点 | 代码位置 | 说明 |
|--------|---------|------|
| 目录名作为规范 ID | `budgetfiles/app.ts:123-125` | 避免用户移动目录导致 ID 失效 |
| loadBudget 幂等性 | `budgetfiles/app.ts:226-237` | 相同 ID 直接返回，允许多次安全调用 |
| 下载时保留备份 | `cloud-storage.ts:230-240` | 目录已存在时只删 db 和 meta，不删整个目录 |
| asyncStorage 异常保护 | `budgetfiles/app.ts:269-275` | 写入失败不影响整体关闭流程 |
| 并行下载优化 | `cloud-storage.ts:437-440` | 文件内容和元数据并行请求 |
| prefs.id 强制覆盖 | `prefs.ts:43` | 无论文件里存什么 id，强制使用目录名 |

### ✅ 代码证实的潜在问题

| 问题 | 代码位置 | 说明 |
|------|---------|------|
| detached 状态 UI 误导 | `BudgetFileSelection.tsx:187-214` | 缺失 case 'detached'，错误显示为 Syncing |
| closeBudget 大部分步骤无保护 | `budgetfiles/app.ts:257-279` | 8 个步骤中只有 1 个有 try-catch |
| 分支3中 closeBudget 两次调用 | `budgetfiles/app.ts:208` | 前端和后端各调用一次，第二次是无操作 |
| broken 文件仍可点击 | `budgetfilesSlice.ts:613-615` | 只是移到末尾，没有禁用点击 |
| loadAllFiles 串行执行 | `budgetfilesSlice.ts:40-41` | 两个 await 串行，比并行慢 |

### 🤔 合理推断（代码未直接证实）

| 推断 | 依据 | 可能性 |
|------|------|--------|
| detached 状态实际同步功能异常 | groupId 是 CRDT 同步组标识，不匹配意味着同步历史断裂 | 高 |
| broken 文件点击后会出错 | 云端已不存在该文件，同步操作会失败 | 高 |
| 第2次 closeBudget 调用是防御性编程 | 第1次关闭后资源已释放，第2次调用是为了确保状态干净 | 中 |
| loadAllFiles 串行是历史遗留代码 | 没有明显理由必须串行，可能可以优化为并行 | 中 |

---

## 八、相关文件索引（精确行号）

| 文件路径 | 关键代码段 | 行号 |
|----------|-----------|------|
| `packages/loot-core/src/types/file.ts` | FileState 类型定义 | 5-11 |
| `packages/loot-core/src/server/prefs.ts` | prefs 内存操作 | 22-87 |
| `packages/loot-core/src/server/db/index.ts` | 数据库打开/关闭 | 70-83 |
| `packages/desktop-client/src/budgetfiles/budgetfilesSlice.ts` | loadAllFiles 串行实现 | 37-47 |
| `packages/desktop-client/src/budgetfiles/budgetfilesSlice.ts` | closeBudget 前端实现 | 98-113 |
| `packages/desktop-client/src/budgetfiles/budgetfilesSlice.ts` | reconcileFiles 算法 | 516-616 |
| `packages/desktop-client/src/budgetfiles/budgetfilesSlice.ts` | unknown 状态判定 | 524-538 |
| `packages/desktop-client/src/budgetfiles/budgetfilesSlice.ts` | detached 状态判定 | 559-571 |
| `packages/desktop-client/src/budgetfiles/budgetfilesSlice.ts` | broken 状态判定 | 573-582 |
| `packages/desktop-client/src/budgetfiles/budgetfilesSlice.ts` | broken 文件排序 | 613-615 |
| `packages/desktop-client/src/components/manager/BudgetFileSelection.tsx` | onSelect 分支逻辑 | 598-612 |
| `packages/desktop-client/src/components/manager/BudgetFileSelection.tsx` | BudgetFileState switch | 187-214 |
| `packages/loot-core/src/server/budgetfiles/app.ts` | closeBudget 后端实现 | 257-280 |
| `packages/loot-core/src/server/budgetfiles/app.ts` | loadBudget 幂等性检查 | 226-237 |
| `packages/loot-core/src/server/budgetfiles/app.ts` | downloadBudget 后端链路 | 182-216 |
| `packages/loot-core/src/server/budgetfiles/app.ts` | _loadBudget 内部实现 | 508-640 |
| `packages/loot-core/src/server/cloud-storage.ts` | listRemoteFiles 返回 null | 374-403 |
| `packages/loot-core/src/server/cloud-storage.ts` | download 并行请求 | 405-440 |
| `packages/loot-core/src/server/cloud-storage.ts` | importBuffer 只写文件 | 193-249 |
