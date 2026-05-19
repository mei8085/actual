# 桌面端更新与刷新机制深度分析

本文档深度解析 Actual Budget 桌面端在以下两种场景下的完整技术实现：
1. 检测到应用自身新版本 → 提示 → 用户接受 → 刷新/重启
2. 当前预算文件被远端同步覆盖 → 提示 → 用户选择 Revert/Upload → 数据重置与重载

---

## 一、应用版本更新机制深度解析

### 1.1 检测时机与通知状态来源

#### 整体架构概览

应用版本更新检测采用**双轨机制**：

| 检测方式 | 触发时机 | 适用环境 | 通知类型 |
|---------|---------|---------|---------|
| **主动检测（PWA SW）** | Service Worker `onNeedRefresh` 事件 | 浏览器/PWA | 热更新提示（需立即刷新） |
| **被动检测（GitHub API）** | `FinancesApp` 组件挂载时 | 浏览器 + Electron | 新版本发布提示（仅信息） |

---

#### 检测链路一：PWA Service Worker 热更新（浏览器环境）

**代码位置**：
- 初始化：`packages/desktop-client/src/browser-preload.js:330-348`
- 监听：`packages/desktop-client/src/components/FinancesApp.tsx:109-140`

**完整调用链**：

```
浏览器启动 → browser-preload.js 执行
    ↓
registerSW({ immediate: true, onNeedRefresh: markUpdateReadyForDownload })
    ↓
[ 后台等待 Service Worker 检测到新版本 ]
    ↓
新 SW 进入 waiting 状态 → 触发 onNeedRefresh 回调
    ↓
markUpdateReadyForDownload() 被调用
    ↓
isUpdateReadyForDownload = true
isUpdateReadyForDownloadPromise.resolve(true)
    ↓
FinancesApp 中 waitForUpdateReadyForDownload() Promise 被 resolve
    ↓
dispatch(addNotification({ id: 'update-reload-notification' }))
    ↓
UI 显示："A new version of Actual is available! Click the button below to reload and apply the update."
    ↓
按钮："Update now" → 调用 global.Actual.applyAppUpdate()
```

**关键状态变量**：
- `isUpdateReadyForDownload` (boolean): 标记是否有更新待下载
- `isUpdateReadyForDownloadPromise` (Promise): 阻塞直到更新就绪

---

#### 检测链路二：GitHub API 被动版本检查（浏览器 + Electron）

**代码位置**：
- 版本获取：`packages/desktop-client/src/util/versions.ts`
- 触发：`packages/desktop-client/src/components/FinancesApp.tsx:144-146`

**完整调用链**：

```
FinancesApp 组件挂载 → useEffect 触发
    ↓
dispatch(getLatestAppVersion())
    ↓
检查 globalPrefs.notifyWhenUpdateIsAvailable 是否为 true
    ↓
getLatestVersion() → fetch GitHub API /repos/actualbudget/actual/releases/latest
    ↓
返回 tag_name (如 "v26.5.2")
    ↓
getIsOutdated(latestVersion) → 比较 window.Actual.ACTUAL_VERSION 与 latestVersion
    ↓
如果 outdated → dispatch(setAppState({ versionInfo: { latestVersion, isOutdated: true } }))
    ↓
另一个 useEffect 监听 versionInfo 变化
    ↓
如果 isOutdated && lastUsedVersion !== latestVersion
    ↓
dispatch(addNotification({ id: 'update-notification' }))
    ↓
UI 显示："Version {{latestVersion}} of Actual was recently released."
    ↓
按钮："Open changelog" → 打开浏览器到 releases 页面
onClose: setLastUsedVersion(latestVersion)  // 避免重复提示
```

**关键偏好设置**：
- `notifyWhenUpdateIsAvailable` (全局偏好): 用户是否希望收到更新通知
- `flags.updateNotificationShownForVersion` (本地偏好): 记录已提示过的版本号

---

#### Electron 环境的特殊处理

