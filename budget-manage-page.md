# 预算管理页面状态机深度分析

## 修正说明

本文档是对之前分析的修正和补充。之前的分析存在以下不准确之处：

1. **`unknown` 状态的触发条件理解不完整**：不仅是网络错误，未登录或服务器返回错误也会触发
2. **`detached` 状态的 UI 展示问题**：代码中该状态被错误地显示为 "Syncing"
3. **下载链路的分层不清晰**：没有区分前端 thunk、后端 handler、cloud-storage 三层的职责
4. **`loadBudget` 的去重逻辑遗漏**：后端有幂等性检查
5. **`broken` 文件的处理方式不准确**：不是完全隐藏，而是移到列表末尾

---

## 一、文件状态类型系统

### 1.1 核心状态枚举

定义在 `packages/loot-core/src/types/file.ts:5-11`：

```typescript
type FileState =
  | 'local'     // 仅本地存在，未同步
  | 'remote'    // 仅云端存在，可下载
  | 'synced'    // 本地已下载且与云端同步
  | 'detached'  // 本地存在但 groupId 不匹配（同步状态断裂）
  | 'broken'    // 用户不应访问此文件
  | 'unknown';  // 无法确定云端状态
```

### 1.2 文件类型层次

| 类型 | 说明 | 关键字段 |
|------|------|----------|
| `LocalFile` | 纯本地文件 | `state: 'local'` |
| `SyncableLocalFile` | 可同步的本地文件（状态异常） | `cloudFileId`, `groupId`, `state: 'broken' \| 'unknown'` |
| `SyncedLocalFile` | 已同步的本地文件 | `cloudFileId`, `groupId`, `state: 'synced' \| 'detached'` |
| `RemoteFile` | 云端文件 | `cloudFileId`, `groupId`, `state: 'remote'` |

---

## 二、本地与远端文件的并列展示

### 2.1 数据获取三层架构

预算文件列表的加载涉及三层协作：

