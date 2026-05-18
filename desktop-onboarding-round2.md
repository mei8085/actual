# 桌面客户端首次启动引导流程深度分析（第二轮）

## 1. 跳过服务器步骤后的登录状态切换

### 1.1 核心入口：`onSkip()` 函数

**位置**: `packages/desktop-client/src/components/manager/ConfigServer.tsx:367-371`

```typescript
async function onSkip() {
  await setServerUrl(null);      // 1. 清除服务器配置
  await dispatch(loggedIn());    // 2. 触发"已登录"状态
  void navigate('/');            // 3. 跳转到首页
}
```

### 1.2 `setServerUrl(null)` 的深层影响

**位置**: `packages/loot-core/src/server/main.ts:95-116`

```typescript
handlers['set-server-url'] = async function ({ url, validate = true }) {
  if (url == null) {
    await asyncStorage.removeItem('user-token');  // 清除用户token
  }
  
  await asyncStorage.setItem('server-url', url);   // 持久化 null
  await asyncStorage.setItem('did-bootstrap', true); // 标记为已引导
  setServer(url);  // 设置为 null
  return {};
};
```

**关键效果**:
- `server-url` 被设置为 `null`
- `did-bootstrap` 被设置为 `true`（重要：跳过了正常的bootstrap流程）
- 用户token被清除

### 1.3 `loggedIn()` - "伪登录"状态

**位置**: `packages/desktop-client/src/users/usersSlice.ts:22-43`

```typescript
export const loggedIn = createAppAsyncThunk(
  `${sliceName}/loggedIn`,
  async (_, { dispatch }) => {
    await dispatch(getUserData());  // 获取用户数据
    void dispatch(loadAllFiles());  // 加载所有预算文件
  },
);
```

### 1.4 无服务器下的 `getUser()` 返回

**位置**: `packages/loot-core/src/server/auth/app.ts:146-205`

```typescript
async function getUser() {
  const serverConfig = getServer();
  if (!serverConfig) {
    // 没有服务器配置时
    if (!(await asyncStorage.getItem('did-bootstrap'))) {
      return null;  // 未引导时返回 null
    }
    return { offline: false };  // ✅ 已引导时返回"伪登录"状态
  }
  // ... 有服务器时的正常验证逻辑
}
```

### 1.5 状态切换完整流程

```
用户点击 "Don't use a server"
        │
        ▼
onSkip()
  ├─> setServerUrl(null)
  │    ├─> server-url = null
  │    ├─> did-bootstrap = true
  │    └─> user-token 被清除
  │
  └─> dispatch(loggedIn())
       ├─> getUserData()
       │    └─> send('subscribe-get-user')
       │         └─> getUser()
       │              └─> 检测到 server=null 且 did-bootstrap=true
       │                   └─> 返回 { offline: false }
       │
       └─> loadAllFiles()
            ├─> get-budgets (本地文件列表)
            └─> get-remote-files (返回 null, 因为无服务器)
        │
        ▼
跳转到 / (ManagementApp)
        │
        ▼
ManagementApp 检测到 userData 存在 (即使是 {offline:false})
        │
        ▼
显示欢迎页 / 预算文件列表
```

---

## 2. 本地模式与嵌入式同步模式的分流条件

### 2.1 三种模式的判定依据

| 模式 | `serverURL` 值 | `syncServerConfig` | 同步模式 |
|------|---------------|-------------------|----------|
| **纯本地模式** | `null` | 无 | `disabled` / `offline` |
| **嵌入式同步** | `http://localhost:5007` | `{port: 5007, autoStart: true}` | `enabled` |
| **外部服务器** | `https://...` | 无 | `enabled` |

### 2.2 同步模式设置逻辑

**位置**: `packages/loot-core/src/server/budgetfiles/app.ts:618-635`

```typescript
if (process.env.NODE_ENV !== 'test') {
  if (id === DEMO_BUDGET_ID) {
    setSyncingMode('disabled');  // 演示预算禁用同步
  } else {
    if (getServer()) {
      setSyncingMode('enabled');   // ✅ 有服务器: 启用同步
    } else {
      setSyncingMode('disabled');  // ❌ 无服务器: 禁用同步
    }
    await asyncStorage.setItem('lastBudget', id);
    await cloudStorage.possiblyUpload();  // 仅在有 cloudFileId+groupId 时执行
  }
}
```

### 2.3 `possiblyUpload()` 的守护条件

**位置**: `packages/loot-core/src/server/cloud-storage.ts:341-363`

