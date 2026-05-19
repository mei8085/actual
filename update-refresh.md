# 桌面端更新与刷新机制分析

本文档分析 Actual Budget 桌面端在以下两种场景下的完整流程：
1. 检测到应用自身新版本
2. 当前预算文件被远端同步覆盖

## 一、应用版本更新机制

### 1.1 检测时机

应用版本更新检测分为两种实现：浏览器环境（含 PWA）和 Electron 桌面环境。

#### 浏览器/PWA 环境
**代码位置**：`packages/desktop-client/src/browser-preload.js:330-348`

检测基于 Service Worker 的更新机制：
```javascript
let isUpdateReadyForDownload = false;
let markUpdateReadyForDownload;
const isUpdateReadyForDownloadPromise = new Promise(resolve => {
  markUpdateReadyForDownload = () => {
    isUpdateReadyForDownload = true;
    resolve(true);
  };
});

const updateSW = IS_DEV
  ? () => window.location.reload()
  : registerSW({
      immediate: true,
      onNeedRefresh: markUpdateReadyForDownload,
    });
```

- 关键流程：
1. 通过 `virtual:pwa-register` 注册 Service Worker
2. 当检测到新的 Service Worker 版本时，触发 `onNeedRefresh` 回调
3. 设置 `isUpdateReadyForDownload = true 并 resolve Promise

**UI 层监听位置**：`packages/desktop-client/src/components/FinancesApp.tsx:109-140`

在 `FinancesApp` 组件初始化时启动监听：
```javascript
const init = useEffectEvent(() => {
  async function run() {
    await global.Actual.waitForUpdateReadyForDownload(); // 阻塞直到更新准备就绪
    dispatch(addNotification({ /* 更新通知 */ }));
  }
  void run();
});
```

#### Electron 桌面环境
**代码位置**：`packages/desktop-electron/preload.ts:79-83`

**注意：Electron 环境当前**中自动更新功能被禁用：
```javascript
// No auto-updates in the desktop app
isUpdateReadyForDownload: () => false,
waitForUpdateReadyForDownload: () =>
  new Promise<void>(() => {
    // This is used in browser environment; do nothing in electron
  }),
```

Electron 应用的更新通过外部机制（如系统级软件更新）完成，应用内仅做版本号显示和更新提示。

#### 被动版本检查
**代码位置**：`packages/desktop-client/src/util/versions.ts`

- `getLatestVersion()`: 从 GitHub API 获取最新 release 版本号
- `getIsOutdated()`: 比较当前版本与最新版本

**调用时机**：`packages/desktop-client/src/components/FinancesApp.tsx:144-146`

```javascript
useEffect(() => {
  void dispatch(getLatestAppVersion());
}, [dispatch]);
```

组件挂载时检查一次，如果用户开启了 `notifyWhenUpdateIsAvailable` 偏好设置且版本过旧，会显示更新通知。

### 1.2 提示 UI 状态

更新通知有两种类型：

#### 类型一：PWA 热更新提示（仅浏览器）
**代码位置**：`FinancesApp.tsx:118-136`

```javascript
{
  type: 'message',
  title: 'A new version of Actual is available!',
  message: 'Click the button below to reload and apply the update.',
  sticky: true,
  id: 'update-reload-notification',
  button: {
    title: 'Update now',
    action: async () => {
      await global.Actual.applyAppUpdate();
    },
  },
}
```

#### 类型二：新版本发布提示
**代码位置**：`FinancesApp.tsx:154-182`

显示条件：
- `notifyWhenUpdateIsAvailable` 偏好开启
- 检测到更新版本
- 该版本未被用户关闭过（通过 `flags.updateNotificationShownForVersion` 本地偏好记录）

```javascript
{
  type: 'message',
  title: 'A new version of Actual is available!',
  message: 'Version {{latestVersion}} of Actual was recently released.',
  sticky: true,
  id: 'update-notification',
  button: {
    title: 'Open changelog',
    action: () => window.open('https://actualbudget.org/docs/releases'),
  },
  onClose: () => {
    setLastUsedVersion(versionInfo.latestVersion);
  },
}
```

#### 类型三：更新完成提示
**代码位置**：`packages/desktop-client/src/components/UpdateNotification.tsx`

这是一个独立组件，在 `updateInfo` 和 `showUpdateNotification` 均为 true 时显示，提供 "Restart" 按钮。

### 1.3 用户接受后底层动作

#### 浏览器/PWA 环境
**代码位置**：`browser-preload.js:495-502`

```javascript
applyAppUpdate: async () => {
  updateSW();
  // Wait for the app to reload
  await new Promise(() => {
    // Do nothing
  });
}
```

`updateSW()` 来自 `virtual:pwa-register`，会：
1. 跳过 waiting 的 Service Worker
2. 触发页面刷新

#### Electron 环境
**代码位置**：`packages/desktop-electron/preload.ts:108-110`

Electron 中 `applyAppUpdate` 未实现（抛出错误）。但提供了 `relaunch` 方法：
```javascript
relaunch: () => {
  void ipcRenderer.invoke('relaunch');
}
```

**主进程处理**：`packages/desktop-electron/index.ts:566-569`
```javascript
ipcMain.handle('relaunch', () => {
  app.relaunch();
  app.exit();
});
```

## 二、预算文件被远端同步覆盖机制

### 2.1 检测时机

当同步过程中检测到本地文件的 groupId 或加密密钥与服务器不匹配时触发。

**代码位置**：`packages/desktop-client/src/sync-events.ts:249-283`

监听 `sync-event` 事件，当 `event.type === 'error' 且 `event.subtype` 为以下两种情况时触发：
- `file-has-reset`: 服务器上的同步已被重置
- `file-has-new-key`: 服务器上有新的加密密钥

