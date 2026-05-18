# 桌面客户端首次启动引导流程分析

## 1. 整体架构概览

Actual 桌面客户端采用 **多进程架构**：
- **主进程 (Main Process)**: Electron 主进程，负责窗口管理、进程调度
- **后台服务进程 (Background Server)**: 运行 loot-core 业务逻辑，处理预算数据操作
- **嵌入式同步服务进程 (Embedded Sync Server)**: 可选进程，提供多设备同步能力
- **渲染进程 (Renderer Process)**: React UI 界面

```
Electron Main (index.ts)
├─> BrowserWindow (UI)
├─> UtilityProcess (loot-core bundle)  -- 预算数据操作
└─> UtilityProcess (sync-server)       -- 同步服务（可选）
```

## 2. 首次启动引导流程

### 2.1 启动阶段

**入口文件**: `packages/desktop-electron/index.ts`

1. **环境初始化** (L57-L65)
   ```typescript
   // 设置数据目录和文档目录
   process.env.ACTUAL_DOCUMENT_DIR = app.getPath('documents');
   process.env.ACTUAL_DATA_DIR = app.getPath('userData');
   ```

2. **检查自动启动同步服务** (L444-L449)
   ```typescript
   const globalPrefs = await loadGlobalPrefs();
   if (globalPrefs.syncServerConfig?.autoStart) {
     await startSyncServer(); // 启动嵌入式同步服务
   }
   ```

3. **创建窗口和后台进程** (L496-L507)
   ```typescript
   await createWindow();          // 创建 UI 窗口
   await createBackgroundProcess(); // 启动 loot-core 后台服务
   ```

### 2.2 UI 引导路由

**路由入口**: `packages/desktop-client/src/components/manager/ManagementApp.tsx`

引导流程由 `useBootstrapped` Hook 控制 (`common.tsx:25-102`):

```
启动
  │
  ▼
检查是否已配置服务器 URL
  ├─> 未配置 ──> /config-server (服务器配置页)
  │
  └─> 已配置 ──> 检查是否已初始化
                    ├─> 未初始化 ──> /bootstrap (设置密码)
                    │
                    └─> 已初始化 ──> /login (登录)
                              │
                              ▼
                        登录成功 ──> 预算文件列表 / 欢迎页
```

## 3. 本地预算 vs 同步服务器选择

### 3.1 配置界面

**文件**: `packages/desktop-client/src/components/manager/ConfigServer.tsx`

桌面端特有组件 `ElectronServerConfig` 提供三种选择：

| 选项 | 行为 | 数据流向 |
|------|------|----------|
| **启动嵌入式服务器** | 调用 `startSyncServer()` 启动本地同步进程，设置 `serverURL = http://localhost:port` | 本地进程间通信 |
| **使用外部服务器** | 用户输入远程服务器 URL，验证后设置为 `serverURL` | HTTPS 网络请求 |
| **不使用服务器** | 清除 `syncServerConfig`，停止同步服务，设置 `serverURL = null` | 纯本地操作 |

### 3.2 选择"不使用服务器"流程

```typescript
// ConfigServer.tsx:110-118
async function dontUseSyncServer() {
  setSyncServerConfig(null);                    // 清除同步服务配置
  if (electronSyncServerRunning) {
    await window.globalThis.Actual.stopSyncServer(); // 停止运行中的同步服务
  }
  onDoNotUseServer(); // 跳转到本地预算流程
}
```

选择后效果：
- `serverURL` 被设置为 `null`
- `setSyncingMode('disabled')` 禁用同步 (`budgetfiles/app.ts:625`)
- 所有预算文件仅存储在本地

## 4. 嵌入式同步进程启动机制

### 4.1 启动流程

**文件**: `packages/desktop-electron/index.ts:211-329`

```typescript
async function startSyncServer() {
  // 1. 加载全局配置
  const globalPrefs = await loadGlobalPrefs();
  
  // 2. 配置参数
  const syncServerConfig = {
    port: globalPrefs.syncServerConfig?.port || 5007,
    hostname: '127.0.0.1',
    ACTUAL_SERVER_DATA_DIR: path.resolve(process.env.ACTUAL_DATA_DIR!, 'actual-server'),
    // ... 其他路径配置
  };

  // 3. 查找 sync-server 模块
  const syncServerRoot = path.dirname(require.resolve('@actual-app/sync-server/package.json'));
  const serverPath = path.join(syncServerRoot, 'build/app.js');

  // 4. 创建子进程
  syncServerProcess = utilityProcess.fork(serverPath, [], forkOptions);
  
  // 5. 等待启动确认
  syncServerProcess.on('message', msg => {
    if (msg.type === 'server-started') {
      resolve(); // 服务已就绪
    }
  });
}
```

### 4.2 进程间通信

- **主进程 → 同步服务**: 通过 `postMessage` 发送指令
- **同步服务 → 主进程**: 通过 `process.parentPort.postMessage` 发送状态
- **UI → 主进程**: 通过 IPC 调用 `start-sync-server` / `stop-sync-server`

### 4.3 数据目录结构