**代码位置**：`packages/desktop-electron/preload.ts:79-110`

**Electron 中自动更新功能被完全禁用**：

```javascript
// No auto-updates in the desktop app
isUpdateReadyForDownload: () => false,
waitForUpdateReadyForDownload: () =>
  new Promise<void>(() => {
    // This is used in browser environment; do nothing in electron
  }),

applyAppUpdate: async () => {
  throw new Error('applyAppUpdate not implemented in electron app');
}
```

**但 Electron 提供了完整的重启能力**：

```javascript
relaunch: () => {
  void ipcRenderer.invoke('relaunch');
}
```

**主进程处理**：`packages/desktop-electron/index.ts:566-569`
```javascript
ipcMain.handle('relaunch', () => {
  app.relaunch();   // Electron API: 重启应用
  app.exit();       // 立即退出当前进程
});
```

> **重要说明**：Electron 应用的实际更新依赖外部机制（如 Windows Store、macOS Sparkle、Linux 包管理器），应用内仅做版本号显示和更新发布提示。

---

### 1.2 通知 UI 状态完整说明

系统中存在三种更新相关的通知/UI 组件：

| 类型 | 触发条件 | 显示位置 | 按钮动作 |
|-----|---------|---------|---------|
| **PWA 热更新通知** | SW 检测到新版本 | 通知条 | "Update now" → `applyAppUpdate()` |
| **新版本发布通知** | GitHub API 检测到更新 | 通知条 | "Open changelog" → 打开浏览器 |
| **更新完成提示** | `updateInfo` 状态被设置 | 右下角浮动 | "Restart" → `updateApp()` |

> **注意**：第三种 `UpdateNotification` 组件的 `updateInfo` 状态在当前代码中**没有任何地方设置**，仅在 `appSlice.ts` 中定义了状态结构和 `updateApp` action。这是一个预留接口，目前未被实际使用。

---

### 1.3 用户点击后的完整调用链与重启入口

#### 浏览器/PWA 环境："Update now" 按钮

**代码位置**：`packages/desktop-client/src/browser-preload.js:495-502`

```
用户点击 "Update now"
    ↓
global.Actual.applyAppUpdate()
    ↓
updateSW()  // 来自 virtual:pwa-register
    ↓
[ 内部执行 ]
    ├─ 跳过 waiting 的 Service Worker
    └─ 触发 location.reload()
    ↓
页面刷新 → 新 SW 生效
```

---

#### Electron 环境：无直接更新入口

Electron 中 `applyAppUpdate` 会抛出错误。但如果代码其他地方调用 `relaunch()`：

```
global.Actual.relaunch()
    ↓
ipcRenderer.invoke('relaunch')
    ↓
主进程 ipcMain.handle('relaunch')
    ↓
app.relaunch()   // Electron 原生 API：安排重启
app.exit()       // 立即退出
    ↓
应用进程终止 → 操作系统重新启动应用
```

---

## 二、预算文件被远端同步覆盖机制深度解析

### 2.1 检测时机与错误来源

**代码位置**：`packages/desktop-client/src/sync-events.ts:249-283`

**完整检测链路**：

```
同步过程中（sync() 被调用）
    ↓
fullSync() 执行 → 与服务器比较 groupId 和 encryptKeyId
    ↓
如果服务器 groupId 与本地不匹配 → 抛出 SyncError('file-has-reset')
如果服务器 encryptKeyId 与本地不匹配 → 抛出 SyncError('file-has-new-key')
    ↓
app.events.emit('sync', { type: 'error', subtype: 'file-has-reset' })
    ↓
listenForSyncEvent() 监听到 sync-event
    ↓
event.type === 'error' && event.subtype === 'file-has-reset' / 'file-has-new-key'
    ↓
检查 store.getState().prefs.local.cloudFileId 是否存在
    ↓
构造 notification 并 dispatch(addNotification({ id: 'needs-revert' }))
    ↓
UI 显示："Syncing has been reset on this cloud file"
```