```typescript
export async function possiblyUpload() {
  const { cloudFileId, groupId, lastUploaded } = prefs.getPrefs();

  // 条件1: 距上次上传超过7天
  const threshold = lastUploaded && monthUtils.addDays(lastUploaded, UPLOAD_FREQUENCY_IN_DAYS);
  if (lastUploaded && currentDay < threshold) {
    return;  // 未到上传时间
  }

  // 条件2: 必须同时有 cloudFileId 和 groupId
  if (!cloudFileId || !groupId) {
    return;  // ⭐ 纯本地预算永远不会走到上传逻辑
  }

  upload().catch(() => {});  // 异步上传, 不阻塞
}
```

### 2.4 纯本地模式的额外防护

**位置**: `packages/loot-core/src/server/main.ts:309-317`

```typescript
// init() 函数中, 当 config 无 serverURL 时:
setServer(null);

app.events.on('load-budget', () => {
  setSyncingMode('offline');  // 加载预算时强制设为 offline 模式
});
```

**四种同步模式说明**:
| 模式 | 含义 |
|------|------|
| `enabled` | 正常同步, 消息会发送到服务器 |
| `offline` | 离线模式, 消息暂存本地, 不发送 |
| `disabled` | 完全禁用, 不记录同步消息 |
| `import` | 导入模式, 特殊处理 |

---

## 3. 预算目录 ID 生成规则

### 3.1 ID 生成算法

**位置**: `packages/loot-core/src/server/util/budget-name.ts:41-61`

```typescript
export async function idFromBudgetName(name: string): Promise<string> {
  // 步骤1: 名称规范化
  // 替换所有空格和非字母数字字符为连字符
  let id = name.replace(/( |[^A-Za-z0-9])/g, '-') 
           + '-' 
           + uuidv4().slice(0, 7);  // 步骤2: 添加7位UUID后缀

  // 步骤3: 确保唯一性 (虽然概率极低)
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

### 3.2 示例

| 预算名称 | 生成的 ID |
|---------|-----------|
| `My Finances` | `My-Finances-a1b2c3d` |
| `家庭预算 2024` | `2024-x9y8z7w` |
| `Bob's Budget` | `Bob-s-Budget-k3m5n7p` |

### 3.3 ID 的安全校验

**位置**: `packages/loot-core/src/platform/server/fs/shared.ts:26-30`

```typescript
if (id.match(/[^A-Za-z0-9\-_]/)) {
  throw new Error(
    `Invalid budget id "${id}". Check the id of your budget...`,
  );
}
```

> 💡 **设计意图**: ID 只允许字母、数字、连字符和下划线，防止路径遍历攻击。

### 3.4 目录即身份

```
budget-id = 规范化名称 + UUID后缀
budget-dir = DOCUMENT_DIR/Actual/<budget-id>/
```

**关键特性**:
- 目录名 = 预算 ID
- 移动/重命名目录不会破坏功能（因为加载时会强制覆盖 metadata.json 中的 id）
- `metadata.json` 中的 id 字段仅作参考，真正的 ID 是目录名

---

## 4. Metadata 关键字段在各阶段的变化

### 4.1 Metadata 类型定义

**位置**: `packages/loot-core/src/types/prefs.ts:67-78`

```typescript
export type MetadataPrefs = Partial<{
  budgetName: string;           // 预算显示名称
  id: string;                   // 预算ID (与目录名一致)
  lastUploaded: string;         // 上次上传日期 (YYYY-MM-DD)
  cloudFileId: string;          // 云端文件ID (UUID)
  groupId: string;              // CRDT同步组ID
  encryptKeyId: string;         // 加密密钥ID
  lastSyncedTimestamp: string;  // 上次同步时间戳
  resetClock: boolean;          // 是否需要重置CRDT时钟
  lastScheduleRun: string;      // 上次执行计划任务时间
  userId: string;               // 用户ID (未使用)
}>;
```

### 4.2 生命周期状态变化

#### 阶段1: 刚创建（纯本地）

```typescript
// budgetfiles/app.ts:438-441
await fs.writeFile(
  fs.join(budgetDir, 'metadata.json'),
  JSON.stringify(prefs.getDefaultPrefs(id, budgetName))
);

// prefs.ts:89-91
export function getDefaultPrefs(id: string, budgetName: string) {
  return { id, budgetName };  // ⭐ 只有两个字段!
}
```

**此时 metadata.json**:
```json
{
  "id": "My-Finances-a1b2c3d",
  "budgetName": "My Finances"
}
```

#### 阶段2: 首次上传到同步服务器

**位置**: `packages/loot-core/src/server/cloud-storage.ts:251-338`