```
┌─────────────────────────────────────────────────────────────┐
│  前端 UI 层 (BudgetFileSelection.tsx)                        │
│  - 调用 loadAllFiles() thunk                                 │
│  - 从 Redux store 读取 allFiles 渲染列表                     │
└───────────────────────────┬─────────────────────────────────┘
                            │
┌───────────────────────────▼─────────────────────────────────┐
│  前端状态层 (budgetfilesSlice.ts)                            │
│  - loadAllFiles: 并行调用 get-budgets 和 get-remote-files   │
│  - reconcileFiles: 状态协调核心算法                          │
└───────────────────────────┬─────────────────────────────────┘
                            │
┌───────────────────────────▼─────────────────────────────────┐
│  后端服务层 (budgetfiles/app.ts + cloud-storage.ts)          │
│  - get-budgets: 遍历本地文件系统读取 metadata.json           │
│  - get-remote-files: 调用同步服务器 API 获取云端文件列表     │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 前端 loadAllFiles 实现

`budgetfilesSlice.ts:37-47`：

```typescript
export const loadAllFiles = createAppAsyncThunk(
  `${sliceName}/loadAllFiles`,
  async (_, { dispatch, getState }) => {
    // 并行获取本地和云端文件
    const budgets = await send('get-budgets');
    const files = await send('get-remote-files');
    
    // 触发状态协调
    dispatch(setAllFiles({ budgets, remoteFiles: files }));
    
    return getState().budgetfiles.allFiles;
  }
);
```

**关键点**：
- 两个请求是并行发送的，不是串行
- `setAllFiles` reducer 内部调用 `reconcileFiles` 进行状态协调
- 无论哪一个请求失败，都会进入协调逻辑

### 2.3 后端 get-budgets 实现

`budgetfiles/app.ts:105-140`：

```typescript
async function getBudgets() {
  const paths = await fs.listDir(fs.getDocumentDir());
  const budgets = await Promise.all(
    paths.map(async name => {
      const prefsPath = fs.join(fs.getDocumentDir(), name, 'metadata.json');
      if (await fs.exists(prefsPath)) {
        const prefs = JSON.parse(await fs.readFile(prefsPath));
        return {
          id: name,  // 目录名作为规范 ID
          cloudFileId: prefs.cloudFileId,
          groupId: prefs.groupId,
          name: prefs.budgetName || '(no name)',
        };
      }
      return null;
    })
  );
  return budgets.filter(Boolean);
}
```

**设计决策**：使用目录名作为预算的规范 ID，避免用户移动/重命名目录导致 ID 失效。

### 2.4 后端 get-remote-files 实现

`cloud-storage.ts:374-403`：

```typescript
export async function listRemoteFiles(): Promise<RemoteFile[]> {
  const userToken = await asyncStorage.getItem('user-token');
  if (!userToken) {
    return null;  // 未登录，返回 null
  }
  
  try {
    const res = await fetchJSON(SYNC_SERVER + '/list-user-files', {
      headers: { 'X-ACTUAL-TOKEN': userToken }
    });
    
    if (res.status === 'error') {
      return null;  // 服务器返回错误
    }
    
    return res.data.map(file => ({
      ...file,
      hasKey: encryption.hasKey(file.encryptKeyId)
    }));
  } catch (e) {
    return null;  // 网络请求失败
  }
}
```

**返回 null 的三种情况**：
1. 没有 user token（用户未登录）
2. 网络请求失败（服务器不可达）
3. 服务器返回错误状态

---

## 三、unknown / detached / broken 的判定边界

### 3.1 reconcileFiles 算法详解

`budgetfilesSlice.ts:516-616` 是状态机的核心。算法的输入是：
- `localFiles: Budget[]` - 本地预算列表
- `remoteFiles: RemoteFile[] | null` - 云端文件列表（可能为 null）

输出是合并后的 `File[]` 数组。

### 3.2 状态判定真值表

对于**有 cloudFileId 和 groupId** 的本地文件，状态判定如下：

| remoteFiles | 找到匹配远端 | groupId 匹配 | 结果状态 |
|-------------|-------------|-------------|----------|
| `null`      | -           | -           | `unknown` |
| 非 `null`   | 否          | -           | `broken` |
| 非 `null`   | 是          | 否          | `detached` |
| 非 `null`   | 是          | 是          | `synced` |

对于**没有 cloudFileId** 的本地文件，直接判定为 `local`。

对于**未被本地文件匹配**的远端文件，标记为 `remote`。

### 3.3 各状态详细说明

#### unknown 状态
- **触发条件**：`remoteFiles == null`（未登录、网络错误、服务器错误）
- **语义**：无法确定云端文件的真实状态
- **用户可见**：显示 "Network unavailable" 和 `SvgCloudUnknown` 图标
- **代码位置**：`budgetfilesSlice.ts:529-538`

#### detached 状态
- **触发条件**：云端文件存在但 `remote.groupId !== localFile.groupId`
- **语义**：本地文件和云端文件虽然关联同一个 cloudFileId，但属于不同的同步组
- **可能原因**：同步状态被重置、文件在其他设备上被重新上传
- **用户可见**：⚠️ **UI 缺陷** - 代码中走到 `default` 分支，错误地显示为 "Syncing" 和 `SvgCloudCheck` 图标
- **代码位置**：`budgetfilesSlice.ts:559-571`

#### broken 状态
- **触发条件**：本地有 `cloudFileId` 但在云端列表中找不到
- **语义**：用户可能已失去访问权限，或文件在服务器上被删除
- **用户可见**：移到列表末尾，显示为 "Local"（状态被强制设为本地）
- **代码位置**：`budgetfilesSlice.ts:573-582`

### 3.4 排序与过滤逻辑

`budgetfilesSlice.ts:589-615`：

```typescript
const sorted = sortFiles(
  files
    .concat(/* 未匹配的远端文件 */)
    .filter(f => !f.deleted)  // 过滤已删除文件
);

// broken 文件移到末尾
return sorted
  .filter(f => f.state !== 'broken')
  .concat(sorted.filter(f => f.state === 'broken'));
```

**重要修正**：`broken` 文件不是被隐藏，而是被移到列表末尾仍然显示。

---

## 四、切换活跃预算的完整链路

### 4.1 前端触发点

`BudgetFileSelection.tsx:598-612` 中的 `onSelect` 函数根据当前状态有三种分支：

```typescript
const onSelect = async (file: File) => {
  const isRemoteFile = file.state === 'remote';
  
  if (!id) {
    // 情况1：当前没有打开的预算
    if (isRemoteFile) {
      await dispatch(downloadBudget({ cloudFileId: file.cloudFileId }));
    } else {
      await dispatch(loadBudget({ id: file.id }));
    }
  } else if (!isRemoteFile && file.id !== id) {
    // 情况2：当前有预算，切换到另一个本地预算
    await dispatch(closeAndLoadBudget({ fileId: file.id }));
  } else if (isRemoteFile) {
    // 情况3：当前有预算，下载并切换到云端预算
    await dispatch(closeAndDownloadBudget({ cloudFileId: file.cloudFileId }));
  }
};
```

### 4.2 链路对比：三种切换场景

#### 场景1：无打开预算 → 加载本地预算

```
前端 loadBudget thunk (budgetfilesSlice.ts:55-96)
  ↓ send('load-budget', { id })