**关键错误触发点**（后端）：
- `packages/loot-core/src/server/sync/index.ts` 的 `_fullSync()` 函数中进行 groupId 和 keyId 校验
- 校验失败时抛出 `SyncError`，并通过事件系统传递到前端

---

### 2.2 提示 UI 与用户选择

**通知内容**：`sync-events.ts:263-282`

```javascript
notif = {
  title: 'Syncing has been reset on this cloud file',
  message: 'You need to revert it to continue syncing. Any unsynced data will be lost. If you like, you can instead [upload this file](#upload) to be the latest version.',
  messageActions: { upload: () => store.dispatch(resetSync()) },
  sticky: true,
  id: 'needs-revert',
  button: {
    title: 'Revert',
    action: () => {
      void store.dispatch(closeAndDownloadBudget({ cloudFileId }));
    },
  },
};
```

用户有两个互斥选择：

| 选项 | 含义 | 数据流向 | 风险 |
|-----|------|---------|------|
| **Revert** | 放弃本地修改，以服务器版本为准 | 服务器 → 本地 | 未同步的本地修改丢失 |
| **Upload** | 以本地版本为准，强制覆盖服务器 | 本地 → 服务器 | 其他设备的修改丢失 |

---

### 2.3 Revert 分支：状态重置与数据重载完整流程

**入口**：`closeAndDownloadBudget({ cloudFileId })`

**代码位置**：`packages/desktop-client/src/budgetfiles/budgetfilesSlice.ts:289-294`

```javascript
export const closeAndDownloadBudget = createAppAsyncThunk(
  `${sliceName}/closeAndDownloadBudget`,
  async ({ cloudFileId }, { dispatch }) => {
    await dispatch(closeBudget());          // 第一步：关闭当前预算
    await dispatch(downloadBudget({ cloudFileId, replace: true }));  // 第二步：下载新版本
  },
);
```

---

#### Revert 第一步：closeBudget() 完整状态重置

**代码位置**：`packages/desktop-client/src/budgetfiles/budgetfilesSlice.ts:98-113`

```
dispatch(closeBudget())
    ↓
检查 getState().prefs.local.id 是否存在（预算已加载）
    ↓
┌─────────────────────────────────────────────────────────┐
│ dispatch(resetApp())  [关键：触发全应用状态重置]          │
└─────────────────────────────────────────────────────────┘
    ↓
    ↳ 所有监听 resetApp 的 slice 执行状态重置：
    │
    ├─ appSlice: 重置为 initialState（保留 loadingText 和 managerHasInitialized）
    ├─ budgetfilesSlice: 重置为 initialState
    ├─ accountsSlice: 重置为 initialState
    ├─ transactionsSlice: 重置为 initialState
    ├─ modalsSlice: 重置为 initialState（关闭所有模态框）
    ├─ notificationsSlice: 重置为 initialState（清除所有通知）
    ├─ prefsSlice: 重置为 initialState（**保留 global 和 server 偏好**）
    └─ usersSlice: 重置为 initialState
    ↓
queryClient.clear()  // 清除 React Query 所有缓存数据
    ↓
dispatch(setAppState({ loadingText: 'Closing...' }))
    ↓
send('close-budget')  // IPC 调用后端关闭预算
    ↓
后端执行 closeBudget():
    ├─ sheet.waitOnSpreadsheet()  // 等待电子表格完成
    ├─ sheet.unloadSpreadsheet()  // 卸载电子表格
    ├─ clearFullSyncTimeout()     // 清除同步超时
    ├─ mainApp.stopServices()     // 停止所有后台服务
    ├─ db.closeDatabase()         // 关闭 SQLite 数据库连接
    ├─ asyncStorage.setItem('lastBudget', '')  // 清除上次预算记录
    ├─ prefs.unloadPrefs()        // 卸载偏好设置
    └─ stopBackupService()        // 停止备份服务
    ↓
dispatch(setAppState({ loadingText: null }))
    ↓
[特殊情况] 如果 SharedArrayBufferOverride 存在 → window.location.reload()
```