```typescript
export async function upload() {
  const { id, groupId, budgetName, cloudFileId: originalCloudFileId, encryptKeyId } = prefs.getPrefs();
  
  let cloudFileId = originalCloudFileId;
  if (!cloudFileId) {
    cloudFileId = uuidv4();  // 生成新的 cloudFileId
  }

  // 上传到服务器...
  
  if (res.status === 'ok') {
    await prefs.savePrefs({
      lastUploaded: monthUtils.currentDay(),  // 今天日期
      cloudFileId,                            // 新增
      groupId: res.groupId,                   // 服务器返回的groupId
    });
  }
}
```

**上传后 metadata.json**:
```json
{
  "id": "My-Finances-a1b2c3d",
  "budgetName": "My Finances",
  "lastUploaded": "2024-01-15",
  "cloudFileId": "550e8400-e29b-41d4-a716-446655440000",
  "groupId": "group-xyz-789"
}
```

#### 阶段3: 同步过程中

`lastSyncedTimestamp` 会在每次同步成功后更新（由同步逻辑写入）。

#### 阶段4: 下载到新设备

**位置**: `packages/loot-core/src/server/cloud-storage.ts:218-226`

```typescript
// 下载时更新 metadata
meta = {
  ...meta,
  cloudFileId: fileData.fileId,
  groupId: fileData.groupId,
  lastUploaded: monthUtils.currentDay(),
  encryptKeyId: fileData.encryptMeta ? fileData.encryptMeta.keyId : null,
};
```

### 4.3 关键字段作用详解

| 字段 | 存在条件 | 作用 |
|------|----------|------|
| `id` | 始终 | 本地唯一标识，与目录名一致 |
| `budgetName` | 始终 | 用户可见的显示名称 |
| `cloudFileId` | 已上传 | 服务器上的文件标识，UUID格式 |
| `groupId` | 已上传 | CRDT同步组标识，关联本地与云端 |
| `lastUploaded` | 已上传 | 上次完整上传日期，用于控制上传频率（7天） |
| `encryptKeyId` | 已加密 | 加密密钥标识，用于端到端加密 |
| `lastSyncedTimestamp` | 已同步 | 上次增量同步时间戳 |
| `resetClock` | 特殊情况 | 标记是否需要在新设备上重置CRDT时钟 |

### 4.4 加载时的强制修正

**位置**: `packages/loot-core/src/server/prefs.ts:41-44`

```typescript
export async function loadPrefs(id?: string): Promise<MetadataPrefs> {
  // ... 读取文件
  prefs = JSON.parse(await fs.readFile(fullpath));
  
  // ⭐ 强制覆盖: 无论文件中存的是什么id, 都以目录名为准
  prefs.id = id;
  return prefs;
}
```

> 💡 **设计意图**: 这使得用户可以自由重命名/移动预算目录，系统会自动修正ID。

---

## 5. 完整状态流转图

### 5.1 引导流程分流

```
首次启动
  │
  ▼
server-url 为 null?
  ├─> 是 ──> did-bootstrap 为 true?
  │        ├─> 是 ──> 返回 {offline:false} (伪登录) ──> 本地模式
  │        └─> 否 ──> 显示配置服务器页面 (/config-server)
  │
  └─> 否 ──> 有服务器URL
           ├─> 验证服务器连接
           ├─> 需要bootstrap? ──> /bootstrap
           └─> 已bootstrap? ──> /login
```

### 5.2 预算创建后状态流转

```
创建预算 (纯本地)
  │
  ├─> metadata: {id, budgetName}
  ├─> 同步模式: disabled
  └─> 不会上传
       │
       ▼
[可选] 用户点击"启用同步"
       │
       ▼
调用 upload()
  │
  ├─> 生成 cloudFileId (UUID)
  ├─> 上传到服务器
  ├─> 服务器返回 groupId
  ├─> 更新 metadata: {cloudFileId, groupId, lastUploaded}
  └─> 同步模式: enabled
       │
       ▼
后续操作
  ├─> 每次加载预算: 检查是否需要上传 (7天阈值)
  ├─> 数据变更: 生成同步消息, 发送到服务器
  └─> 定期全量同步
```

---

## 6. 关键代码位置速查

| 功能 | 文件 | 行号 |
|------|------|------|
| 跳过服务器处理 | `ConfigServer.tsx` | L367-371 |
| 设置服务器URL | `loot-core/.../main.ts` | L95-116 |
| 获取用户状态 | `loot-core/.../auth/app.ts` | L146-205 |
| 伪登录实现 | `usersSlice.ts` | L22-43 |
| 预算ID生成 | `loot-core/.../budget-name.ts` | L41-61 |
| Metadata默认值 | `loot-core/.../prefs.ts` | L89-91 |
| 上传逻辑 | `loot-core/.../cloud-storage.ts` | L251-338 |
| 条件上传守护 | `loot-core/.../cloud-storage.ts` | L341-363 |
| 同步模式设置 | `loot-core/.../budgetfiles/app.ts` | L618-635 |
