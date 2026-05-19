# 预算管理页面状态机分析（代码证实版）

## 重要说明

本文档所有结论均基于代码实证。每条结论前标注：
- ✅ **代码证实**：有明确代码为依据
- 🤔 **合理推断**：代码未直接说明，基于上下文的逻辑推断

---

## 一、loadAllFiles 获取顺序（已证实）

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

**完整调用栈**：
```
loadAllFiles (thunk)
    ↓ await send('get-budgets')
    ↓ 等待返回...
    ↓ await send('get-remote-files')
    ↓ 等待返回...
    ↓ dispatch(setAllFiles)
        ↓ reconcileFiles 状态协调
```

### ✅ 代码证实：setAllFiles 内部调用 reconcileFiles

`budgetfilesSlice.ts:453-460`：

```typescript
setAllFiles(state, action: PayloadAction<SetAllFilesPayload>) {
  state.budgets = action.payload.budgets;
  state.remoteFiles = action.payload.remoteFiles;
  state.allFiles = reconcileFiles(
    action.payload.budgets,
    action.payload.remoteFiles,
  );
}
```

---

## 二、closeBudget 的异常处理边界（已证实）

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

**异常处理边界总结**：
| 操作 | 是否有异常保护 | 失败影响 |
|------|---------------|----------|
| `sheet.waitOnSpreadsheet()` | ❌ 无 | 整个 closeBudget 失败 |
| `sheet.unloadSpreadsheet()` | ❌ 无 | 整个 closeBudget 失败 |
| `clearFullSyncTimeout()` | ❌ 无 | 整个 closeBudget 失败 |
| `mainApp.stopServices()` | ❌ 无 | 整个 closeBudget 失败 |
| `db.closeDatabase()` | ❌ 无 | 整个 closeBudget 失败 |
| `asyncStorage.setItem()` | ✅ 有 | 不影响其他步骤 |
| `prefs.unloadPrefs()` | ❌ 无 | 整个 closeBudget 失败 |
| `stopBackupService()` | ❌ 无 | 整个 closeBudget 失败 |

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

## 四、切换活跃预算的完整链路（代码证实）

### 4.1 前端触发点（✅ 代码证实）

`BudgetFileSelection.tsx:598-612`：
```typescript
const onSelect = async (file: File) => {
  const isRemoteFile = file.state === 'remote';

  if (!id) {
    // 分支1：当前没有打开的预算
    if (isRemoteFile) {
      await dispatch(downloadBudget({ cloudFileId: file.cloudFileId }));
    } else {
      await dispatch(loadBudget({ id: file.id }));
    }
  } else if (!isRemoteFile && file.id !== id) {
    // 分支2：当前有预算，切换到另一个本地预算
    await dispatch(closeAndLoadBudget({ fileId: file.id }));
  } else if (isRemoteFile) {
    // 分支3：当前有预算，下载并切换到云端预算
    await dispatch(closeAndDownloadBudget({ cloudFileId: file.cloudFileId }));
  }
};
```

### 4.2 分支2：切换本地预算完整链路（✅ 代码证实）

```
前端 closeAndLoadBudget thunk (budgetfilesSlice.ts:277-283)
    ↓
调用 closeBudget()
    ├─ ✅ 代码证实：resetApp() 重置 Redux 状态
    ├─ ✅ 代码证实：queryClient.clear() 清除 React Query 缓存
    └─ send('close-budget') 调用后端
        ├─ 等待电子表格完成
        ├─ 关闭数据库
        ├─ 清除 lastBudget（有 try-catch）
        ├─ 卸载 prefs
        └─ 停止备份服务
    ↓
调用 loadBudget({ id: fileId })
    ├─ send('load-budget', { id })
    │   └─ 后端 loadBudget handler
    │       ├─ ✅ 代码证实：检查 currentPrefs.id === id
    │       │   ├─ 相同 → 直接返回（幂等性）
    │       │   └─ 不同 → 先 closeBudget() 再 _loadBudget(id)
    │       └─ _loadBudget(id)
    │           ├─ 打开数据库
    │           ├─ 版本迁移
    │           ├─ 加载电子表格
    │           ├─ 启动服务
    │           └─ 启用同步
    └─ ✅ 代码证实：loadPrefs() 刷新前端偏好设置
```

### 4.3 分支3：下载云端预算完整链路（✅ 代码证实）

```
前端 closeAndDownloadBudget thunk (budgetfilesSlice.ts:289-295)
    ↓
调用 closeBudget() → 同分支2
    ↓
调用 downloadBudget({ cloudFileId, replace: true })

前端 downloadBudget thunk (budgetfilesSlice.ts:302-379)
    ├─ 显示 "Downloading..."
    ├─ send('download-budget', { cloudFileId })
    │   └─ 后端 downloadBudget handler (budgetfiles/app.ts:182-216)
    │       ├─ ✅ 代码证实：cloudStorage.download(cloudFileId)
    │       │   ├─ 并行获取 fileInfo 和 fileBuffer
    │       │   ├─ 解密（如有 encryptMeta）
    │       │   └─ importBuffer 写入本地
    │       ├─ ✅ 代码证实：await closeBudget() → 第2次调用关闭
    │       ├─ ✅ 代码证实：await loadBudget({ id }) → 第1次加载
    │       └─ ✅ 代码证实：await syncBudget()
    │           └─ initialFullSync() 执行初始全量同步
    └─ 成功后 Promise.all 并行执行：
        ├─ loadGlobalPrefs()
        ├─ ✅ 代码证实：loadAllFiles() → 重新获取并协调文件列表
        └─ ✅ 代码证实：loadBudget({ id }) → 第2次调用加载
```