后端 loadBudget handler (budgetfiles/app.ts:226-242)
  ├─ 检查 currentPrefs.id === id → 不相等
  ├─ 调用 closeBudget()（但 prefs 为空，实际什么也不做）
  └─ 调用 _loadBudget(id)
      ├─ 打开数据库
      ├─ 版本迁移
      ├─ 加载电子表格
      ├─ 启动服务
      └─ 启用同步
  ↓ 返回
前端继续
  ├─ 关闭模态框
  └─ 调用 loadPrefs()
```

#### 场景2：有打开预算 → 切换到另一个本地预算

```
前端 closeAndLoadBudget thunk (budgetfilesSlice.ts:277-283)
  ├─ 调用 closeBudget()
  │   ├─ resetApp() → 重置 Redux 状态
  │   ├─ queryClient.clear() → 清除 React Query 缓存
  │   └─ send('close-budget')
  │       ├─ 等待电子表格完成
  │       ├─ 关闭数据库
  │       ├─ 清除 lastBudget
  │       └─ 卸载 prefs
  └─ 调用 loadBudget({ id: fileId })
      └─ 同场景1的 loadBudget 流程
```

#### 场景3：有打开预算 → 下载并切换到云端预算

```
前端 closeAndDownloadBudget thunk (budgetfilesSlice.ts:289-295)
  ├─ 调用 closeBudget() → 同场景2的关闭流程
  └─ 调用 downloadBudget({ cloudFileId, replace: true })

前端 downloadBudget thunk (budgetfilesSlice.ts:302-379)
  ├─ 显示 "Downloading..."
  ├─ send('download-budget', { cloudFileId })
  │   └─ 后端 downloadBudget handler (budgetfiles/app.ts:182-216)
  │       ├─ cloudStorage.download(cloudFileId)
  │       │   ├─ 并行下载文件内容和文件信息
  │       │   ├─ 解密（如果需要）
  │       │   └─ importBuffer → 写入本地文件系统
  │       ├─ 调用 closeBudget() → 再次关闭（冗余但安全）
  │       ├─ 调用 loadBudget({ id }) → 加载新下载的预算
  │       └─ 调用 syncBudget() → 执行初始全量同步
  ├─ 成功后并行执行：
  │   ├─ loadGlobalPrefs()
  │   ├─ loadAllFiles() → 重新加载文件列表
  │   └─ loadBudget({ id }) → ⚠️ 再次调用 loadBudget
  └─ 隐藏 loading