**关键设计**：
- `resetApp` 是一个全局 action，通过 `extraReducers` 被各个 slice 监听
- `prefsSlice` 特殊处理：**保留 global 和 server 偏好**，只清空 local 偏好
- 后端 `closeBudget()` 确保所有资源被正确释放

---

#### Revert 第二步：downloadBudget() 数据重载

**代码位置**：`packages/desktop-client/src/budgetfiles/budgetfilesSlice.ts:302-378`

```
dispatch(downloadBudget({ cloudFileId, replace: true }))
    ↓
dispatch(setAppState({ loadingText: 'Downloading...' }))
    ↓
send('download-budget', { cloudFileId })  // IPC 调用后端下载
    ↓
后端执行 downloadBudget(cloudFileId):
    │
    ├─ cloudStorage.download(cloudFileId)
    │   │
    │   ├─ 并行请求：
    │   │   ├─ GET /download-user-file → 获取加密的预算文件 zip
    │   │   └─ GET /get-user-file-info → 获取文件元数据（groupId, encryptMeta 等）
    │   │
    │   ├─ 如果 encryptMeta 存在 → 使用本地密钥解密 buffer
    │   │   └─ 解密失败 → 抛出 FileDownloadError('decrypt-failure')
    │   │
    │   └─ importBuffer(fileData, buffer)
    │       │
    │       ├─ 解压 zip → 提取 db.sqlite 和 metadata.json
    │       ├─ 构造新 meta：更新 cloudFileId, groupId, encryptKeyId
    │       ├─ 写入本地文件系统：<budgetDir>/db.sqlite 和 metadata.json
    │       └─ 返回 { id: meta.id }
    │
    ├─ closeBudget()  // 再次关闭（确保干净）
    ├─ loadBudget({ id })  // 加载新下载的预算
    └─ syncBudget() → initialFullSync()  // 执行初始全量同步
    ↓
前端收到 { id, error }
    ↓
如果成功：
    ├─ Promise.all([
    │   dispatch(loadGlobalPrefs()),   // 重新加载全局偏好
    │   dispatch(loadAllFiles()),      // 重新加载所有文件列表
    │   dispatch(loadBudget({ id })),  // 加载预算数据到 UI
    │ ])
    └─ dispatch(setAppState({ loadingText: null }))
```

**后端关键函数**：
- `cloudStorage.download()`: `packages/loot-core/src/server/cloud-storage.ts:405-466`
- `importBuffer()`: `packages/loot-core/src/server/cloud-storage.ts:193-249`
- `initialFullSync()`: `packages/loot-core/src/server/sync/index.ts:585-595`

---

### 2.4 Upload 分支：状态重置与数据重载完整流程

**入口**：`resetSync()`

**代码位置**：`packages/desktop-client/src/app/appSlice.ts:47-86`

```javascript
export const resetSync = createAppAsyncThunk(
  `${sliceName}/resetSync`,
  async (_, { dispatch }) => {
    const { error } = await send('sync-reset');
    if (error) {
      // 处理各种错误情况（加密密钥问题等）
    } else {
      await dispatch(sync());  // 成功后触发同步
    }
  },
);
```

---

#### Upload 后端完整执行流程

**代码位置**：`packages/loot-core/src/server/sync/reset.ts:10-91`

