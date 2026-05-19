# 预算管理页面状态机分析

## 一、文件状态类型系统

预算文件的状态管理基于一个精心设计的类型系统，定义在 `packages/loot-core/src/types/file.ts`。

### 1.1 核心状态枚举

```typescript
type FileState =
  | 'local'     // 仅本地存在，未同步
  | 'remote'    // 仅云端存在，可下载
  | 'synced'    // 本地已下载且与云端同步
  | 'detached'  // 本地存在但 groupId 不匹配（同步状态断裂）
  | 'broken'    // 用户不应访问此文件
  | 'unknown';  // 离线，无法确定状态
```

### 1.2 文件类型层次

| 类型 | 说明 | 关键字段 |
|------|------|----------|
| `LocalFile` | 纯本地文件 | `state: 'local'` |
| `SyncableLocalFile` | 可同步的本地文件（状态异常） | `cloudFileId`, `groupId`, `state: 'broken' \| 'unknown'` |
| `SyncedLocalFile` | 已同步的本地文件 | `cloudFileId`, `groupId`, `state: 'synced' \| 'detached'` |
| `RemoteFile` | 云端文件 | `cloudFileId`, `groupId`, `state: 'remote'` |

---

## 二、本地与远端文件并列展示

### 2.1 数据获取流程

在 `packages/desktop-client/src/budgetfiles/budgetfilesSlice.ts` 中：

1. **加载本地文件**：`loadBudgets()` 调用 `get-budgets` 消息，遍历文档目录下的所有预算文件夹，读取 `metadata.json` 获取预算信息。

2. **加载远端文件**：`loadRemoteFiles()` 调用 `get-remote-files` 消息，从同步服务器获取云端文件列表。

3. **合并加载**：`loadAllFiles()` 并行获取本地和远端文件，然后调用 `reconcileFiles()` 进行状态协调。

### 2.2 状态协调算法（reconcileFiles）

`reconcileFiles(localFiles, remoteFiles)` 是整个状态机的核心，它将本地和远端文件列表合并为统一的展示列表。

**算法流程：**

```
1. 遍历所有本地文件：
   a. 如果文件有 cloudFileId 和 groupId：
      i. 若 remoteFiles 为 null（网络错误）→ 标记为 'unknown'
      ii. 查找匹配的远端文件：
          - 找到且 groupId 匹配 → 标记为 'synced'
          - 找到但 groupId 不匹配 → 标记为 'detached'（冲突状态）
          - 未找到 → 标记为 'broken'
   b. 如果没有 cloudFileId → 标记为 'local'

2. 添加未匹配的远端文件：
   - 遍历 remoteFiles 中未被 reconciled 标记的文件
   - 标记为 'remote' 状态

3. 过滤已删除的文件

4. 排序：
   - 按名称字母顺序排序
   - 名称相同时按 ID 排序
   - 'broken' 状态的文件移到列表末尾
```

**关键代码位置**：`budgetfilesSlice.ts:516-616`

### 2.3 UI 展示层

在 `BudgetFileSelection.tsx` 中：

- 使用 `GridList` 组件展示所有文件
- 每个文件项通过 `BudgetFileState` 组件显示状态图标和文字
- 本地文件显示 `SvgFileDouble` 图标和 "Local" 文字
- 云端文件显示 `SvgCloudDownload` 图标和 "Available for download" 文字
- 已同步文件显示 `SvgCloudCheck` 图标和 "Syncing" 文字
- 离线状态显示 `SvgCloudUnknown` 图标和 "Network unavailable" 文字

---

## 三、冲突状态识别

### 3.1 冲突类型

系统识别以下几种冲突/异常状态：

| 状态 | 触发条件 | 用户可见表现 |
|------|----------|--------------|
| `detached` | 本地文件的 `groupId` 与云端不匹配 | 显示为 "Syncing" 但实际同步已断裂，需要重置同步 |
| `broken` | 本地有 `cloudFileId` 但云端不存在该文件 | 移到列表末尾，用户不应正常访问 |
| `unknown` | 离线时无法获取云端状态 | 显示 "Network unavailable" |
| 加密密钥缺失 | 文件有 `encryptKeyId` 但用户没有密钥 | 钥匙图标变灰，无法打开 |

### 3.2 冲突检测点

1. **在 `reconcileFiles` 中**（`budgetfilesSlice.ts:546-571`）：
   ```typescript
   if (remote.groupId === localFile.groupId) {
     state: 'synced'  // 正常同步
   } else {
     state: 'detached'  // groupId 不匹配，检测到冲突
   }
   ```

2. **在下载预算时**（`budgetfilesSlice.ts:318-365`）：
   - `decrypt-failure`：解密失败，可能缺少密钥
   - `file-exists`：本地已有相同 ID 的文件

3. **在加载预算时**（`app.ts:550-568`）：
   - `out-of-sync-migrations`：数据库迁移版本不一致
   - `out-of-sync-data`：数据同步状态异常

---

## 四、切换活跃预算的清理与重新加载

### 4.1 切换流程总览

当用户点击切换预算时，`BudgetFileSelection.tsx:598-612` 中的 `onSelect` 函数被调用：

```
用户选择文件
    ↓
判断文件类型（本地/远端）
    ↓
如果已有打开的预算 → 先关闭
    ↓
加载新预算（本地加载 or 远端下载）
```

### 4.2 预算关闭流程（closeBudget）

定义在 `app.ts:257-280`，这是清理工作的核心：