同步服务使用独立的数据目录：
```
ACTUAL_DATA_DIR/
└─> actual-server/
    ├─> server-files/    -- 服务器内部数据
    └─> user-files/      -- 用户上传的文件
```

## 5. 工作目录与预算文件绑定机制

### 5.1 目录结构设计

**核心逻辑**: `packages/loot-core/src/platform/server/fs/shared.ts`

```typescript
// 预算目录 = 文档目录 + 预算ID
export const getBudgetDir = (id: string) => {
  return join(getDocumentDir(), id);
};
```

**实际目录结构**:
```
ACTUAL_DOCUMENT_DIR/Actual/
├─> budget-id-1/
│   ├─> db.sqlite        -- SQLite 数据库文件
│   ├─> metadata.json    -- 预算元数据
│   └─> backups/         -- 备份目录（如果启用）
├─> budget-id-2/
│   └─> ...
└─> _demo-budget/        -- 演示预算（特殊ID）
```

### 5.2 预算文件创建流程

**文件**: `packages/loot-core/src/server/budgetfiles/app.ts:398-464`

```typescript
async function createBudget({ budgetName, testMode }) {
  // 1. 生成预算ID (从名称哈希生成)
  const id = await idFromBudgetName(budgetName);
  
  // 2. 创建目录
  const budgetDir = fs.getBudgetDir(id);
  await fs.mkdir(budgetDir);
  
  // 3. 复制初始数据库模板
  await fs.copyFile(fs.bundledDatabasePath, fs.join(budgetDir, 'db.sqlite'));
  
  // 4. 创建元数据文件
  await fs.writeFile(
    fs.join(budgetDir, 'metadata.json'),
    JSON.stringify(prefs.getDefaultPrefs(id, budgetName))
  );
  
  // 5. 加载预算
  await _loadBudget(id);
}
```

### 5.3 元数据绑定

`metadata.json` 存储关键绑定信息：
```json
{
  "id": "budget-id-123",
  "budgetName": "My Budget",
  "cloudFileId": "file-abc-123",    // 云端文件ID（同步时存在）
  "groupId": "group-xyz-789",       // CRDT 同步组ID
  "encryptKeyId": "key-a1b2c3",     // 加密密钥ID（如果启用）
  "owner": "user@example.com",      // 所有者
  "lastSyncedTimestamp": 1234567890 // 上次同步时间
}
```

### 5.4 预算加载流程

**文件**: `budgetfiles/app.ts:508-640`

```typescript
async function _loadBudget(id) {
  // 1. 获取目录路径
  const dir = fs.getBudgetDir(id);
  
  // 2. 加载元数据和首选项
  await prefs.loadPrefs(id);
  
  // 3. 打开 SQLite 数据库
  await db.openDatabase(id);
  
  // 4. 启动备份服务
  startBackupService(id);
  
  // 5. 加载电子表格引擎
  await sheet.loadSpreadsheet(db, onSheetChange);
  
  // 6. 设置同步模式
  if (getServer()) {
    setSyncingMode('enabled');  // 有服务器则启用同步
  } else {
    setSyncingMode('disabled'); // 无服务器则纯本地
  }
}
```

## 6. 关键数据流

### 6.1 选择本地预算（无服务器）

```
用户点击"Don't use a server"
  │
  ▼
setServerURL(null)
  │
  ▼
创建预算时:
  - 目录: <DOCUMENTS>/Actual/<budget-id>/
  - 元数据: 无 cloudFileId / groupId
  - 同步模式: disabled
  │
  ▼
数据仅存储在本地 SQLite
```

### 6.2 选择嵌入式同步服务器

```
用户配置端口并点击"Start"
  │
  ▼
1. 保存 syncServerConfig 到 global-store.json
2. 调用 startSyncServer() 启动子进程
3. 等待 server-started 消息
4. setServerURL(http://localhost:5007)
  │
  ▼
创建预算时:
  - 本地存储同上
  - 自动上传到本地同步服务
  - 元数据包含 cloudFileId / groupId
  - 同步模式: enabled
```

## 7. 关键代码位置参考

| 功能 | 文件 | 关键行 |
|------|------|--------|
| Electron 主进程入口 | `desktop-electron/index.ts` | L439-507 |
| 同步服务启动 | `desktop-electron/index.ts` | L211-329 |
| UI 引导路由 | `desktop-client/.../ManagementApp.tsx` | L67-223 |
| 服务器配置 UI | `desktop-client/.../ConfigServer.tsx` | L28-283 |
| 预算文件管理 | `loot-core/.../budgetfiles/app.ts` | 全文 |
| 目录路径解析 | `loot-core/.../fs/shared.ts` | L7-L33 |
| 应用初始化 | `loot-core/.../main.ts` | L184-225 |

## 8. 设计特点

1. **目录即身份**: 预算目录名即为预算 ID，移动/重命名目录不影响功能
2. **元数据驱动**: 通过 `metadata.json` 实现本地文件与云端文件的绑定
3. **进程隔离**: 同步服务独立进程，崩溃不影响主应用
4. **渐进式功能**: 可随时从纯本地切换到同步模式（需重新上传预算）
5. **平台抽象**: `fs` 模块抽象层，桌面端和浏览器端使用不同实现