```
send('sync-reset')
    ↓
后端执行 resetSync(keyState?):
    │
    ├─ 如果没有传入 keyState → cloudStorage.checkKey()
    │   └─ 验证本地密钥与服务器密钥是否匹配
    │   └─ 不匹配 → 返回 { error: { reason: 'file-has-new-key' } }
    │
    ├─ cloudStorage.resetSyncState(keyState)
    │   └─ 向服务器 POST /reset-sync-state
    │   └─ 服务器创建新的 groupId
    │
    ├─ runMutator()  [关键：数据库操作]
    │   │
    │   └─ 执行 SQL 清理同步状态：
    │       ├─ DELETE FROM messages_crdt;          -- 清除所有 CRDT 消息
    │       ├─ DELETE FROM messages_clock;         -- 清除时钟状态
    │       ├─ DELETE FROM transactions WHERE tombstone = 1;  -- 清除已删除的交易
    │       ├─ DELETE FROM accounts WHERE tombstone = 1;      -- 清除已删除的账户
    │       ├─ DELETE FROM payees WHERE tombstone = 1;        -- 清除已删除的收款人
    │       ├─ DELETE FROM categories WHERE tombstone = 1;    -- 清除已删除的分类
    │       ├─ DELETE FROM category_groups WHERE tombstone = 1;  -- 清除已删除的分类组
    │       ├─ DELETE FROM schedules WHERE tombstone = 1;     -- 清除已删除的计划
    │       ├─ DELETE FROM rules WHERE tombstone = 1;         -- 清除已删除的规则
    │       ├─ ANALYZE;  -- 分析表统计信息
    │       └─ VACUUM;   -- 压缩数据库
    │
    ├─ db.loadClock()  -- 重新加载时钟
    │
    ├─ prefs.savePrefs({
    │   groupId: null,              -- 清空旧 groupId（将在上传后设置新的）
    │   lastSyncedTimestamp: null,  -- 清空同步时间戳
    │   lastUploaded: null,         -- 清空上次上传时间
    │ })
    │
    ├─ 如果传入 keyState → 更新本地加密密钥存储
    │
    └─ cloudStorage.upload()  [关键：上传本地文件作为最新版本]
        │
        ├─ exportBuffer() → 将本地数据库打包为 zip
        ├─ 如果有 encryptKeyId → 使用密钥加密 zip
        ├─ POST /upload-user-file 到同步服务器
        └─ 成功后保存 prefs：{ lastUploaded, cloudFileId, groupId: 新的 groupId }
    ↓
返回 {} （成功）或 { error }
    ↓
前端 dispatch(sync()) → 执行同步拉取服务器上的最新状态
```

**关键设计**：
- `resetSync` 是**破坏性操作**：清除所有 CRDT 历史，压缩数据库
- 服务器创建新的 `groupId`，旧 groupId 失效
- 其他设备下次同步时会检测到 `file-has-reset` 错误，被迫执行 Revert
- 本地未同步的修改**不会丢失**，因为它们已经存在于本地数据库中

---

### 2.5 Revert vs Upload 关键对比

| 维度 | Revert（还原） | Upload（上传） |
|-----|---------------|---------------|
| **数据流向** | 服务器 → 本地 | 本地 → 服务器 |
| **本地未同步修改** | 丢失 | 保留 |
| **其他设备影响** | 无影响 | 其他设备被迫 Revert |
| **groupId 变化** | 使用服务器的新 groupId | 创建全新的 groupId |
| **CRDT 历史** | 继承服务器的历史 | 全部清除，重新开始 |
| **数据库操作** | 替换本地 db 文件 | 清理 tombstone，压缩 VACUUM |
| **触发动作** | `closeAndDownloadBudget` | `resetSync` |
| **后端入口** | `download-budget` | `sync-reset` |

---

## 三、完整状态重置清单（resetApp 影响范围）

当 `resetApp` action 被 dispatch 时，以下 slice 会执行状态重置：

**代码位置**：各个 slice 的 `extraReducers` 中监听 `resetApp`

| Slice | 重置行为 | 文件路径 |
|-------|---------|---------|
| **appSlice** | 重置为 initialState，保留 `loadingText` 和 `managerHasInitialized` | `app/appSlice.ts:143-147` |
| **budgetfilesSlice** | 重置为 initialState | `budgetfiles/budgetfilesSlice.ts:466` |
| **accountsSlice** | 重置为 initialState | `accounts/accountsSlice.ts:84` |
| **transactionsSlice** | 重置为 initialState | `transactions/transactionsSlice.ts:69` |
| **modalsSlice** | 重置为 initialState（关闭所有模态框） | `modals/modalsSlice.ts:754` |
| **notificationsSlice** | 重置为 initialState（清除所有通知） | `notifications/notificationsSlice.ts:97` |
| **prefsSlice** | 重置为 initialState，**保留 global 和 server 偏好** | `prefs/prefsSlice.ts:186-190` |
| **usersSlice** | 重置为 initialState | `users/usersSlice.ts:79` |