```javascript
case 'file-has-reset':
case 'file-has-new-key':
  const { cloudFileId } = store.getState().prefs.local;
  // 显示 "Syncing has been reset" 通知
```

### 2.2 提示 UI

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

用户有两个选择：
1. **Revert（还原）**: 关闭当前本地文件，从服务器重新下载最新版本
2. **Upload（上传）**: 将本地文件作为最新版本上传到服务器（重置同步）

### 2.3 用户接受后底层动作

#### 选择 "Revert" 动作流程

**代码位置**：`packages/desktop-client/src/budgetfiles/budgetfilesSlice.ts:289-294`

```javascript
export const closeAndDownloadBudget = createAppAsyncThunk(
  `${sliceName}/closeAndDownloadBudget`,
  async ({ cloudFileId }: CloseAndDownloadBudgetPayload, { dispatch }) => {
    await dispatch(closeBudget());
    await dispatch(downloadBudget({ cloudFileId, replace: true }));
  },
);
```

##### 第一步：关闭当前预算
**代码位置**：`budgetfilesSlice.ts:98-113`

```javascript
export const closeBudget = createAppAsyncThunk(
  `${sliceName}/closeBudget`,
  async (_, { dispatch, getState, extra: { queryClient } }) => {
    const prefs = getState().prefs.local;
    if (prefs && prefs.id) {
      dispatch(resetApp());           // 重置应用状态
      queryClient.clear();            // 清除 React Query 缓存
      dispatch(setAppState({ loadingText: t('Closing...') }));
      await send('close-budget');    // 通知后端关闭预算
      dispatch(setAppState({ loadingText: null }));
      if (localStorage.getItem('SharedArrayBufferOverride')) {
        window.location.reload();    // 特殊情况下刷新页面
      }
    }
  },
);
```

##### 第二步：从服务器下载预算
**代码位置**：`budgetfilesSlice.ts:302-378`

```javascript
export const downloadBudget = createAppAsyncThunk(
  `${sliceName}/downloadBudget`,
  async ({ cloudFileId, replace = false }: DownloadBudgetPayload, { dispatch }) => {
    dispatch(setAppState({ loadingText: t('Downloading...') }));

    const { id, error } = await send('download-budget', { cloudFileId });

    if (!error && id) {
      await Promise.all([
        dispatch(loadGlobalPrefs()),
        dispatch(loadAllFiles()),
        dispatch(loadBudget({ id })),      // 加载新下载的预算
      ]);
      dispatch(setAppState({ loadingText: null }));
      return id;
    }
  },
);
```

#### 选择 "Upload" 动作流程

**代码位置**：`packages/desktop-client/src/app/appSlice.ts:47-86`

```javascript
export const resetSync = createAppAsyncThunk(
  `${sliceName}/resetSync`,
  async (_, { dispatch }) => {
    const { error } = await send('sync-reset');
    if (!error) {
      await dispatch(sync());
    }
  },
);
```

`sync-reset` 会：
1. 清除本地同步状态
2. 在服务器上创建新的同步 ID
3. 上传当前本地文件作为最新版本
4. 触发同步

## 三、关键代码索引

| 功能 | 文件路径 | 关键行号 |
|------|----------|----------|
| PWA 更新检测 | `packages/desktop-client/src/browser-preload.js` | 330-348 |
| PWA 更新应用 | `packages/desktop-client/src/browser-preload.js` | 495-502 |
| Electron 更新禁用 | `packages/desktop-electron/preload.ts` | 79-83 |
| Electron 重启 | `packages/desktop-electron/index.ts` | 566-569 |
| 版本号比较 | `packages/desktop-client/src/util/versions.ts` | 全部 |
| 更新通知 UI | `packages/desktop-client/src/components/FinancesApp.tsx` | 109-192 |
| 更新完成提示 | `packages/desktop-client/src/components/UpdateNotification.tsx` | 全部 |
| 同步事件监听 | `packages/desktop-client/src/sync-events.ts` | 全部 |
| 远端覆盖检测 | `packages/desktop-client/src/sync-events.ts` | 249-283 |
| 关闭预算 | `packages/desktop-client/src/budgetfiles/budgetfilesSlice.ts` | 98-113 |
| 下载预算 | `packages/desktop-client/src/budgetfiles/budgetfilesSlice.ts` | 302-378 |
| 关闭并重新下载 | `packages/desktop-client/src/budgetfiles/budgetfilesSlice.ts` | 289-294 |
| 重置同步 | `packages/desktop-client/src/app/appSlice.ts` | 47-86 |
| 应用状态重置 | `packages/desktop-client/src/app/appSlice.ts` | 131-149 |

## 四、流程总结

### 应用更新流程
```
启动 → FinancesApp 挂载
    ↓
waitForUpdateReadyForDownload() 等待 Promise
    ↓
Service Worker 检测到新版本 → resolve Promise
    ↓
显示 "Update now" 通知
    ↓
用户点击 "Update now"
    ↓
updateSW() → 激活新 SW → 页面刷新
```

### 预算文件被覆盖流程
```
同步过程中
    ↓
检测到 file-has-reset / file-has-new-key 错误
    ↓
显示 "Syncing has been reset" 通知
    ↓
用户选择:
  ├─ Revert → closeBudget() → downloadBudget() → loadBudget()
  └─ Upload → resetSync() → sync()
```