### 4.4 关键细节：loadBudget 的幂等性（✅ 代码证实）

`budgetfiles/app.ts:226-242`：
```typescript
async function loadBudget({ id }) {
  const currentPrefs = prefs.getPrefs();

  if (currentPrefs) {
    if (currentPrefs.id === id) {
      // ✅ 代码证实：ID 相同直接返回，不重复加载
      return {};
    } else {
      await closeBudget();
    }
  }

  return _loadBudget(id);
}
```

**这解释了为什么分支3中 loadBudget 被调用两次是安全的**：
- 第1次：后端 downloadBudget handler 中调用，实际执行加载
- 第2次：前端 thunk 中调用，此时 prefs.id 已匹配，直接空转返回

### 4.5 关键细节：closeBudget 被调用两次（✅ 代码证实）

分支3中：
1. 前端 `closeAndDownloadBudget` thunk 中调用 `closeBudget()` → 第1次
2. 后端 `downloadBudget` handler 中调用 `closeBudget()` → 第2次

**第2次调用时**，`prefs.getPrefs()` 已被 `cloudStorage.download()` 中的 `importBuffer` 间接加载（importBuffer 写入 metadata.json，但不会加载到内存），所以：
- 后端 `closeBudget` 没有前置检查，会执行所有步骤
- 但此时数据库可能已被 loadBudget 打开，行为取决于各函数的实现

---

## 五、cloudStorage.download 内部流程（✅ 代码证实）

### 5.1 并行下载优化

`cloud-storage.ts:405-440`：
```typescript
export async function download(cloudFileId) {
  // ✅ 代码证实：两个请求并行执行
  const [userFileInfoRes, userFileRes] = await Promise.all([
    fetchJSON('/get-user-file-info'),   // 获取元数据
    fetch('/download-user-file')        // 获取 zip 文件内容
  ]);
  // ...
}
```

### 5.2 importBuffer 写入逻辑

`cloud-storage.ts:193-249`：
```typescript
export async function importBuffer(fileData, buffer) {
  // 1. 解压 zip
  const dbEntry = entries.find(e => e.entryName.includes('db.sqlite'));
  const metaEntry = entries.find(e => e.entryName.includes('metadata.json'));

  // 2. ✅ 代码证实：更新 metadata，用服务器的数据覆盖
  meta = {
    ...meta,
    cloudFileId: fileData.fileId,      // 服务器返回的 fileId
    groupId: fileData.groupId,          // 服务器返回的 groupId
    lastUploaded: monthUtils.currentDay(),
    encryptKeyId: fileData.encryptMeta ? fileData.encryptMeta.keyId : null,
  };

  // 3. ✅ 代码证实：目录已存在时只删 db 和 meta，保留备份
  if (await fs.exists(budgetDir)) {
    // 不删除整个目录，保留备份
    if (await fs.exists(dbFile)) await fs.removeFile(dbFile);
    if (await fs.exists(metaFile)) await fs.removeFile(metaFile);
  }

  // 4. 写入新文件
  await fs.writeFile(dbFile, dbContent);
  await fs.writeFile(metaFile, JSON.stringify(meta));

  return { id: meta.id };
}
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
    │               └─ loadBudget   └─ downloadBudget
    │                               ├─ cloudStorage.download
    │                               ├─ closeBudget (第2次) ✅
    │                               ├─ loadBudget (第1次) ✅
    │                               ├─ syncBudget ✅
    │                               └─ 前端并行:
    │                                  loadGlobalPrefs
    │                                  loadAllFiles
    │                                  loadBudget (第2次) → 空转 ✅
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

### ✅ 代码证实的潜在问题

| 问题 | 代码位置 | 说明 |
|------|---------|------|
| detached 状态 UI 误导 | `BudgetFileSelection.tsx:187-214` | 缺失 case 'detached'，错误显示为 Syncing |
| closeBudget 大部分步骤无保护 | `budgetfiles/app.ts:257-279` | 8 个步骤中只有 1 个有 try-catch |
| 分支3中 closeBudget 两次调用 | `budgetfiles/app.ts:208` | 前端和后端各调用一次，第二次可能有副作用 |
| broken 文件仍可点击 | `budgetfilesSlice.ts:613-615` | 只是移到末尾，没有禁用点击 |
| loadAllFiles 串行执行 | `budgetfilesSlice.ts:40-41` | 两个 await 串行，比并行慢 |

### 🤔 合理推断（代码未直接证实）

| 推断 | 依据 | 可能性 |
|------|------|--------|
| detached 状态实际同步功能异常 | groupId 是 CRDT 同步组标识，不匹配意味着同步历史断裂 | 高 |
| broken 文件点击后会出错 | 云端已不存在该文件，同步操作会失败 | 高 |
| 第2次 closeBudget 调用是冗余的 | 第1次关闭后资源已释放，第2次可能是防御性编程 | 中 |
| loadAllFiles 串行是历史遗留代码 | 没有明显理由必须串行，可能可以优化为并行 | 中 |

---

## 八、相关文件索引（精确行号）

| 文件路径 | 关键代码段 | 行号 |
|----------|-----------|------|
| `packages/loot-core/src/types/file.ts` | FileState 类型定义 | 5-11 |
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
| `packages/loot-core/src/server/cloud-storage.ts` | listRemoteFiles 返回 null | 374-403 |
| `packages/loot-core/src/server/cloud-storage.ts` | download 并行请求 | 405-440 |
| `packages/loot-core/src/server/cloud-storage.ts` | importBuffer 保留备份 | 230-240 |