**额外清理**：
- `queryClient.clear()`: 清除 React Query 缓存
- `send('close-budget')`: 后端关闭数据库、停止服务

---

## 四、关键代码索引表

| 功能模块 | 文件路径 | 关键行号 |
|---------|----------|----------|
| **PWA 更新检测** | `packages/desktop-client/src/browser-preload.js` | 330-348 |
| **PWA 更新应用** | `packages/desktop-client/src/browser-preload.js` | 495-502 |
| **Electron 更新禁用** | `packages/desktop-electron/preload.ts` | 79-83, 108-110 |
| **Electron 重启** | `packages/desktop-electron/index.ts` | 566-569 |
| **版本号比较工具** | `packages/desktop-client/src/util/versions.ts` | 全部 |
| **更新通知 UI** | `packages/desktop-client/src/components/FinancesApp.tsx` | 109-192 |
| **更新完成提示组件** | `packages/desktop-client/src/components/UpdateNotification.tsx` | 全部 |
| **同步事件监听** | `packages/desktop-client/src/sync-events.ts` | 全部 |
| **远端覆盖检测与通知** | `packages/desktop-client/src/sync-events.ts` | 249-283 |
| **关闭预算** | `packages/desktop-client/src/budgetfiles/budgetfilesSlice.ts` | 98-113 |
| **下载预算** | `packages/desktop-client/src/budgetfiles/budgetfilesSlice.ts` | 302-378 |
| **关闭并重新下载** | `packages/desktop-client/src/budgetfiles/budgetfilesSlice.ts` | 289-294 |
| **重置同步** | `packages/desktop-client/src/app/appSlice.ts` | 47-86 |
| **应用状态重置定义** | `packages/desktop-client/src/app/appSlice.ts` | 142-147 |
| **后端 resetSync 实现** | `packages/loot-core/src/server/sync/reset.ts` | 10-91 |
| **后端 download 实现** | `packages/loot-core/src/server/budgetfiles/app.ts` | 182-216 |
| **云存储 download** | `packages/loot-core/src/server/cloud-storage.ts` | 405-466 |
| **云存储 importBuffer** | `packages/loot-core/src/server/cloud-storage.ts` | 193-249 |
| **云存储 upload** | `packages/loot-core/src/server/cloud-storage.ts` | 251-339 |
| **初始全量同步** | `packages/loot-core/src/server/sync/index.ts` | 585-595 |

---

## 五、完整流程图

### 5.1 应用 PWA 热更新流程

```
┌─────────────────────────────────────────────────────────────┐
│                     浏览器启动阶段                            │
└─────────────────────────────────────────────────────────────┘
                              ↓
          browser-preload.js: registerSW()
                              ↓
          [ 后台 Service Worker 持续检查更新 ]
                              ↓
          检测到新版本 SW 进入 waiting 状态
                              ↓
          onNeedRefresh → markUpdateReadyForDownload()
                              ↓
          isUpdateReadyForDownloadPromise.resolve()
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                     UI 通知阶段                              │
└─────────────────────────────────────────────────────────────┘
                              ↓
FinancesApp: waitForUpdateReadyForDownload() resolved
                              ↓
dispatch(addNotification({ id: 'update-reload-notification' }))
                              ↓
          用户看到："A new version of Actual is available!"
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                     用户操作阶段                              │
└─────────────────────────────────────────────────────────────┘
                              ↓
          用户点击 "Update now" 按钮
                              ↓
          global.Actual.applyAppUpdate()
                              ↓
          updateSW() → 跳过 waiting SW → 页面刷新
                              ↓
                      新版本生效
```