```typescript
async function closeBudget() {
  // 1. 等待电子表格计算完成
  await sheet.waitOnSpreadsheet();
  sheet.unloadSpreadsheet();

  // 2. 清除同步超时
  clearFullSyncTimeout();
  await mainApp.stopServices();

  // 3. 关闭数据库连接
  db.closeDatabase();

  // 4. 清除本地存储
  await asyncStorage.setItem('lastBudget', '');

  // 5. 卸载偏好设置
  prefs.unloadPrefs();

  // 6. 停止备份服务
  stopBackupService();
}
```

**前端补充清理**（`budgetfilesSlice.ts:98-113`）：
```typescript
export const closeBudget = createAppAsyncThunk(
  `${sliceName}/closeBudget`,
  async (_, { dispatch, getState, extra: { queryClient } }) => {
    dispatch(resetApp());        // 重置应用状态
    queryClient.clear();         // 清除 React Query 缓存
    dispatch(setAppState({ loadingText: t('Closing...') }));
    await send('close-budget');  // 调用后端关闭逻辑
    // ...
  }
);
```

### 4.3 预算加载流程（loadBudget）

定义在 `app.ts:508-640`：

```typescript
async function _loadBudget(id) {
  // 1. 验证目录存在
  const dir = fs.getBudgetDir(id);

  // 2. 加载偏好设置和数据库
  await prefs.loadPrefs(id);
  await db.openDatabase(id);

  // 3. 执行数据库版本迁移
  await updateVersion();

  // 4. 加载同步时钟
  await db.loadClock();

  // 5. 启动备份服务
  startBackupService(id);

  // 6. 加载电子表格引擎
  await sheet.loadSpreadsheet(db, onSheetChange);

  // 7. 加载业务数据
  await mappings.loadMappings();
  await rules.loadRules();
  syncMigrations.listen();
  mainApp.startServices();

  // 8. 启用同步
  setSyncingMode('enabled');
  await asyncStorage.setItem('lastBudget', id);
  await cloudStorage.possiblyUpload();
}
```

### 4.4 组合操作

系统提供了几个组合 thunk 来简化常见操作：

- **`closeAndLoadBudget`**：关闭当前预算 → 加载新的本地预算
- **`closeAndDownloadBudget`**：关闭当前预算 → 下载并加载云端预算
- **`loadBackup`**：关闭当前预算 → 恢复备份 → 加载恢复后的预算

---

## 五、状态机图示

```
                    ┌─────────────────┐
                    │   预算管理器    │
                    └─────────────────┘
                             │
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
       ┌──────────┐   ┌──────────┐   ┌──────────┐
       │ 本地文件 │   │ 同步文件 │   │ 云端文件 │
       └──────────┘   └──────────┘   └──────────┘
              │              │              │
              └──────────────┼──────────────┘
                             ▼
                    ┌─────────────────┐
                    │ reconcileFiles  │  状态协调
                    └─────────────────┘
                             │
         ┌───────────┬───────┴───────┬───────────┐
         ▼           ▼               ▼           ▼
    ┌─────────┐ ┌────────┐      ┌──────────┐ ┌────────┐
    │ local   │ │ synced │      │ detached │ │ remote │
    └─────────┘ └────────┘      └──────────┘ └────────┘
         │          │                 │           │
         └──────────┼─────────────────┼───────────┘
                    ▼                 ▼
              点击切换           groupId 冲突
                    │                 │
                    ▼                 ▼
            ┌─────────────┐   ┌─────────────────┐
            │ closeBudget │   │ 显示冲突指示器 │
            └─────────────┘   └─────────────────┘
                    │
                    ▼
            ┌────────────┐
            │ loadBudget │
            └────────────┘
```

---

## 六、关键设计要点

### 6.1 数据一致性保证

- 使用目录名作为预算的规范 ID，避免用户移动/重命名目录导致的问题
- `groupId` 用于检测同步状态的一致性，不匹配时标记为 `detached`
- 所有数据库操作都经过版本迁移检查

### 6.2 容错设计

- 网络错误时优雅降级为 `unknown` 状态，不吓用户
- 关闭预算时使用 try-catch 包裹，确保即使部分失败也能继续卸载
- 加载失败时确保资源被正确清理（调用 `closeBudget()`）

### 6.3 性能优化

- 预算列表通过 `reconcileFiles` 一次性计算完成
- 使用 `reconciled` Set 避免重复匹配
- 排序算法稳定，相同名称按 ID 排序

### 6.4 安全考虑

- 加密文件通过 `encryptKeyId` 和 `hasKey` 字段双重验证
- 解密失败时引导用户修复密钥，而不是直接报错
- `broken` 状态的文件被移到列表末尾，减少误操作风险

---

## 七、相关文件索引

| 文件路径 | 主要职责 |
|----------|----------|
| `packages/loot-core/src/types/file.ts` | 文件状态类型定义 |
| `packages/desktop-client/src/budgetfiles/budgetfilesSlice.ts` | 前端状态管理、reconcileFiles 算法 |
| `packages/desktop-client/src/components/manager/BudgetFileSelection.tsx` | 预算选择 UI 组件 |
| `packages/loot-core/src/server/budgetfiles/app.ts` | 服务端预算管理核心逻辑 |
| `packages/loot-core/src/server/budget/base.ts` | 预算基础操作 |
| `packages/desktop-client/src/app/appSlice.ts` | 应用级状态重置 |