```

### 4.3 关键细节：loadBudget 的幂等性

后端 `loadBudget` handler 有去重检查 `budgetfiles/app.ts:226-242`：

```typescript
async function loadBudget({ id }) {
  const currentPrefs = prefs.getPrefs();
  
  if (currentPrefs) {
    if (currentPrefs.id === id) {
      // 如果已加载相同预算，直接返回
      return {};
    } else {
      await closeBudget();
    }
  }
  
  return _loadBudget(id);
}
```

这解释了为什么场景3中前端可以安全地再次调用 `loadBudget` - 后端会检查 ID 是否相同，如果相同则直接返回，不会重复加载。

### 4.4 下载流程详解：cloudStorage.download

`cloud-storage.ts:405-466`：

```typescript
export async function download(cloudFileId) {
  // 并行执行两个请求
  const [fileInfo, fileBuffer] = await Promise.all([
    fetchJSON('/get-user-file-info'),  // 获取元数据（加密信息等）
    fetch('/download-user-file')       // 获取文件内容（zip）
  ]);
  
  // 解密（如果需要）
  let buffer = fileBuffer;
  if (fileInfo.encryptMeta) {
    buffer = await encryption.decrypt(buffer, fileInfo.encryptMeta);
  }
  
  // 解压并写入本地文件系统
  return importBuffer(fileInfo, buffer);
}
```

`importBuffer` 函数 `cloud-storage.ts:193-249`：
- 解压 zip 包，提取 `db.sqlite` 和 `metadata.json`
- 更新 metadata：设置 `cloudFileId`、`groupId`、`encryptKeyId`
- 如果目录已存在，只删除 db 和 meta 文件（保留备份）
- 写入新文件

---

## 五、状态机图示

```
                    ┌─────────────────┐
                    │  loadAllFiles   │
                    │  (并行获取)     │
                    └─────────────────┘
                             │
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
       ┌──────────┐   ┌──────────┐   ┌──────────────┐
       │get-budgets│   │get-remote│   │ remoteFiles  │
       │ (本地)    │   │-files    │   │  == null?    │
       └──────────┘   └──────────┘   └──────┬───────┘
              │              │              │ 是
              └──────────────┼──────────────┘
                             ▼ 否
                    ┌─────────────────┐
                    │ reconcileFiles  │
                    └─────────────────┘
         ┌───────────┬───────┼───────┬───────────┐
         ▼           ▼       ▼       ▼           ▼
    ┌─────────┐ ┌────────┐ ┌──────┐ ┌──────────┐ ┌────────┐
    │ local   │ │ synced │ │unknown│ │ detached │ │ remote │
    └─────────┘ └────────┘ └──────┘ └──────────┘ └────────┘
                                                          │
                                                          ▼
                                                用户点击 remote 文件
                                                          │
                                                          ▼
                                          ┌──────────────────────────┐
                                          │ closeAndDownloadBudget   │
                                          └───────────┬──────────────┘
                                                      │
                        ┌─────────────────────────────┼─────────────────────────────┐
                        ▼                             ▼                             ▼
              ┌──────────────────┐        ┌──────────────────────┐        ┌──────────────────┐
              │ 前端 closeBudget │        │ 后端 downloadBudget  │        │ 前端 loadAllFiles │
              │  - resetApp      │        │  - 下载解密导入      │        │  - 刷新列表状态   │
              │  - 清除缓存      │        │  - closeBudget       │        └──────────────────┘
              └──────────────────┘        │  - loadBudget        │
                                          │  - syncBudget        │
                                          └───────────┬──────────┘
                                                      │
                                                      ▼
                                          ┌──────────────────────┐
                                          │ 前端 loadBudget      │
                                          │ (后端去重，实际空转)  │
                                          └──────────────────────┘
```

---

## 六、关键设计要点与潜在问题

### 6.1 优秀设计

1. **目录名作为规范 ID**：避免用户手动移动目录导致 ID 失效
2. **loadBudget 幂等性**：后端检查 ID 是否相同，允许多次安全调用
3. **下载时保留备份**：目录已存在时只删除 db 和 meta，不删除整个目录
4. **容错关闭**：`closeBudget` 中每个步骤都有 try-catch，确保部分失败时仍能继续
5. **并行下载优化**：文件内容和元数据并行请求，减少总耗时

### 6.2 潜在问题

1. **detached 状态 UI 误导**：`BudgetFileState` 组件中 detached 状态走到 default 分支，显示为 "Syncing"，用户无法感知同步断裂
   - 代码位置：`BudgetFileSelection.tsx:187-214`

2. **场景3中 closeBudget 被调用两次**：
   - 前端 thunk 中调用一次
   - 后端 downloadBudget handler 中又调用一次
   - 虽然无害（第二次 prefs 已空），但属于冗余调用

3. **场景3中 loadBudget 被调用两次**：
   - 后端 downloadBudget handler 中调用一次
   - 前端 thunk 成功后又调用一次
   - 依赖后端的幂等性检查避免重复加载

4. **broken 文件仍然显示**：虽然移到末尾，但用户仍能点击，可能导致错误

### 6.3 性能考量

- `reconcileFiles` 是 O(n*m) 复杂度（每个本地文件遍历远端列表），但预算文件数量通常很小，不是问题
- 每次操作后都调用 `loadAllFiles`，确保状态最新

---

## 七、相关文件索引

| 文件路径 | 主要职责 | 关键行号 |
|----------|----------|----------|
| `packages/loot-core/src/types/file.ts` | 文件状态类型定义 | 5-47 |
| `packages/desktop-client/src/budgetfiles/budgetfilesSlice.ts` | 前端状态管理、reconcileFiles 算法 | 37-47, 55-96, 277-379, 516-616 |
| `packages/desktop-client/src/components/manager/BudgetFileSelection.tsx` | 预算选择 UI、onSelect 逻辑 | 164-265, 598-612 |
| `packages/loot-core/src/server/budgetfiles/app.ts` | 后端预算管理 handlers | 105-140, 182-280, 508-640 |
| `packages/loot-core/src/server/cloud-storage.ts` | 云端文件操作 | 374-466, 193-249 |
| `packages/desktop-client/src/app/appSlice.ts` | 应用级状态重置 | 37, 143-147 |