### 5.2 预算文件被覆盖 Revert 流程

```
┌─────────────────────────────────────────────────────────────┐
│                     同步检测阶段                              │
└─────────────────────────────────────────────────────────────┘
                              ↓
          sync() → fullSync() → 比较 groupId
                              ↓
          不匹配 → SyncError('file-has-reset')
                              ↓
          sync-event 事件触发
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                     UI 通知阶段                              │
└─────────────────────────────────────────────────────────────┘
                              ↓
listenForSyncEvent 监听到错误
                              ↓
dispatch(addNotification({ id: 'needs-revert' }))
                              ↓
          用户看到："Syncing has been reset on this cloud file"
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                用户选择 Revert 分支                           │
└─────────────────────────────────────────────────────────────┘
                              ↓
          closeAndDownloadBudget({ cloudFileId })
                              ↓
          ┌─────────────────────────────────────┐
          │ Step 1: closeBudget()              │
          ├─────────────────────────────────────┤
          │ 1. dispatch(resetApp())            │
          │    → 所有 slice 重置状态           │
          │ 2. queryClient.clear()             │
          │ 3. send('close-budget')            │
          │    → 后端关闭数据库、停止服务       │
          └─────────────────────────────────────┘
                              ↓
          ┌─────────────────────────────────────┐
          │ Step 2: downloadBudget()           │
          ├─────────────────────────────────────┤
          │ 1. 从服务器下载 zip                │
          │ 2. 解密（如果需要）                 │
          │ 3. 解压写入本地文件系统             │
          │ 4. loadBudget() 加载新数据         │
          │ 5. initialFullSync() 初始同步      │
          └─────────────────────────────────────┘
                              ↓
                      数据恢复完成
```

### 5.3 预算文件被覆盖 Upload 流程

```
┌─────────────────────────────────────────────────────────────┐
│                用户选择 Upload 分支                           │
└─────────────────────────────────────────────────────────────┘
                              ↓
          resetSync() → send('sync-reset')
                              ↓
          ┌──────────────────────────────────────────────┐
          │ 后端 resetSync() 执行                        │
          ├──────────────────────────────────────────────┤
          │ 1. checkKey() 验证密钥                       │
          │ 2. resetSyncState() → 服务器创建新 groupId   │
          │ 3. 执行 SQL 清理：                           │
          │    DELETE messages_crdt, messages_clock      │
          │    DELETE tombstone = 1 的记录               │
          │    ANALYZE + VACUUM                          │
          │ 4. 保存 prefs: 清空 groupId, 同步时间戳      │
          │ 5. upload() → 本地文件上传到服务器           │
          │    → 服务器接受为最新版本                    │
          └──────────────────────────────────────────────┘
                              ↓
          sync() → 与服务器同步新状态
                              ↓
                      重置完成
```

---

## 六、设计洞察与注意事项

### 6.1 关键设计决策

1. **resetApp 作为全局重置信号**：通过 Redux action 机制，让各个 slice 自主处理重置逻辑，避免耦合

2. **prefsSlice 特殊处理**：重置时保留 `global` 和 `server` 偏好，确保用户登录状态、服务器配置等不丢失

3. **Revert 时的双重 closeBudget**：前端 closeBudget 后，后端 downloadBudget 内部再次调用 closeBudget，确保状态干净

4. **Upload 时的 CRDT 历史清除**：这是一个重要的优化点——重置同步时清除所有历史消息，显著减小数据库体积

### 6.2 潜在风险点

1. **Revert 时未同步数据丢失**：用户选择 Revert 前应确保没有重要的本地修改未同步

2. **Upload 时其他设备被迫 Revert**：重置同步会导致所有其他设备必须重新下载文件

3. **Electron 更新机制缺失**：当前 Electron 应用无内置自动更新，依赖外部渠道

4. **UpdateNotification 组件未使用**：`updateInfo` 状态定义但未被设置，可能是遗留代码或未来扩展
